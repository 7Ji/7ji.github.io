---
layout: post
title:  "Complex WireGuard Setup made easy with wireguard-deployer"
date:   2024-06-14 11:30:00 +0800
categories: networking
---

## Background

It's been a few months since I've moved out from my hometown to work in a different city. As both a data hoarder and a device hoarder I've brought a lot of devices with me, but many of them are still left in my hometown. 

Some of my devices, including some servers, are left in my parents' home, most of others that I use often are here in my apartment, but there're still some random devices that my small apartment can't hold, and I had to move these to company, either piling them on my office desk or in server room.

Recall that I have so many devices that I need to have my own Arch Linux mirror (multiple proxy mirrors set up using pacoloco to be more precise): I have the following devices scattered all around different places that I need access to:

- Multiple devices in my parents' home
  - BananaPi BPI-R4 with OpenWrt as router
  - Network video recorder
  - 3 IP cams (in case I want real-time feed)
  - NUC as downloader and remote backup server
- Multiple devices in my apartment
  - BananaPi BPI-R4 with OpenWrt as router
  - Orange Pi 5 Plus as git server + Arch repo caching proxy + aarch64 repo builder
  - Orange Pi 5 as git mirrorer + aarch64 distcc volunteer
  - BananaPi BPI-M5 as downloader + aarch64 distcc volunteer
  - Main desktop PC, sometimes as a build server + amd64 distcc volunteer
- Multiple devices at my work place
  - 5 aarch64 boxes with ALARM as aarch64 distcc volunteers
  - An Arch VM running on shared server with as Arch repo caching proxy + x86_64 repo builder + x86_64 distcc volunteer
  - Alt desktop PC, where work-time development and some off-time FOSS development happens
  - Old laptop with broken screen, as remote Windows server for Win-only stuffs
- Multiple portable devices
  - Laptop, where most off-time FOSS development happens
  - Edge S with Lineage OS as SSH client when neither the laptops or desktops are in reach
- An ALI ECS as repo syncing forwarder

These only include the devices that I want direct access to, and do not include devices that I don't trust or allow outgoing traffic from (e.g. multiple of my consoles, setup boxes, phones, etc)

I (or rather, my digital presence) would be in any of the above places, and would want to connect to devices in other places with minimum overhead.

## Without WireGuard

For a long time, until the moveout, as the ISP provides public IPv4 + IPv6 at my parents', I used to connect to most of the stuffs with simply SSH: if there's a new device I just `ssh-keygen` a new key pair and I can forward some public IPv4 stuffs and only need to add several IPv6 suffix + port connection rules.

The old method turns out to be more and more annoying as I have to connect to many devices behind firewalls: rules at my parents' need not to be updated; but the landlord's router only has public IPv6 and I could only get a /64 suffix, and had to create multiple rules, each opening suffix:22/-64 for a device; then there's my work place, with public IPv4 + IPv6, but of course I don't want to expose either stuffs from company network into public, or my networks into company network.

## WireGuard manually done

Fortunately there is WireGuard, for the purpose of a LAN-like connection experience WireGuard is currently the most elegant solution, as a simple yet powerful VPN implementation in mainline kernel it shadows all other VPS solutions.

