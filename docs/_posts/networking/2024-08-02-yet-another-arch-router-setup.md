---
layout: post
title:  "Yet Another Arch Linux Router"
date:   2024-08-02 20:00:00 +0800
categories: networking
---

## Background

Recently I got a used enterprise-level edge router, VMWare SD-WAN Edge 620, with two 10G-SFP+ ports and six 1G-RJ45 ports, powered by a 4-core Intel C3558 with 8G of DDR4-ECC memory.

With such a powerful hardware I decided to replace my current apartment router (BananaPi BPi-R4, with MT7988A (4-core A73 @1.8Ghz) + 4G DDR4 + 8G eMMC, running OpenWrt snapshot) with it. So I could get an unplugged BPI-R4 to tinker with (a new device to add Arch Linux ARM support to!).

As I didn't want VMs nor containers (router in VM/container results in circular network dependency that's hard to fix once broken, a.k.a. all-in-boom; VM/container on router results in security holes and tainted firewall), and wanted to have some cutting edge caching proxy running (pacoloco as an Arch repo caching proxy + wireguard + some tproxy services, to be precise). No ESXi, no ProxmoxVE: I don't want any hypervisors and only wanted to choose a generic Linux distro as base. And no pre-defined configuration: I want to set every possible component up by myself.

So Arch router again. I had been using Arch Linux on servers for five years, and I used to use it for router four years ago for almost a year. But I gave up then due to Network Manager breaking and the broken router results in inaccessibility to Internet from time to time, and some family members complaining, of course. This time I decided to use saner components for the whole router, and embrace a more gentle management style.

The following are the essential components I chose for another Arch router four years ago, what I chose for the new router now instead, and, for reference, what OpenWrt uses

||OpenWrt<=21.02|OpenWrt>=22.03|Retired Arch Router|New Arch Router|
|-|-|-|-|-|
|Network Manager|UCI + netifd|UCI + netifd|Network Manager|systemd-networkd|
|Firewall Frontend|firewall3|firewall4|-|-|
|Firewall Backend|iptables|nftables|nftables|nftables|
|DNS Server|dnsmasq|dnsmasq|dnsmasq|bind9 / named|
|DHCP v4 Server|dnsmasq|dnsmasq|dnsmasq|kea|
|DHCP v6 Server|odhcpd|odhcpd|-|part of systemd-networkd, SLAAC only|
|SSH Server|dropbear|dropbear|openssh|openssh|

The new setup has been running stably for a month, and I had finished some projects with BPI-R4, I think it's time to document how I did the setup

## Setup

### System installation

