# AutoPyLab

This collection of scripts sets up a fully virtualized homelab and provides a couple of endpoints
for starting it up and tearing it down.

The setup script is designed to be modular so that if something breaks, you can independently
reconfigure the affected component(s) of the system without having to destroy and rebuild the
components that are still functioning.

The idea is to go from this:

```    
        ,---.   ,,,
      ,'     ';'   `. ,,
     ;'              `  `-,
    :      internet        :
     `--------------------'
             ||                                       ,--------------.
  ,-------------------------.                         |  user input  |
  |  consumer grade router  |                         '--------------'
  |      or similar         |                                   |
  `-------------------------'                                   |
             ||                                                 |
,------------||-------------------------------------------------|------.
|            ||                                                 |      |
|    ,===================.       ,========================.  {kernel}  |
|    |  host's real NIC  |=======|  host's network stack  |     |      |
|    `==================='       `========================'     |      |
|                                            ||                 |      |
|                                    ,======================.   |      |
|        host machine                |  host userspace      |<--'      | 
|                                    `======================'          |
|                                                                      |
`----------------------------------------------------------------------'
```

to this:

```    
        ,---.   ,,,
      ,'     ';'   `. ,,
     ;'              `  `-,
    :      internet        :
     `--------------------'
             ||                                                                   ,--------------.
  ,-------------------------.                                                     |  user input  |
  |  consumer grade router  |                                                     '--------------'
  |      or similar         |                                                               |
  `-------------------------'                                                               |
             ||                                                                             |
,------------||-----------------------------------------------------------------------------|------.
|            ||                                                                             |      |
|    ,===================.       ,====================.      ,========================.  {kernel}  |
|    |  host's real NIC  |=======|  external vbridge  |======|  host's network stack  |     |      |
|    `==================='       `===================='      `========================'     |      |
|                                          ||                                    ||         |      |
|                             ,---------------------------------,        ,======================.  |
|                             |  virtual router/gateway         |        |  host userspace      |  | 
|       host machine          |  * iptables - routing/firewall  |        `======================'  | 
|                             |  * snort - IDS/IPS              |                                  | 
|                             `---------------------------------'                                  |
|                                          ||                                                      |
|                                ,====================.                                            |
|                                |  internal vbridge  |                                            |
|                                `===================='                                            |
|                                   ||           ||                                                |
|           ,-------------------------.          ||                                                |
|           |  main guest             |   ,----------------------------.                           |
|           |  * wireguard server     |   | arbitrary secondary guests |.                          |
|           |    - remote access      |   | with arbitrary services    ||.                         |
|           |  * arbitrary services   |   `----------------------------'||                         |
|           `-------------------------'    `----------------------------'|                         |
|                                           `----------------------------'                         |
|                                                                                                  |
`--------------------------------------------------------------------------------------------------'
```

AutoPyLab expects that you're starting with a fresh installation of Debian 12 stable with a
hard-wired internet connection via an arbitrary consumer-grade gateway, and any security setup
that seems appropriate to you. Root privileges are needed to set the system up, but VMs are run
as the non-privileged user so that if an adversary were to escape the VM, they would still have
to escalate their priveleges on the host.

From the public internet, the only intended way to access the lab is by using a VPN client to
connect to the wireguard server running on the main guest. A pre-shared key is required to even be
able to tell that wireguard is listening for traffic. Once peers can see the server with the PSK,
wireguard will only allow peers whose keys are in wireguard's list of trusted peers to access the
VPN. __Note from wireguard's whitepaper re: tunneling all traffic in this way:__
> since this roaming property ensures that peers will have the very latest external source IP and
> UDP port, there is no requirement for NAT to keep sessions open for long. (For use cases in which
> it is imperative to keep open a NAT session or stateful ﬁrewall indeﬁnitely, the interface can be
> optionally conﬁgured to periodically send persistent authenticated keepalives.)

It's also possible to allow direct external access to services on guest machines by modifying the
routing and firewalling rules on the virtual gateway, but this is not advisable for security reasons

# Why?

The lab is architected for maximum utilization of the highly efficient and secure mechanisms built
into the host's kernel. It is composed entirely of KVM-based virtual machines, interacting via
virtio devices. As such, traffic and data flowing in, around, and out of the lab stays entirely in
the kernel.

# Security considerations

Of course, the entire stack from the physical gateway to the services running on the guest VM must
be considered. Independent of this software, external traffic arrives at the physical gateway, a
significant source of variability, then passes into the host machine's NIC and associated hardware.
After the traffic crosses the NIC-kernel boundary on the host's hardware, this software is
reponsible for the secure handling of that traffic, dependent on the security of the host itself.

Physical access to the host machine and its gateway is up first for consideration. The cybersecurity
of this setup hinges on the physical security of the environment the host machine is in. It's
advisable to use secure passwords (or more secure authentication methods) for both the regular user
and root on the host. It's also advisable to set up full disk encryption so that, in the event of a
physical security breach, simple possession of the storage disk(s) being used by the lab does not
equate to owning the lab.

Host security being considered, we can now focus on the lab itself. Of particular concern are layers
III, IV, and V of the system as described in the figure below. This is where the architecture
switches from careful, automatic, opinionated code reuse to prioritize configurability and
flexibility. By default, there will be no services running on the internal VM and accessible from
outside the host machine at layer V besides wireguard, and the default configuration for layers III
and IV are minimalistic, dropping all traffic initiated from outside of the lab except wireguard
packets. The lab is still able to initiate two-way exchanges of packets with external hosts, e.x.
you can access a terminal on the guest via serial console on the host and browse the web, etc.,
unhindered - responses to this guest-initiated traffic must be handled by the entire stack, and may
warrant further security considerations depending on how the guest is used.

What I'm trying to say is, examine the flow of traffic below and ensure that each node is adequately
safe for your use case.

```
I. External supporting hardware
Physical consumer-grade router
| ^
| |
v |
NIC hardware
| ^
| |
v |
II. host kernel
    NIC driver
    | ^
    | |
    v |
    host bridge module
    | ^
    | |
    v |
    III. virtual router kernel
        guest virtio driver
        | ^
        | |
        v |
    host bridge module
    | ^
    | |
    v |
    IV. guest VM kernel
        guest  #2 virtio driver
        | ^
        | |
        v |
        V. guest service
        further traffic may originate here depending on services
```

With the simple default configuration in place, we can drill down to layer V and focus on the sole
means of accessing the lab, the wireguard VPN server running on the guest.

From the public internet, wireguard narrows the attack surface down to a single port behind multiple
keys. An adversary would need the PSK (which is probably private, depending on your setup) to see
the wireguard server, and they would need one of the private keys owned by any client machines that
you configure the lab to trust (by adding the associated public key to wireguard's peers) to then
access the VPN. It bears mentioning that it's probably still possible for an advanced adversary to
determine that wireguard is listening for connections using a timing attack - it takes some
non-negligible amount of time for UDP packets to travel from the host's physical router to
wireguard's listening port and back, whereas truly closed ports will reject packets more
immediately. This is especially true if the host is stressed for resources.

Once in the VPN, an adversary would effectively own the internal network of guest VMs. To move
vertically, they would need to either exploit the virtual gateway as its client or exploit one of
the layers enclosing the VM with the encrypted contents of their network traffic across the system
(which I'm pretty sure is mathematically inconceivable).

Special attention needs to be given to the interface between the virtual gateway and the guest
network. The gateway exposes DHCP and DNS via dnsmasq, and switching and firewalling facilities via
iptables. An alternative FreeBSD-based gateway could be used if the BSD kernel's network stack is
preferable, in which case switching and firewalling are handled by pf.
