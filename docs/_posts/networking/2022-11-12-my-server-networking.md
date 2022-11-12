---
layout: post
title:  "Networking configuration on my server"
date:   2022-11-12 14:00:00 +0800
categories: networking
---

## Background
It's been several weeks since I migrated from `NetworkManager` to `systemd-networkd` on my server as a result of seperating the PPPoE-dialing function to a dedicated OpenWrt device (I used to use `ArchLinux` + `nftables` + `dnsmasq` + `NetworkManager` as a manually set up router). The `systemd-networkd` is pretty stable and solid, unlike `NetworkManager` whose Teaming function was broken for a long time [since late last year](https://gitlab.freedesktop.org/NetworkManager/NetworkManager/-/issues/975). I find it maybe necessary to document the configuration if others want to have a solid mixed setup

## Physical

The server is a re-purposed old ITX Desktop PC I used to use as the main driver during my College days. It has the following NICs:
 - 1x Intel i211 1Gb NIC, pre-installed on the MoBo, connected through the B350 chipset with 1x PCI-E 2.0 link speed
   - enp9s0, 1Gb
 - 1x HP 331 FLR with BCM 5719 chip, 4x 1Gb NIC, wired through a M.2 A+E key to PCI-E cable to the wireless card slot, through the B350 chipset with 1x PCI-E 2.0 link speed (capable of 4x PCI-E 2.0)
   - enp8s0f0, 1Gb
   - enp8s0f1, 1Gb
   - enp8s0f2, 1Gb
   - enp8s0f3, 1Gb
 - 1x TAX197 with 4x RTL8125B chip, 4x 2.5Gb NIC, wired through a PCI-E riser to the main PCI-E slot, direct to CPU with 4x PCI-E 2.0 uplink and 4 1x PCI-E 2.0 downlink
   - enp13s0, 2.5Gb
   - enp14s0, 2.5Gb
   - enp16s0, 2.5Gb
   - enp17s0, 2.5Gb

## Wiring
```
#--------#             #---------#
| enp9s0 | ----------- | Netgear |
#--------#             |         |
                       | GS724T  |
#--------#             |     v3  |
|enp8s0f0| ------------|         |
#--------#             | 24*1Gb  |
|enp8s0f1| ------------| switch  |
#--------#             |         |
|enp8s0f2| ------------|         |
#--------#             |         |
|enp8s0f3| ------------|         |
#--------#             #---------#

                         #--------#
#---------#              |  NUC   |
| enp13s0 | ------------ #--------#
#---------#              #--------#
| enp14s0 | ------------ |   PC   |
#---------#              #--------#
| enp16s0 | (not connected)
#---------#              #--------#
| enp17s0 | ------------ | laptop |
#---------#              #--------#
```