Do the base system installation following [Arch Installation Guide](https://wiki.archlinux.org/title/Installation_guide), don't do installation with guided installer, as each maintainer of such installer would have their preference for components. We only want the needed bare-minimum parts to make the system bootable, without even configuring network manager.

### Network

After first boot and logging in you should have no network. There shouldn't be any network manager running. If your network is already up and running then you're on your own to figure out how to remove the pre-configured network stuffs.

#### Figuring out port layout

Try to `ip link` `up` all ports shown in `ip link` that starts with `en` or `eth`, with no network cable connected, e.g.

```sh
sudo ip link set enp3s0f0u2u3 up
```

After this all ports shall still be shown as `state DOWN` in `ip link`

Then connect cable to ports one by one and check which port has `state UP` after being plugged in.

After all ports are figured out, `ip link` `down` all ports, e.g.

```sh
sudo ip link set enp3s0f0u2u3 down
```

The below layout is what I had on my Edge 620.
```
     10 GbE   |           GbE         
--------------+-----------------------
              |[ens2f2] [ens2f0] [eno5] 
 [eno7] [eno8]|[ens2f3] [ens2f1] [eno6]
```

#### Port usage and network layout

The basic idea is that no port bandwidth shall be wasted. A 1G port should be connect to another 1G port, 2.5G to 2.5G, 10G to 10G. 100M devices are banned from the network. Cross-bandwidth bridging is done by hardware.

- `ens2f3` (1G) would be used as the WAN port, connecting to landlord's modem + router with 300M down link speed and 30M up link speed.
- all other ports would be joined to a bridge, they can communicate with each other freely, but most traffic happen under switches and most devices connected directly to the router don't access other LAN devices through the software bridge
  - `eno7` (10G) would be connected to a 8x2.5G + 10G switch
    - all 2.5G devices would be connected to the 2.5G ports on switch, some doing bonding
    - a wireless router in AP mode with 3x1G + 2.5G would be connected to the switch to function as 2.5G-1G bridge
      - a 1G switch would be connected to the 1G port on AP
      - high-traffic 1G devices would be connected to either the AP or the 1G switch depending on their in-LAN inter-traffic
  - `eno8` (10G) would be connected to my home desktop
  - all other 1G ports would be connected to low-traffic 1G devices (consoles, set-top boxes, etc)

#### WAN

As I'm renting a room (in a three-room apartment) I had to use my landlord's ISP subscription. The ISP fiber modem + router has no public IPv4, and as not the actual subscriber I had no priviledge to ask the ISP for a public IPv4 address. Luckily still, the ISP delegates an IPv6 /60 suffix. And I could at least get a /64 suffix without breaking the network for my other room-mates. So on my WAN, I would have:
- a private IPv4 address (private to the apartment LAN)
- a public IPv6 address (in the /64 for apartment LAN)
- a public IPv6 /64 suffix (different from the /64 for apartment LAGN, in the /60 sent by ISP).

For these I need to configure the WAN to do DHCPv4 (to get a private IPv4 address to do IPv4 routing), IPv6 SLAAC (to get a public IPv6 address to do IPv6 routing) and DHCPv6-PD (to get a IPv6 /64 suffix to assign to my other devices in my own LAN), so let's create a `.network` file for the interface.

_`/etc/systemd/network/20-GbE-Down-Left-as-WAN.network`_

```conf
[Match]
Name=ens2f3

[Network]
DHCP=yes
IPv6AcceptRA=yes
IPv4Forwarding=yes
IPv6Forwarding=yes
IPv6PrivacyExtensions=no

[Link]
RequiredForOnline=no

[DHCPv6]
WithoutRA=solicit
```

In which: 
- `Network.DHCP=yes` ensures we do both DHCPv4 and DHCPv6 to gain both an IPv4 address and an IPv6 address for the interface
- `Network.IPv4Forwarding=yes` and `Network.IPv6Forwarding=yes` ensure we allow forwarding to this interface (unfortunately for IPv6 the option does not work as expected, addtional sysctl set up for IPv6 is needed, read further)
- `Network.IPv6AcceptRA=yes` ensures we always do SLAAC for IPv6
- `Network.IPv6PrivacyExtension=no` ensures we have a consistent DUID when doing SLAAC for IPv6
- `Link.RequiredForOnline=no` ensures startup of DNS and DHCP servers won't be delayed if WAN connection is down at boot
- `DHCPv6.WithoutRA=solicit` ensures we always start a DHCPv6 client to obtain IPv6 Prefix Delegation, even when there's no Router Advertisement (like when outer LAN only does DHCPv6 for PD but not SLAAC)

#### LAN Bridge

All LAN interfaces need to be joined into a single bridge, let's create a `.netdev` file for the bridge to init it, 

_`/etc/systemd/network/10-Bridge.netdev`_

```conf
[NetDev]
Name=bridge0
Kind=bridge
# MACAddress= 
```

Note we didn't explicitly set the MAC Address. As this is a virtual interface it won't have persistent MAC address. Some client won't be happy for this (like Windows clients would consider this a new network each time the router MAC address changes as it assotiates a "network" to a router MAC address). On activating a MAC Address would be generated for it, and after that we would want to make it persistent by uncommenting the `MACAddress` line and setting it up.

Then let's create a `.network` file for all LAN interfaces to configure them as slaves of the above bridge, 

_`/etc/systemd/network/20-10GbE-GbE-Others-as-Bridge-Slave.network`_

```conf
[Match]
Name=ens2f0 ens2f1 ens2f2 eno5 eno6 eno7 eno8

[Network]
Bridge=bridge0

[Link]
RequiredForOnline=no
```

Here `Link.RequiredForOnline=no` is needed as not all LAN ports would be connected at all time. Without it, `systemd-networkd-wait-online.service` would hang for minutes waiting for all these ports. If, however, you're sure that all these ports would be online 100% of the time, then you can surely remove the `[Link]` section

Then let's create a `.network` file to configure L-3 network on the bridge interface

_`/etc/systemd/network/30-Bridge.network`_

```conf
[Match]
Name=bridge0

[Network]
Address=192.168.67.1/24
DHCPPrefixDelegation=yes
IPMasquerade=ipv4
IPv6SendRA=yes
IPv6AcceptRA=no
IPv6Forwarding=yes

[DHCPPrefixDelegation]
UplinkInterface=ens2f3
SubnetId=0
Announce=yes

[IPv6SendRA]
OtherInformation=yes
```


In which: 
- `Network.DHCPPrefixDelegation=yes` ensures we assign the IPv6 prefix we got from another interface (in this case, WAN, `ens2f3`) to devices connected to this interface.
- `Network.IPMasquerade=ipv4` ensures we do NATv4 for devices connected to this interface. This also implicitly sets `Network.IPv4Forwarding=yes` so traffic forwarded to this interface is allowed.
- `Network.IPv6SendRA=yes` and `DHCPPrefixDelegation.Announce=yes` ensure we send Router Advertisement to announce the delegated /64 prefix, i.e. do SLAAC for devices connected to this interface to let themselves configure an IPv6 address.
- `Network.IPv6AcceptRA=no` ensures we won't do SLAAC for ourself on this interface, this is not necessarily needed but it would prevent a bad-behaving fake router under the LAN doing dirty stuffs breaking the network of the router itself.
- `Network.IPv6Forwarding=yes` ensure we allow IPv6 forwarding to this interface (unfortunately for IPv6 the option does not work as expected, addtional sysctl set up for IPv6 is needed, read further), we didn't set `Network.IPv4Forwarding=yes` as it's implicitly set by `Network.IPMasquerade=ipv4`.
- `DHCPPrefixDelegation.UplinkInterface=ens2f3` ensures we get IPv6 prefix from the WAN interface, `ens2f3`
- `DHCPPrefixDelegation.SubnetId=0` ensures we assign the first possible /64 from the prefix, in this case all of the /64 prefix we get. This is still needed even when we only have a single /64 prefix.
- `IPv6SendRA.OtherInformation=yes` ensures we tell LAN devices we have a DHCPv6 server, but we only ever send out addtional infos like routing, DNS, etc, but never a DHCPv6 address. We don't want a single device to have multiple IPv6 GLA address. 

#### Wireguard

I'd also want to connect this apartment network to my wireguard network, partially documented in [a previous blog post](../../06/14/complex-wireguard-setup-made-easy.html)

So let's create a `.netdev` and a `.network` for the wireguard interface.

The configs are generated by [wireguard-deployer](https://github.com/7Ji/wireguard-deployer) I recently wrote with a minimum config like following (it has changed a lot from my previous post, due to `hk1` losing its IPv6 connection):
```yaml
psk: false
netdev: 30-wireguard
network: 40-wireguard
peers:
  ali:
    ip: 192.168.77.1
    endpoint: ali.fuckblizzard.com # all other can connect to
    direct: [fuo, pdh, hk1, t16]
  fuo:
    netdev: 40-Wireguard
    network: 50-Wireguard
    ip: 192.168.77.2
    endpoint: 
      pdh: fuo.fuckblizzard.com
    forward:
      - 192.168.67.0/24
    keep: [ali]
    direct: [ali, pdh]
  pdh:
    ip: 192.168.77.3
    endpoint: 
      ^neighbor: pd4.fuckblizzard.com
      fuo: pd6.fuckblizzard.com
    forward:
      - 192.168.7.0/24
      - 192.168.15.0/24
      - 192.168.17.0/24
    keep: [ali]
    direct: [ali, fuo, t16]
  hk1:
    ip: 192.168.77.96
    endpoint:
      ^child: hk1.lan
    direct: [ali, pdh]
    keep: [ali, pdh]
    children:
      rz5:
        ip: 192.168.77.97
        endpoint: rz5.lan
      a7j:
        ip: 192.168.77.98
        endpoint: a7j.lan
      v7j:
        ip: 192.168.77.99
        endpoint: v7j.lan
  t16:
    ip: 192.168.77.128
    direct: [ali, pdh]
```


_`/etc/systemd/network/40-Wireguard.netdev`_

```conf
[NetDev]
Name=wg0
Kind=wireguard

[WireGuard]
ListenPort=51820
PrivateKeyFile=/etc/systemd/network/keys/wg/private-fuo

# ali
[WireGuardPeer]
PublicKey=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Endpoint=ali.fuckblizzard.com:51820
AllowedIPs=192.168.77.1
AllowedIPs=192.168.77.128
AllowedIPs=192.168.77.96
AllowedIPs=192.168.77.97
AllowedIPs=192.168.77.98
AllowedIPs=192.168.77.99
PersistentKeepalive=25

# pdh
[WireGuardPeer]
PublicKey=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Endpoint=pd6.fuckblizzard.com:51820
AllowedIPs=192.168.15.0/24
AllowedIPs=192.168.17.0/24
AllowedIPs=192.168.7.0/24
AllowedIPs=192.168.77.3
```

_`/etc/systemd/network/50-Wireguard.network`_

```conf
[Match]
Name=wg0

[Network]
Address=192.168.77.2/24
IPv4Forwarding=yes

[Link]
RequiredForOnline=no

[Route]
Destination=192.168.15.0/24
Scope=link

[Route]
Destination=192.168.17.0/24
Scope=link

[Route]
Destination=192.168.7.0/24
Scope=link
```

In which:
- `Network.IPv4Forwarding=yes` ensures IPv4 forwarding to this interface is allowed
- `Link.RequiredForOnline=no` ensures failed wireguard connection won't result in `systemd-networkd-wait-online` being blocked 


#### Global networkd config

The option `Network.IPv6Forwarding` in `.netdev` file sets `net.ipv6.conf.[interface].forwarding` to 1, similar to how it configs IPv4. 

However, for IPv6, the kernel explicitly checks `net.ipv6.conf.all.forwarding` to decide whether to do IPv6 forwarding, and only does so when it's set to 1. 

Per-interface forwarding on-off option is not a thing, and `net.ipv6.conf.[interface].forwarding` controls actually the interface-specific host/router behaviour (telling neighbors we're a router in Neighbour Advertisements with `IsRouter=1`), so instead of "we would do IPv6 forwarding", it's really "(telling others) we can do IPv6 forwarding".

Due to this, we need to set `Network.IPv6Forwarding=yes` in `/etc/systemd/networkd.conf` so networkd would set sysctl `net.ipv6.conf.all.forwarding` to 1. 

_`/etc/systemd/networkd.conf`_

```conf
[Network]
IPv6Forwarding=yes
```

I'd recommend against setting it in `/etc/sysctl.d`, setting it in `networkd.conf` and we could track all network settings in one place, setting it in `/etc/sysctl.d` and network sysctls would be scattered in two places and hard to track.

Reference: [systemd issue #33414](https://github.com/systemd/systemd/issues/33414)

#### Starting network

Now everything's configured, start the network by doing
```sh
sudo systemctl enable --now systemd-networkd
```

Let's also start a temporary DNS server, we need it before we finish setting up our own actual DNS server. systemd has `resolved` so let's use it for the temp job.

```sh
sudo systemctl start systemd-resolved
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

The network should come up like this (all public addresses obfuscated): 
```
> ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: bridge0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether aa:bb:cc:dd:ee:ff brd ff:ff:ff:ff:ff:ff
    inet 192.168.67.1/24 brd 192.168.67.255 scope global bridge0
       valid_lft forever preferred_lft forever
    inet6 1111:2222:3333:4444:xxxx:xxxx:xxxx:xxxx/64 metric 256 scope global dynamic mngtmpaddr 
       valid_lft 174254sec preferred_lft 87854sec
    inet6 fe80::xxxx:xxxx:xxxx:xxxx/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
3: ens2f0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq master bridge0 state DOWN group default qlen 1000
    link/ether bb:cc:dd:ee:ff:aa brd ff:ff:ff:ff:ff:ff
    altname enp2s0f0
4: ens2f1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq master bridge0 state DOWN group default qlen 1000
    link/ether cc:dd:ee:ff:aa:bb brd ff:ff:ff:ff:ff:ff
    altname enp2s0f1
5: ens2f2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq master bridge0 state DOWN group default qlen 1000
    link/ether dd:ee:ff:aa:bb:cc brd ff:ff:ff:ff:ff:ff
    altname enp2s0f2
6: ens2f3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether ee:ff:aa:bb:cc:dd brd ff:ff:ff:ff:ff:ff
    altname enp2s0f3
    inet 192.168.1.2/24 metric 1024 brd 192.168.1.255 scope global dynamic ens2f3
       valid_lft 244740sec preferred_lft 244740sec
    inet6 1111:2222:3333:5555:xxxx:xxxx:xxxx:xxxx/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 174254sec preferred_lft 87854sec
    inet6 fe80::xxxx:xxxx:xxxx:xxxx/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
7: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
    link/none 
    inet 192.168.77.2/24 scope global wg0
       valid_lft forever preferred_lft forever
8: eno8: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq master bridge0 state DOWN group default qlen 1000
    link/ether ff:aa:bb:cc:dd:ee brd ff:ff:ff:ff:ff:ff
    altname enp5s0f0
9: wlp4s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether ab:cd:ef:ab:cd:ef brd ff:ff:ff:ff:ff:ff
10: eno7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master bridge0 state UP group default qlen 1000
    link/ether cd:ef:ab:cd:ef:ab brd ff:ff:ff:ff:ff:ff
    altname enp5s0f1
11: eno6: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq master bridge0 state DOWN group default qlen 1000
    link/ether ef:ab:cd:ef:ab:cd brd ff:ff:ff:ff:ff:ff
    altname enp7s0f0
12: eno5: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq master bridge0 state DOWN group default qlen 1000
    link/ether df:ab:cd:ef:ab:cd brd ff:ff:ff:ff:ff:ff
    altname enp7s0f1
```

Remember to go back and set the MAC address for the bridge interface now that it's generated (here `aa:bb:cc:dd:ee:ff` on `bridge0`).

Verify the network by doing a simple `pacman -Syu` first, reboot if necessary.

Check the network connection, we now should have a fully working network connection on the router, and LAN device should have partially working network connection.

LAN devices currently should be able to:
- get an IPv6 adddress by SLAAC
- connect to network if we configure the IPv4 address and DNS manually 

So let's go further to config a DHCPv4 server and a DNS server to assin LAN IPs and resolve DNS queries as caching DNS server.

### DHCP server

Devices under the LAN can currently get IPv6 addresses by SLAAC but they can't get IPv4 address and they can't get DNS info. We would need a DHCPv4 server to assign IPv4 addresses to devices under the LAN and announce DNS infos, domain search suffixes, etc in the DHCP lease.

Let's install kea, the ISC's new reference DHCP server implementation after ISC DHCP

```
sudo pacman -S kea
```

kea provides DHCPv4, DHCPv6, DDNS and Control Daemon, we only need DHCPv4 currently.

Config kea DHCPv4 by modifying `/etc/kea/kea-dhcp4.conf`

My config looks like the following:

```json
{
"Dhcp4": {
    "interfaces-config": {
        "interfaces": [ "bridge0/192.168.67.1" ]
    },
    "control-socket": {
        "socket-type": "unix",
        "socket-name": "/tmp/kea4-ctrl-socket"
    },
    "lease-database": {
        "type": "memfile",
        "lfc-interval": 60
    },
    "expired-leases-processing": {
        "reclaim-timer-wait-time": 10,
        "flush-reclaimed-timer-wait-time": 25,
        "hold-reclaimed-time": 3600,
        "max-reclaim-leases": 100,
        "max-reclaim-time": 250,
        "unwarned-reclaim-cycles": 5
    },
    "renew-timer": 90,
    "rebind-timer": 180,
    "valid-lifetime": 600,
    "option-data": [
        {
            "name": "domain-name-servers",
            "data": "192.168.67.1, 192.168.67.1"
        },
        {
            "name": "domain-name",
            "data": "fuo.lan"
        },
        {
            "name": "domain-search",
            "data": "fuo.lan, lan"
        }
    ],
    "subnet4": [
        {
            "id": 1,
            "subnet": "192.168.67.0/24",
            "pools": [ { "pool": "192.168.67.192 - 192.168.67.254" } ],
            "option-data": [
                {
                    "name": "routers",
                    "data": "192.168.67.1"
                }
            ],
            "reservations": [
                {
                    "hw-address": "AA:BB:CC:DD:EE:FF",
                    "ip-address": "192.168.67.2",
                    "hostname": "switch-sirivision-2500m"
                },
                {
                    "hw-address": "BB:CC:DD:EE:FF:00",
                    "ip-address": "192.168.67.3",
                    "hostname": "ap-tplink-ax6000"
                },
                ......
            ]
        }
    ],
    "loggers": [
    {
        "name": "kea-dhcp4",
        "output-options": [
            {
                "output": "/var/log/kea-dhcp4.log"
            }
        ],
        "severity": "INFO"
    }
  ]
}
}
```
In which (all `Dhcp4.` prefix are omitted):
- `interfaces-config.interfaces = [ "bridge0/192.168.67.1" ]` tells kea to do DHCPv4 on `bridge0`, listening on `192.168.67.1`
- `lease-database.type = memfile` and `lease-database.lfc-interval = 60` tells kea to use a in-memory databse and flush it to an on-disk file per-minute.
- `renew-timer = 90` tells kea to let clients to renew their DHCP lease per 90 seconds
- `rebind-timer = 180` tells kea to try to re-bind existing DHCP leases per 3 minutes
- `valid-lifetime = 600` tells kea to send out DHCP leases with 10-minute valid lifetime
- `option-data` tells kea to send addtional infos in the DHCP lease
  - `domain-name-servers` configs DNS for clients, in this case also the router itself
  - `domain-name` configs a rDNS record for clients, in this case resolving `fuo.lan` to router itself
  - `domain-search` configs the local domain serach suffixes for clients, in this case both `fuo.lan` and `lan`, so a dot-less domain record would be searched by itself first and then with suffixes, e.g. `o5p` as `o5p` then as `o5p.fuo.lan` then as `o5p.lan`
- `subnet4` defines a list of IPv4 subnets to do DHCP in
  - `subnet` defines the actual subnet, in this case `192.168.67.0/24`
  - `pool` defines a list of address ranges to use, in this case `192.168.67.192 - 192.168.67.254`
  - `option-data` defines additional `option-data` on top of global `option-data` to send, in this case we send out `routers` to config a gateway and default route rule for clients
  - `reservations` defines reserved leases for certain clients

Start kea's DHCPv4 server

```sh
sudo systemctl enable --now kea-dhcp4
```

LAN devices should now be able to get IPv4 address, routing infos and DNS servers by DHCPv4. They should be able to ping Internet IPv4 addresses, e.g. `ping 8.8.8.8`, but they can't resolve domain names yet unless they configure an Internet DNS server, as we've set the router itself as DNS server in DHCPv4.


### DNS server

We would need a DNS server that functions both as caching server (for generic domains and other `.lan` domains in my wireguard network) and authoritative server (for our `fuo.lan` zone).

Let's install bind9, the ISC's reference DNS server implementation

```
sudo pacman -S bind
```

Config bind9 named by modifying `/etc/named.conf`

My config looks like the following:
```conf
options {
    directory "/var/named";
    pid-file "/run/named/named.pid";

    listen-on-v6  { none; };
    listen-on { 192.168.67.1; 192.168.77.2; 127.0.0.1; };

    allow-recursion { 192.168.67.0/24; 192.168.77.0/24; 127.0.0.1; };
    allow-transfer { none; };
    allow-update { none; };

    recursion yes;
    auth-nxdomain no;
    dnssec-validation no;

    forwarders { 114.114.114.114; 114.114.115.115; };

    version none;
    hostname none;
    server-id none;
};

include "rndc.conf";

zone "localhost" IN {
    type master;
    file "localhost.zone";
};

zone "0.0.127.in-addr.arpa" IN {
    type master;
    file "127.0.0.zone";
};

zone "1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa" {
    type master;
    file "localhost.ip6.zone";
};


zone "7ji.lan" IN {
    type master;
    file "lan.7ji.zone";
};

include "lan.fuo.dhcp.key";

zone "fuo.lan" IN {
    type master;
    file "lan.fuo.zone";
    update-policy {
        grant dhcp-fuo-lan-key wildcard *.fuo.lan A DHCID;
    };
};

zone "67.168.192.in-addr.arpa" {
    type master;
    file "lan.fuo.rdns.zone";
    update-policy {
        grant dhcp-fuo-lan-key wildcard *.67.168.192.in-addr.arpa PTR DHCID;
    };
};

zone "wg7.lan" {
    type forward;
    forward only;
    forwarders {
        192.168.77.1;
    };
};

zone "pdh.lan" {
    type forward;
    forward only;
    forwarders {
        192.168.77.3;
    };
};

zone "lks.lan" {
    type forward;
    forward only;
    forwarders {
        192.168.77.96;
    };
};
```

In the `options` section:
- `listen-on-v6 { none; }; ` disables DNSv6 service and named won't bind to and listen on an IPv6 address
- `listen-on { 192.168.67.1; 192.168.77.2; 127.0.0.1; };` limits the IPv4 addresses we bind to and listen on: only LAN IPv4, wireguard IPv4, and localhost
- `allow-transfer { none; };` and `allow-update { none; };` ensures we're the master DNS server for zones we manage
- `recursion yes;` enables the server to function as a DNS caching server
- `auth-nxdomain no;` ensures we won't touch `AA` bit in NXDOMAIN response, i.e. we won't pretend to be authoritative for zones we don't own
- `dnssec-validation no;` disables DNSSEC
- `forwarders { 114.114.114.114; 114.114.115.115; };` sets upstream DNS servers to refer to for domain zones we don't control
- `version none;` ensures we won't return server version for a query of the name `version.bind` with type `TXT` and class `CHAO`, so we're mostly transparent for clients
- `hostname none;` ensures we won't return server for a query of the name `hostname.bind` with type `TXT` and class `CHAOS`, so we're mostly transparent for clients
- `server-id none;` ensures we won't return server ID for a Name Server Identifier (NSID) query, or a query of the name `ID.SERVER` with type `TXT` and class `CHAOS`, so we're mostly transparent for clients


2 `include` sections are generated and stored privately for security (`tee /dev/stderr` is only for demostrating, you don't actually need it when running)
```sh
> tsig-keygen dhcp-fuo-lan-key | tee /dev/stderr | sudo install --mode 640 --group named /dev/stdin /var/named/lan.fuo.dhcp.key
key "dhcp-fuo-lan-key" {
        algorithm hmac-sha256;
        secret "xnk+ZYUhCQCEl19hIsNLgqswMBzsJZf62vrlaxuwTEU=";
};
> printf '%s\n' '{' '	"name": "dhcp-fuo-lan-key",' '	"algorithm": "hmac-sha256",' '	"secret": "'$(sudo sed -n 's/^.\+secret "\(.\+\)";$/\1/p' /var/named/lan.fuo.dhcp.key)'"' '}' | tee /dev/stderr | sudo install --mode 600 /dev/stdin /etc/kea/kea-dhcp-fuo-lan.key
{
        "name": "dhcp-fuo-lan-key",
        "algorithm": "hmac-sha256",
        "secret": "xnk+ZYUhCQCEl19hIsNLgqswMBzsJZf62vrlaxuwTEU="
}
> rndc-confgen | tee /dev/stderr | sudo install --mode 400 /dev/stdin /etc/rndc.conf.temp
# Start of rndc.conf
key "rndc-key" {
        algorithm hmac-sha256;
        secret "E9vh+qxVIitnSrEcvBWbbciTsf2kquLil4V5XNgRgR4=";
};
    
options {
        default-key "rndc-key";
        default-server 127.0.0.1;
        default-port 953;
};
# End of rndc.conf

# Use with the following in named.conf, adjusting the allow list as needed:
# key "rndc-key" {
#       algorithm hmac-sha256;
#       secret "E9vh+qxVIitnSrEcvBWbbciTsf2kquLil4V5XNgRgR4=";
# };
# 
# controls {
#       inet 127.0.0.1 port 953
#               allow { 127.0.0.1; } keys { "rndc-key"; };
# };
# End of named.conf
> sudo grep -v '^#' /etc/rndc.conf.temp | grep -v '^$' | tee /dev/stderr | sudo install --mode 600 /dev/stdin /etc/rndc.conf
key "rndc-key" {
        algorithm hmac-sha256;
        secret "E9vh+qxVIitnSrEcvBWbbciTsf2kquLil4V5XNgRgR4=";
};
options {
        default-key "rndc-key";
        default-server 127.0.0.1;
        default-port 953;
};
> sudo grep '^#' /etc/rndc.conf.temp | grep -v '\.conf' | cut -c 3- | tee /dev/stderr | sudo install --mode 640 --group named /dev/stdin /var/named/rndc.conf
key "rndc-key" {
        algorithm hmac-sha256;
        secret "E9vh+qxVIitnSrEcvBWbbciTsf2kquLil4V5XNgRgR4=";
};

controls {
        inet 127.0.0.1 port 953
                allow { 127.0.0.1; } keys { "rndc-key"; };
};
> sudo rm /etc/rndc.conf.temp
```

Various zones have different definitions:
- zone `localhost`, zone `0.0.127.in-addr.arpa` and zone `1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa` come from the default config, keep them as-is
- zone `wg7.lan`, zone `pdh.lan`, zone `lks.lan` are forwarded to other routers in the wireguard network, the sole forwarder for each zone is their WireGuard IP.
- zone `7ji.lan` is the local CNAME-only zone that resolves some service domains to CNAME `fuo.lan` domains, so I can use e.g. `repo.7ji.lan` in different places resolved to different LAN domains, `repo.fuo.lan` at my apartment, `repo.pdh.lan` at my parents' house, etc. File `/var/named/lan.7ji.zone` shall be created with owner:group set to `root:named` and permission mode `640` so named can only read it but not update it.

    ```sh
    echo '@                       SOA     @ root (
                                    1          ; serial
                                    0          ; refresh (immediately)
                                    0          ; retry (immediately)
                                    604800     ; expire (1 week)
                                    0          ; minimum (immediately)
                                    )
                            NS      @
                            A       192.168.67.1
    *                       CNAME   fuo.lan.' | sudo install --mode 640 --group named /dev/stdin /var/named/lan.7ji.zone
    ```
    We've not set TTL for any record. TTL would always default to SOA minimum, here 0. This means we don't want any client to cache multi-place `7ji.lan` lookup results.
- zone `fuo.lan` is the apartment local domain zone, we would define some CNAME records to resolve them to DDNS domains. File `/var/named/lan.fuo.zone` shall be created with owner:group set to `named:named` and permission mode `660` so named can both read it and update it:

    ```sh
    echo '@                       SOA     @ root (
                                    1          ; serial
                                    0          ; refresh (immediately)
                                    0          ; retry (immediately)
                                    604800     ; expire (1 week)
                                    0          ; minimum (immediately)
                                    )
                            NS      @
                            A       192.168.67.1
    fuo                     CNAME   @
    xray                    CNAME   @
    repo                    CNAME   @
    git                     CNAME   wtr
    gmr                     CNAME   opi
    bpi                     CNAME   server-bpi-m5
    o5p                     CNAME   server-opi-5plus
    opi                     CNAME   server-opi-5
    wtr                     CNAME   server-aoostar-wtr-pro' | sudo install --mode 660 --owner named --group named /dev/stdin /var/named/lan.fuo.zone
    ```
    We've not set TTL for any record. TTL would always default to SOA minimum, here 0. This means we don't want any client to cache LAN `fuo.lan` lookup results.
- zone `67.168.192.in-addr.arpa` is the apartment reverse DNS domain zone. File `/var/named/lan.fuo.rdns.zone` shall be created with owner:group set to `named:named` and permission mode `660` so named can both read it and update it.

    ```sh
    echo '@                       SOA     @ root (
                                    1          ; serial
                                    0          ; refresh (immediately)
                                    0          ; retry (immediately)
                                    604800     ; expire (1 week)
                                    0          ; minimum (immediately)
                                    )
                            NS      fuo.lan.
    1                       PTR     fuo.lan.' | sudo install --mode 660 --owner named --group named /dev/stdin /var/named/lan.fuo.rdns.zone
    ```
    We've not set TTL for any record. TTL would always default to SOA minimum, here 0. This means we don't want any client to cache LAN `192.168.67.y` reverse lookup results.

With bind9 configured up it's time to start the named server:
```
sudo systemctl enable --now named
```
With bind9 named running, we should be able to (by using `dig` to query `127.0.0.1`):
- Resolve public domains, e.g. `dig github.com @127.0.0.1` -> `A 20.205.243.166`
- Resolve set LAN domains, e.g. `dig xray.fuo.lan @127.0.0.1` -> `CNAME fuo.fuo.lan` -> `CNAME fuo.lan` -> `A 192.168.67.1`
- Resolve set multi-place LAN domains, e.g. `dig repo.7ji.lan @127.0.0.1` -> `CNAME fuo.lan` -> `A 192.168.67.1`

Now let's say goodbye to the temporary systemd-resolved server
```
sudo systemctl stop systemd-resolved
sudo rm /etc/resolv.conf
```

The router should use itself as the DNS server.

```
echo 'nameserver 127.0.0.1
search fuo.lan' | sudo tee /etc/resolv.conf
```

### local DDNS (DHCP + DNS integration)

kea and bind9 can be configured to do DDNS for zones, in this case we want `[host].fuo.lan` resolved to LAN addresses, and a corresponding rDNS zone resolved to domains.

Modify kea's DHCPv4 server config to add DDNS options:
```json
{
"Dhcp4": {
    ......
    "dhcp-ddns": {
        "enable-updates": true
    },
    "ddns-qualifying-suffix": "fuo.lan",
    "ddns-override-client-update": true,
    ......
}
}
```

Restart kea's DHCPv4 server

```
sudo systemctl restart kea-dhcp4
```

Config kea's DHCP DDNS server by modifying `/etc/kea/kea-dhcp-ddns.conf`

My config looks like the following:

```json
"DhcpDdns":
{
  "ip-address": "127.0.0.1",
  "port": 53001,
  "control-socket": {
      "socket-type": "unix",
      "socket-name": "/tmp/kea-ddns-ctrl-socket"
  },
  "tsig-keys": [
    <?include "/etc/kea/kea-dhcp-fuo-lan.key"?>
  ],
  "forward-ddns" : {
      "ddns-domains": [{
          "name": "fuo.lan.",
          "key-name": "dhcp-fuo-lan",
          "dns-servers": [{
              "ip-address": "127.0.0.1"
          }]
      }]
  },
  "reverse-ddns" : {
      "ddns-domains": [{
          "name": "67.168.192.in-addr.arpa.",
          "key-name": "dhcp-fuo-lan",
          "dns-servers": [{
              "ip-address": "127.0.0.1"
          }]
      }]
  },
  "loggers": [
    {
        "name": "kea-dhcp-ddns",
        "output-options": [
            {
                "output": "/var/log/kea-ddns.log"
            }
        ],
        "severity": "INFO",
        "debuglevel": 0
    }
  ]
}
}
```

Start kea's DDNS server
```sh
sudo systemctl enable --now kea-dhcp-ddns
```

Test whether DDNS work by `tail -f /var/log/kea-ddns.log` on router and restarting network on a LAN client. When a client requests and get a DHCP lease, an A record `[hostname].fuo.lan.` to the IP and a PTR record `[suffix].67.168.192.in-addr.arpa.` to the A record domain should be added automatically.

### Pacman caching

I need this as I have a LOT of Arch clients in LAN and want to do `pacman -Syu` at maximum LAN bandwidth (here 10G)

Install `pacoloco`, a pacman caching server

```
sudo pacman -S pacoloco
```

Modify its config in `/etc/pacoloco.yaml`

Mine looks like the following:
```yaml
download_timeout: 3600
purge_files_after: 2592000
repos:
  archlinux:x86_64:
    urls: &urls_archlinux
      - http://mirrors.ustc.edu.cn/archlinux
      - http://mirrors.tuna.tsinghua.edu.cn/archlinux
  archlinuxarm:aarch64:
    urls: &urls_archlinuxarm
      - http://mirrors.ustc.edu.cn/archlinuxarm
      - http://mirrors.tuna.tsinghua.edu.cn/archlinuxarm
  archlinuxarm:armv7h:
    urls: *urls_archlinuxarm
  archlinuxcn:aarch64:
    urls: &urls_archlinuxcn
      - http://mirrors.ustc.edu.cn/archlinuxcn
      - http://mirrors.tuna.tsinghua.edu.cn/archlinuxcn
  archlinuxcn:any:
    urls: *urls_archlinuxcn
  archlinuxcn:arm:
    urls: *urls_archlinuxcn
  archlinuxcn:armv6h:
    urls: *urls_archlinuxcn
  archlinuxcn:armv7h:
    urls: *urls_archlinuxcn
  archlinuxcn:i686:
    urls: *urls_archlinuxcn
  archlinuxcn:x86_64:
    urls: *urls_archlinuxcn
  arch4edu:aarch64:
    urls: &urls_arch4edu
      - http://mirrors.ustc.edu.cn/arch4edu
      - http://mirrors.tuna.tsinghua.edu.cn/arch4edu
  arch4edu:any:
    urls: *urls_arch4edu
  arch4edu:x86_64:
    urls: *urls_arch4edu