As I want site-to-site full-mesh connection, with each site freely connect to both other in-wireguard hosts and non-wireguard hosts, I needed to set up most "site"s on "router"s, this is done tediously, e.g.
- On the OpenWrt router at my parents':
  - Install `luci-proto-wireguard` (which pulls in `kmod-wireguard` and `wireguard-tools` as deps)
  - Reboot (a strange caveat, only reloading `LuCI` won't make `wireguard` proto recognised)
  - Create `wg0` wireguard interface in `wg0` firewall zone
  - Add all other sites and their allowed IPs (one for my aparment allowing 192.168.67.0/24, one for VPs, etc...) as peers, each pair having a pre-shared key
  - Allow `wg0` forwarding into all other zones (`home` with 192.168.7.0/24, `watcher` with 192.168.15.0/24, `vm` with 192.168.17.0/24)
- On one of the aarch64 boxes at my work place that functions as a "router":
  - Create a `systemd-networkd` .netdev creating interface `wg0` on L2, adding all other sites and their allowed IPs, and all other my devices at my work place, each pair having a pre-shared key;
  - Create a `systemd-networkd` .network configuring interface `wg0` on L3, setting address and adding routes for non-wireguard allowed IPs
- On one of the devices at my work place that does not function as a "router":
  - Create a `systemd-networkd` .netdev creating interface `wg0` on L2, adding the "router" and all other my devices at my work place, each pair having a pre-shared key;
  - Create a `systemd-networkd` .network configuring interface `wg0` on L3, setting address and adding routes for non-wireguard allowed IPs

Note that I didn't use any of the existing generation tools as:
- I'm mixing both `systemd-networkd` and `OpenWrt`, and I'm not using `wg-quick`
- For the purpose of not letting work place public traffic get into the WireGuard network, all my devices at work place are direct members of the network. They're all direct members of the work place network, not behind any NAT router / firewall, as they're scattered around multiple places.
- There're two seperate full-mesh networks connected via a single node (one with parents' router + apartment router + VPS + edge "router", another one with edge "router" + all other of my devices at company). Crossing the edge "router" is necessary.
- A single node would have different endpoints depending on the connection source (namely the edge router, public domain for other routers, private domain for other of my work place devices)
- Not all members can connect to others with a configured endpoint directly, namely the ali VPS, as it does not have an IPv6 address it could only know the v4 endpoint address after receiving an incoming connection from my apartment router through NAT

This all turns out to be extra tedious. Adding a single device to the work place subnet is especially annoying as I would have to create almost 10 more pre-shared keys and update almost all of other devices.

So I decided to (and had to) summarize the topology and figure out a way to configure this in a convenient way

## Topology

The topology is basically like the following pic, with unrelated switches and other devices stripped:

