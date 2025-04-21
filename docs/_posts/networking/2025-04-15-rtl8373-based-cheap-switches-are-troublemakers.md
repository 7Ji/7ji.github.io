---
layout: post
title:  "RTL8373-based cheap 2.5 Gbps switches are troublemakers"
date:   2025-04-16 10:00:00 +0800
categories: networking
---

RTL8373 is great: it's the cheapest L2 network switching IC that provides 8x 2.5 Gbps ethernet ports (4 with its built-in PHY, 4 with external PHYs from RTL8224), 1 10 Gbps SFP port, and on top of them fancy L2 management features like VLAN, Link aggregation, etc.

From around March 2024 I've bought 5 of these switches, three of them are from Sirivision, and the remaining two from Hellotek. The price kept dropping as I bought more of them, going from around 290 CNY in March 2024 to around 210 CNY in March 2025. It's almost a steal!

These are all located at different places: two of which were installed at my new house at my hometown to provide in-home 2.5 Gbps networking, two of which were installed at my rent house at the city I work to also provide in-home 2.5 Gbps networking in addition to L3 10 Gbps core switching, and the last one installed in my office for 2.5 Gbps switching among my wired devices.

As they came from some almost unknown branch I didn't expect much from their stock firmware, but the Sirivision switches tend out to be very feature-full: features go from VLAN tagging, LAG, IGMP snoofing to DHCP spoofing protection, almost like they've cut nothing from what Realtek left in their reference BSP and exposed everything, their web UI looks very simple and I appreciate that; on the other hand the Hellotek ones provide only limited features, with only base VLAN tagging, and LAG with limitation (one group must be on 4 native ports and the other group must be on 4 "external" ports), they got no IGMP snoofing, no DHCP spoofing protection, while they have nice web UI it just adds the shame that they choose to limit the features exposed.

They both seem nice and sound, considering their price, providing cheap high speed switching with supposedly little to no hassle. However, in reality they're really troublemakers, and I'll summarize the problems below:

1. Configuration is applied only after booting (only affecting Sirivision)

    The VLAN, LAG, etc settings are not applied directly after booting. There's a short time window (1-2 second) during which the switch acts as a dumb switch.
    
    If you use the switch just as a simple dumb switch without VLAN, LAG, etc then you are safe. However if you have untagged VLAN your untagged traffic would go rogue and escape the VLAN filtering; if you have LAG the LAG ports would forward traffic to each other resulting in loops:

    ```
    Apr 05 23:57:18 wn1 kernel: igc 0000:02:00.0 enp2s0: NIC Link is Down
    Apr 05 23:57:18 wn1 kernel: igc 0000:03:00.0 enp3s0: NIC Link is Down
    Apr 05 23:57:19 wn1 kernel: bond0: (slave enp2s0): link status definitely down, disabling slave
    Apr 05 23:57:19 wn1 kernel: bond0: (slave enp3s0): link status definitely down, disabling slave
    Apr 05 23:57:19 wn1 kernel: bond0: now running without any active interface!
    Apr 05 23:57:19 wn1 kernel: bridge0: port 1(bond0) entered disabled state
    Apr 05 23:57:40 wn1 kernel: igc 0000:03:00.0 enp3s0: NIC Link is Up 2500 Mbps Full Duplex, Flow Control: RX/TX
    Apr 05 23:57:40 wn1 kernel: igc 0000:02:00.0 enp2s0: NIC Link is Up 2500 Mbps Full Duplex, Flow Control: RX/TX
    Apr 05 23:57:40 wn1 kernel: bond0: (slave enp2s0): link status definitely up, 2500 Mbps full duplex
    Apr 05 23:57:40 wn1 kernel: bond0: (slave enp3s0): link status definitely up, 2500 Mbps full duplex
    Apr 05 23:57:40 wn1 kernel: bond0: active interface up!
    Apr 05 23:57:40 wn1 kernel: bridge0: port 1(bond0) entered blocking state
    Apr 05 23:57:40 wn1 kernel: bridge0: port 1(bond0) entered forwarding state
    Apr 05 23:57:40 wn1 kernel: bridge0: received packet on bond0 with own address as source address (addr:f6:fe:da:c5:1d:90, vlan:0)
    Apr 05 23:57:40 wn1 kernel: bridge0: received packet on bond0 with own address as source address (addr:f6:fe:da:c5:1d:90, vlan:0)
    Apr 05 23:57:40 wn1 kernel: bridge0: received packet on bond0 with own address as source address (addr:f6:fe:da:c5:1d:90, vlan:0)
    Apr 05 23:57:41 wn1 kernel: igc 0000:02:00.0 enp2s0: NIC Link is Down
    Apr 05 23:57:41 wn1 kernel: igc 0000:03:00.0 enp3s0: NIC Link is Down
    Apr 05 23:57:41 wn1 kernel: bond0: (slave enp2s0): link status definitely down, disabling slave
    Apr 05 23:57:41 wn1 kernel: bond0: (slave enp3s0): link status definitely down, disabling slave
    Apr 05 23:57:41 wn1 kernel: bond0: now running without any active interface!
    Apr 05 23:57:41 wn1 kernel: bridge0: port 1(bond0) entered disabled state
    Apr 05 23:57:44 wn1 kernel: igc 0000:02:00.0 enp2s0: NIC Link is Up 2500 Mbps Full Duplex, Flow Control: RX/TX
    Apr 05 23:57:44 wn1 kernel: igc 0000:03:00.0 enp3s0: NIC Link is Up 2500 Mbps Full Duplex, Flow Control: RX/TX
    Apr 05 23:57:45 wn1 kernel: bond0: (slave enp2s0): link status definitely up, 2500 Mbps full duplex
    Apr 05 23:57:45 wn1 kernel: bond0: (slave enp3s0): link status definitely up, 2500 Mbps full duplex
    Apr 05 23:57:45 wn1 kernel: bond0: active interface up!
    Apr 05 23:57:45 wn1 kernel: bridge0: port 1(bond0) entered blocking state
    Apr 05 23:57:45 wn1 kernel: bridge0: port 1(bond0) entered forwarding state
    ```

    These are bad things and you definitely would not like them in your network.

    Luckily things are not too bad as you would not reboot the switch very frequently. And the issue seems only affecting Sirivision and not Hellotek.