prefetch:
  cron: 0 0 3 * * * *
```

Start pacoloco

```sh
sudo systemctl enable --now pacoloco
```

Clients in LAN including the router itself should be able to use e.g. `Server = http://fuo.lan:9129/archlinux/$repo/os/$arch` in their `pacman.conf`, but I don't quite like ports in repo URLs.

Let's go further by listen on `repo.7ji.lan` so a device can be moved to different LAN but still uses the caching server, and sanitize the URL a bit so multiple repos can use a same mirrorlist.

Install `nginx`, a HTTP server

```
sudo pacman -S nginx
```

Modify `/etc/nginx.conf` to include site configs
```conf 
http {
    ....
    include sites-enabled/*.conf;
    ...
}
```

Create a repo config `/etc/nginx/sites-available/repo.conf`:
```conf
server {
    listen 80;
    charset UTF-8;
    server_name repo.7ji.lan;

    rewrite ^/(archlinuxarm|archlinuxcn|arch4edu)/([^/]+)/(.+)$ http://repo.fuo.lan:9129/repo/$1:$2/$2/$3 permanent;
    rewrite ^/archlinux/([^/]+)/os/([^/]+)/(.+)$ http://repo.fuo.lan:9129/repo/archlinux:$2/$1/os/$2/$3 permanent;

    location / {
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;

        root /srv/http/repo;
    }
}
```