(I also didn't bother to draw every single mesh lines in the work place network, as that would make the pic too messy)

<img src="../../../../res/wireguard-topology.png">

## With wireguard-deployer

For my sanity I decided to write a config and key generator that yields easily deployable config tarballs, the project is mostly written during the Dragon Boat Festival, and it's open sourced [on GitHub](https://github.com/7Ji/wireguard-deployer)

I can desribe my setup in a single .yaml file like the following:
```yaml
netdev: 30-wireguard
network: 40-wireguard
peers:
  ali:
    ip: 192.168.77.1
    endpoint: ali.fuckblizzard.com
  fuo:
    ip: 192.168.77.2
    endpoint: 
      ^neighbor: fuo.fuckblizzard.com
      ali: ''
    forward:
      - 192.168.67.0/24
      - fd60:c3e0:a2d7::/48
    keep: [ali]
  pdh:
    ip: 192.168.77.3
    endpoint: 
      ^neighbor: pd6.fuckblizzard.com
      ali: pd4.fuckblizzard.com
    forward:
      - 192.168.7.0/24
      - 192.168.15.0/24
      - 192.168.17.0/24
      - fdb5:c701:19a6::/48
  hk1:
    ip: 192.168.77.96
    endpoint: 
      ^neighbor: hk1.fuckblizzard.com
      ^child: hk1.lan
      ali: ''
    keep: [ali]
    children:
      rz5:
        ip: 192.168.77.97
        endpoint: rz5.lan
      v7j:
        ip: 192.168.77.98
        endpoint: v7j.lan
      l3a:
        ip: 192.168.77.99
        endpoint: l3a.lan
      cm2:
        ip: 192.168.77.100
        endpoint: cm2.lan
      r33:
        ip: 192.168.77.101
        endpoint: r33.lan
      mi3:
        ip: 192.168.77.102
        endpoint: mi3.lan
```
And this would yield the following folder structure:
```
wireguard
├── configs
│   ├── ali
│   │   ├── 30-wireguard.netdev
│   │   ├── 40-wireguard.network
│   │   └── keys
│   │       └── wg
│   │           ├── pre-shared-ali-fuo
│   │           ├── pre-shared-ali-hk1
│   │           ├── pre-shared-ali-pdh
│   │           └── private-ali
│   ├── ali.openwrt.conf
│   ├── ali.tar
│   ├── cm2
│   │   ├── 30-wireguard.netdev
│   │   ├── 40-wireguard.network
│   │   └── keys
│   │       └── wg
│   │           ├── pre-shared-cm2-hk1
│   │           ├── pre-shared-cm2-l3a
│   │           ├── pre-shared-cm2-mi3
│   │           ├── pre-shared-cm2-r33
│   │           ├── pre-shared-cm2-rz5
│   │           ├── pre-shared-cm2-v7j
│   │           └── private-cm2
│   ├── cm2.openwrt.conf
│   ├── cm2.tar
│   ├── fuo
│   │   ├── 30-wireguard.netdev
│   │   ├── 40-wireguard.network
│   │   └── keys
│   │       └── wg
│   │           ├── pre-shared-ali-fuo
│   │           ├── pre-shared-fuo-hk1
│   │           ├── pre-shared-fuo-pdh
│   │           └── private-fuo
│   ├── fuo.openwrt.conf
│   ├── fuo.tar
│   ├── hk1
│   │   ├── 30-wireguard.netdev
│   │   ├── 40-wireguard.network
│   │   └── keys
│   │       └── wg
│   │           ├── pre-shared-ali-hk1
│   │           ├── pre-shared-cm2-hk1
│   │           ├── pre-shared-fuo-hk1
│   │           ├── pre-shared-hk1-l3a
│   │           ├── pre-shared-hk1-mi3
│   │           ├── pre-shared-hk1-pdh
│   │           ├── pre-shared-hk1-r33
│   │           ├── pre-shared-hk1-rz5
│   │           ├── pre-shared-hk1-v7j
│   │           └── private-hk1
│   ├── hk1.openwrt.conf
│   ├── hk1.tar
│   ├── l3a
│   │   ├── 30-wireguard.netdev
│   │   ├── 40-wireguard.network
│   │   └── keys
│   │       └── wg
│   │           ├── pre-shared-cm2-l3a
│   │           ├── pre-shared-hk1-l3a
│   │           ├── pre-shared-l3a-mi3
│   │           ├── pre-shared-l3a-r33
│   │           ├── pre-shared-l3a-rz5
│   │           ├── pre-shared-l3a-v7j
│   │           └── private-l3a
│   ├── l3a.openwrt.conf
│   ├── l3a.tar
│   ├── mi3
│   │   ├── 30-wireguard.netdev
│   │   ├── 40-wireguard.network
│   │   └── keys
│   │       └── wg
│   │           ├── pre-shared-cm2-mi3
│   │           ├── pre-shared-hk1-mi3
│   │           ├── pre-shared-l3a-mi3
│   │           ├── pre-shared-mi3-r33
│   │           ├── pre-shared-mi3-rz5
│   │           ├── pre-shared-mi3-v7j
│   │           └── private-mi3
│   ├── mi3.openwrt.conf
│   ├── mi3.tar
│   ├── pdh
│   │   ├── 30-wireguard.netdev
│   │   ├── 40-wireguard.network
│   │   └── keys
│   │       └── wg
│   │           ├── pre-shared-ali-pdh
│   │           ├── pre-shared-fuo-pdh
│   │           ├── pre-shared-hk1-pdh
│   │           └── private-pdh
│   ├── pdh.openwrt.conf
│   ├── pdh.tar
│   ├── r33
│   │   ├── 30-wireguard.netdev
│   │   ├── 40-wireguard.network
│   │   └── keys
│   │       └── wg
│   │           ├── pre-shared-cm2-r33
│   │           ├── pre-shared-hk1-r33
│   │           ├── pre-shared-l3a-r33
│   │           ├── pre-shared-mi3-r33
│   │           ├── pre-shared-r33-rz5
│   │           ├── pre-shared-r33-v7j
│   │           └── private-r33
│   ├── r33.openwrt.conf
│   ├── r33.tar
│   ├── rz5
│   │   ├── 30-wireguard.netdev
│   │   ├── 40-wireguard.network
│   │   └── keys
│   │       └── wg
│   │           ├── pre-shared-cm2-rz5
│   │           ├── pre-shared-hk1-rz5
│   │           ├── pre-shared-l3a-rz5
│   │           ├── pre-shared-mi3-rz5
│   │           ├── pre-shared-r33-rz5
│   │           ├── pre-shared-rz5-v7j
│   │           └── private-rz5
│   ├── rz5.openwrt.conf
│   ├── rz5.tar
│   ├── v7j
│   │   ├── 30-wireguard.netdev
│   │   ├── 40-wireguard.network
│   │   └── keys
│   │       └── wg
│   │           ├── pre-shared-cm2-v7j
│   │           ├── pre-shared-hk1-v7j
│   │           ├── pre-shared-l3a-v7j
│   │           ├── pre-shared-mi3-v7j
│   │           ├── pre-shared-r33-v7j
│   │           ├── pre-shared-rz5-v7j
│   │           └── private-v7j
│   ├── v7j.openwrt.conf
│   └── v7j.tar
└── keys
    ├── pre-shared-ali-fuo
    ├── pre-shared-ali-hk1
    ├── pre-shared-ali-pdh
    ├── pre-shared-cm2-hk1
    ├── pre-shared-cm2-l3a
    ├── pre-shared-cm2-mi3
    ├── pre-shared-cm2-r33
    ├── pre-shared-cm2-rz5
    ├── pre-shared-cm2-v7j
    ├── pre-shared-fuo-hk1
    ├── pre-shared-fuo-pdh
    ├── pre-shared-hk1-l3a
    ├── pre-shared-hk1-mi3
    ├── pre-shared-hk1-pdh
    ├── pre-shared-hk1-r33
    ├── pre-shared-hk1-rz5
    ├── pre-shared-hk1-v7j
    ├── pre-shared-l3a-mi3
    ├── pre-shared-l3a-r33
    ├── pre-shared-l3a-rz5
    ├── pre-shared-l3a-v7j
    ├── pre-shared-mi3-r33
    ├── pre-shared-mi3-rz5
    ├── pre-shared-mi3-v7j
    ├── pre-shared-r33-rz5
    ├── pre-shared-r33-v7j
    ├── pre-shared-rz5-v7j
    ├── private-ali
    ├── private-cm2
    ├── private-fuo
    ├── private-hk1
    ├── private-l3a
    ├── private-mi3
    ├── private-pdh
    ├── private-r33
    ├── private-rz5
    └── private-v7j
```

To deploy this is very simple:
- For `systemd-networkd` devices, send the corresponding `configs/[name].tar`, and extract it at `/etc/systemd/network`. As the entries already have sane permission setups (owner `root:systemd-network` permission `0640` for keys) you don't need to change permissions manually, e.g.:
  ```
  ssh hk1 'sudo tar -C /etc/systemd/network -xv && sudo systemctl restart systemd-networkd' < hk1.tar
  ```
  would configure the host `hk1` quickly
- For current device running `systemd-networkd` device, extract and reconfigure the network similarly:
  ```
  sudo tar -C /etc/systemd/network -xvf rz5.tar && sudo systemctl restart systemd-networkd
  ```
- For OpenWrt devices, paste the correponding content of `configs/[name].openwrt.conf` into its `/etc/config/network`