2. VLAN 1 cannot be safely deleted

    On Hellotek VLAN 1 cannot be deleted at all: the button is grey; on Sirivion you seem to be able to delete it, but deleting it results in a freezing interface with glitched characters as an indication of memory leak.

    Both of these seem to be using VLAN 1 for their internal logic, and while it seems still available for modification you'd better left them in place, just removing ports from it if you want some modification.

3. Management VLAN cannot be set

    The Sirivision ones do not provide the option to modify the management VLAN, while the Helloteks ones do provide the option but it never take effect. The management VLAN can only be 1, but it's not 1 actually, there's just no "management VLAN" in fact, which comes to the next issue.

    And your know what? It seems Hellotek "has" management VLAN but that only works for external traffic! I.e. when through a stacked switch, you now can only access another switch from untagged traffic, how secure and convenient!

4. Management packets are captured from all VLANs

    The switch captures any traffic targetting its IP instead of only from a specific management VLAN. To make things worse, while the Sirivision one captures these traffic from even external traffic forwarded from other switches (i.e. a stacked config), the Hellotek ones only capture such traffic from ports directly connected to it physically.

    This is a bad thing. E.g. let's assume you left the switch to keep its default 192.168.1.199 mangement IP, and you have a VLAN 123 running the 192.168.1.0/24 network, your access to 192.168.1.199 in VLAN 123 from a port connected to the switch leads you always to the switch, not the actual target. 

5. VLAN configuration cannot be saved when under load (only affecting Sirivision)

    If there's already traffic in a VLAN that's not fully configured, you can never get it fully configured. The web UI just freezes and config would never be saved. You can only configure things in one go. 

    This makes it very inconvenient if you want your main VLAN not being VLAN 1, and to make things worse recall that "VLAN 1 cannot be safely deleted" and "Both of these seem to be using VLAN 1 for their internal logic", so in general you'd better not be using VLAN 1 and have to do this. To work around this and make your VLANs seperate from the default VLAN 1 hassle you have to configure ports in batch but not including your currently used port, and then swap ports around and set the remaining port. And sometimes you cannot set the remaining ports and have to delete the VLAN and re-create it.

6. VLAN and port naming has no memory boundary check

    While it appears the VLAN IDs and ports can be named for lookup, setting them sometimes result in success but most time result in glitched characters. And when they're  glitched some other settings might get flushed. This is most likely a memory boundary checking issue and when it happens your only reliable fix is to revert everything, most likely to factory reset.

7. Under specific workload the switch would power cycle (only affecting Hellotek)

    I'm using a single port on one of the Hellotek switch as an upstream trunk port to connect to the ISP modem, carrying both PPPoE Internet upstream in a VLAN and IPoE IPTV upstream in another VLAN, the IPTV VLAN is then carried in trunk through another Hellotek switch, then in trunk through a K2P running Openwrt acting as both switch and AP. Strangely the main switch i.e. the one connected directly to the modem would reset itself when IPTV stream start to run. E.g. if I don't watch TV then everything seems fine, if I start watching TV and the IPTV set-top box initiated its IPoE stream then after a couple of minutes the main switch would reset, cutting both my Internet and IPTV connections. If they keep running for a few minutes after the switch comes back online then it would not reset any more, very very strange.

    To get away from this issue I had to replace the main switch to a 10 Gbps L3 switch and add a 2.5 Gbps base-T SFP+ module.

So while these switches seem cheap, be prepared for various issues if you decide to use them for fancy network setups.