The section `location /` can be omitted if you don't have local-only repos. I have a repo `7Ji` that I store locally, so I did addtionally:
```sh
sudo mkdir -p /srv/http/repo/7Ji
```

The config would need to be linked to another folder so it would be recognized:
```sh
sudo ln -s ../repo.conf /etc/nginx/sites-enabled/
```

Start nginx:
```sh
sudo systemctl enable --now nginx
```

With this config, clients can use the folloiwng `/etc/pacman.d/mirrorlist`:
```conf
Server = http://repo.7ji.lan/archlinux/$repo/os/$arch
```
and the following `/etc/pacman.d/mirrorlist-3rdparty`:
```conf
Server = http://repo.7ji.lan/$repo/$arch
```
and set up their `/etc/pacman.conf` simply like
```conf
[core]
Include = /etc/pacman.d/mirrorlist

[extra]
Include = /etc/pacman.d/mirrorlist

[multilib]
Include = /etc/pacman.d/mirrorlist

[7Ji]
Include = /etc/pacman.d/mirrorlist-3rdparty

[archlinuxcn]
Include = /etc/pacman.d/mirrorlist-3rdparty

[arch4edu]
Include = /etc/pacman.d/mirrorlist-3rdparty
```

Note for ALARM `/etc/pacman.d/mirrorlist` would be a little bit different:
```conf
Server = http://repo.7ji.lan/archlinuxarm/$arch/$repo
```

