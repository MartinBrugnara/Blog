Useful command in _daily_ debian machines administration.
=

Work in progress ...

# Networking
Additional information are available on [debian wiki](https://wiki.debian.org/NetworkConfiguration) and on [arch-wiki](https://wiki.archlinux.org/index.php/Network_configuration#Configure_the_IP_address).

## Permanent
#### Interface configuration
`/etc/network/interfaces`:
```
# eth0: w/hotplug - common for laptops - and DHCP.
auto eth0
allow-hotplug eth0
iface eth0 inet dhcp

# eth1: wo - common for servers - and static IP.
auto eth0
allow-hotplug eth0
iface eth0 inet static
    address 192.168.0.101
    netmask 255.255.255.0
    gateway 192.168.0.1

    # (optional) parameters

    # write these DNS to /etc/resolv.conf when this eth is connected
    # require resolvconf package.
    dns-nameservers 12.34.56.78 12.34.56.79
```

#### DNS configuration
`/etc/resolv.conf `:
(NOTE: leave empty if _resolvconf_ is in use)
```
nameserver 12.34.56.78
nameserver 12.34.56.79
```

#### Traffic routing (Internet_sharing)
Reference [arch-wiki](https://wiki.archlinux.org/index.php/Internet_sharing).
- Enable _ipv4.ip_forward_:
    Volatile solution: `> sysctl net.ipv4.ip_forward=1`
    Permanent solution) add `net.ipv4.ip_forward=1` to `/etc/sysctl.d/30-ipforward.conf`.
- Enable NAT:
```
iptables -t nat -A POSTROUTING -o wan_dev -j MASQUERADE
iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# For each natted network
iptables -A FORWARD -i net0 -o lan_dev -j ACCEPT
```

## Live
#### ip addr
The _iproute2_ package is required.
Reference [arch-wiki](https://wiki.archlinux.org/index.php/Network_configuration#Manual_assignment)
```
# Base commands
> ip link set interface (up|down)
> ip addr add IP_address/subnet_mask broadcast broadcast_address dev interface
> ip route add default via default_gateway

# Cleanup commands
#  Remove any assigned IP address
> ip addr flush dev interface
#  Remove any assigned gateway
> ip route flush dev interface

## Debug
- Show all alive host on a subnet: `nmap -sP ip/mask`.
    An example: `nmap -sP 192.168.2.1/24`.
    Ref [stackexchange](https://security.stackexchange.com/questions/36198/how-to-find-live-hosts-on-my-network)

# SSH (client)
Setup ssh personal configuration file to simplify your life.
`~/.ssh/config`
```
# security fix (CVE-0216-0777 and CVE-0216-0778)
# https://www.digitalocean.com/community/tutorials/how-to-fix-openssh-s-client-bug-cve-0216-0777-and-cve-0216-0778-by-disabling-useroaming
Host *
    UseRoaming no

# Simple host 
# NOTE: not required to be an existent domain
Host gw.ny 
    # IP or valid domain.
    Hostname 10.10.10.10
    Port 22
    User jhon
    IdentityFile /Users/jhon/.ssh/id_ecdsa

# Connect to an internal machine (m1) using gw as proxy.
# gw public exposed
# m1 lan only
Host m1.ny
    ProxyCommand ssh gw.ny nc 10.0.0.3 %p
```