## Configuration
 - `enp9s0`, `enp8s0f0`, `enp8s0f1`, `enp8s0f2`, `enp8s0f3` (5x 1Gb NICs) are in the same bonding group (`bond0`) in 802.3ad/LACP mode, the bonding is effectively a trunk port to the switch

    The bonding netdev  
    `00-bond0_to_switch.netdev`
    ```
    [NetDev]
    Name=bond0
    Kind=bond

    [Bond]
    Mode=802.3ad
    TransmitHashPolicy=layer3+4
    LACPTransmitRate=fast
    MIIMonitorSec=1s
    AdSelect=bandwidth
    ```
    The slaves network  
    `10-enp8s0_enp9s0_to_switch_as_bond0_slave.network`
    ```
    [Match]
    Name=enp8s0f* enp9s0
    Type=ether

    [Network]
    Bond=bond0
    ```
 - `bond0`, `enp13s0`, `enp14s0`, `enp16s0`, `enp17s0` (the above bonding and 4x 2.5Gb NICs) are bridged together as `bridge0`. VLAN filtering is enabled, 17 is the default PVID

    The bridging netdev  
    `20-bridge0_all.netdev`
    ```
    [NetDev]
    Description=Bridge for all ports
    Name=bridge0
    Kind=bridge
    MACAddress=6e:92:da:5e:97:4a

    [Bridge]
    DefaultPVID=17
    VLANFiltering=yes
    VLANProtocol=802.1q
    ```
    The above bonding as bridge slave, VLAN 7 and 17 are allowed on it  
    `30-bond0_to_switch_as_bridge0_slave.network`
    ```
    [Match]
    Name=bond0

    [Network]
    Bridge=bridge0

    [BridgeVLAN]
    VLAN=7

    [BridgeVLAN]
    VLAN=17
    ```
    The NIC that connects to Desktop PC as bridge slave, VLAN 7, 17 and 77 are allowed on it   
    `30-enp14s0_to_pc_as_bridge0_slave.network`
    ```
    [Match]
    Name=enp14s0
    Type=ether

    [Network]
    Bridge=bridge0

    [BridgeVLAN]
    VLAN=7

    [BridgeVLAN]
    VLAN=17

    [BridgeVLAN]
    VLAN=77
    ```
    The other 2.5Gb NICs as bridge slave, traffics leaving from and comming to these ports without VLAN tags, are handled as VLAN 7 as it's the Primary VLAN   
    `30-enp13s0_enp16s0_enp17s0_as_bridge0_slave.network`
    ```
    [Match]
    Name=enp13s0 enp16s0 enp17s0
    Type=ether

    [Network]
    Bridge=bridge0

    [BridgeVLAN]
    PVID=7
    EgressUntagged=7
    ```
 - `vlan7`, `vlan17`, `vlan77` are created on the bridge so the server itself can access these VLANs  
    The network file for the bridge, which creates VLANs and does not get its own address. Traffics to and from VLAN 7, 17 and 77 are allowed so the server itself can receive/send on these VLANs  
    ``40-bridge-vlans.network``
    ```
    [Match]
    Name=bridge0
    Type=bridge

    [Network]
    VLAN=vlan7
    VLAN=vlan17
    VLAN=vlan77

    LinkLocalAddressing=no
    LLDP=no
    EmitLLDP=no
    IPv6AcceptRA=no
    IPv6SendRA=no

    [BridgeVLAN]
    VLAN=7

    [BridgeVLAN]
    VLAN=17

    [BridgeVLAN]
    VLAN=77
    ```
    The netdev file for `vlan7`, to declare the VLAN Id 7 to be used when creating the vlan netdev (the name `vlan7` is just for the ease to look up)  
    `50-vlan-7-home.netdev`
    ```
    [NetDev]
    Name=vlan7
    Kind=vlan

    [VLAN]
    Id=7
    ```
    The netdev file for `vlan17`, to declare the VLAN Id 17 to be used when creating the vlan netdev (the name `vlan17` is just for the ease to look up)  
    `50-vlan-17-vm.netdev`
    ```
    [NetDev]
    Name=vlan17
    Kind=vlan

    [VLAN]
    Id=17
    ```
    The netdev file for `vlan77`, to declare the VLAN Id 77 to be used when creating the vlan netdev (the name `vlan77` is just for the ease to look up)  
    `50-vlan-77-vm.netdev`
    ```
    [NetDev]
    Name=vlan77
    Kind=vlan

    [VLAN]
    Id=77
    ```
    The network file for `vlan7` to set a static IPv4 address and possibly accept dynamic IPv6 addresses  
    `50-vlan-7-home.network`
    ```
    [Match]
    Name=vlan7
    Type=vlan

    [Network]
    Address=192.168.7.10/24
    Gateway=192.168.7.1
    ```
    The network file for `vlan17` to set a static IPv4 address and disable IPv6
    `50-vlan-17-vm.network`
    ```
    [Match]
    Name=vlan17
    Type=vlan

    [Network]
    Address=192.168.17.2/24

    LinkLocalAddressing=no
    LLDP=no
    EmitLLDP=no
    IPv6AcceptRA=no
    IPv6SendRA=no
    ```
    The network file for `vlan77` to set a static IPv4 address and disable IPv6
    `50-vlan-77-management.network`
    ```
    [Match]
    Name=vlan77
    Type=vlan

    [Network]
    Address=192.168.77.2/24

    LinkLocalAddressing=no
    LLDP=no
    EmitLLDP=no
    IPv6AcceptRA=no
    IPv6SendRA=no
    ```