### Firewall

Install `nftables`, the modern firewall

```
sudo pacman -S nftables
```

Modify `/etc/nftables.conf` as you like it. Notably you would need to open `dhcpv6-client` ports on WAN side even if you want to limit WAN access, otherwise you would not be able to get DHCPv6 routing info and prefix delegation.

Mine looks like the following:
```nft
define port_wireguard = 51820;
define port_transmission = 51413;
define port_qbittorrent = 60726;
define open_tcp_udp = { dhcpv6-client };
define open_tcp = { $open_tcp_udp, ssh };
define open_udp = { $open_tcp_udp, $port_wireguard };
define allow_forward_wtr_tcp_udp = { $port_transmission, $port_qbittorrent };
define allow_forward_wtr_tcp = { $allow_forward_wtr_tcp_udp, ssh };
define allow_forward_wtr_udp = { $allow_forward_wtr_tcp_udp };
destroy table inet filter
table inet filter {
  chain input {
    type filter hook input priority filter
    policy drop

    ct state invalid drop comment "early drop of invalid connections"
    ct state {established, related} accept comment "allow tracked connections"
    iifname { lo, bridge0, wg0 } accept comment "allow from loopback, lan, and wireguard"
    ip protocol icmp accept comment "allow icmp"
    meta l4proto ipv6-icmp accept comment "allow icmp v6"
    tcp dport $open_tcp accept comment "allow dhcp v6 client, sshd"
    udp dport $open_udp accept comment "allow dhcp v6 client, wireguard"
    pkttype host limit rate 5/second counter reject with icmpx type admin-prohibited
    counter
  }
  chain forward {
    type filter hook forward priority filter
    ct state established,related accept comment "allow forwarded established and related flows"
    iifname bridge0 accept comment "allow lan forwarding to wan and wireguard"
    iifname wg0 accept comment "allow wireguard forwarding to lan and wan"
    iifname ens2f3 oifname bridge0 jump forward_wan_lan comment "allow certain wan-lan forwarding"
    policy drop
  }
  chain forward_wan_lan {
    ip6 daddr & ::ffff:ffff:ffff:ffff == ::707f:3ff:feb9:ee5c jump forward_wtr
    # return to chain forward
  }
  chain forward_wtr {
    tcp dport $allow_forward_wtr_tcp accept comment "allow transmission, sshd"
    udp dport $allow_forward_wtr_udp accept comment "allow transmission"
    # return to chain forward_wan_lan
  }
}
```

Start nftables firewall

```
sudo systemctl enable --now nftables
```

### End

With the above setup you should have a stable and secure network structure. We could also config transparent proxy, but let's leave it for another post.