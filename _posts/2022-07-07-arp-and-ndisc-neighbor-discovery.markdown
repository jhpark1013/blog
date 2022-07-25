---
title: Linux kernel patches - new feature in ARP & NDISC neighbor discovery - part 1 (of 3)
date: 2022-07-07T11:45:46-04:00
---

We're already at the half way point of the Linux kernel internship! I've been meaning to log daily. Although, this plan has derailed quite a bit, I'll try to fill you in on what I've been up to!

I submitted a [patchset](https://lore.kernel.org/netdev/cover.1657755188.git.jhpark1013@gmail.com/) that adds a feature to the ARP (address resolution protocol) and NDISC (neighbor discovery) process. I've learned a bunch of new things which I'll try to document here.

Before jumping into the details, I want to thank my Outreachy mentors: Roopa Prabhu, Stefano Brivio, and Andy Roulin. They've helped me understand the neighbor discovery process in the Linux kernel networking stack and provided invaluable support & feedback in creating these patches!

The patchset I've been working on adds a new feature to the Linux kernel networking stack. The patches were sent to the [net-next](https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net-next.git/) tree of the Linux kernel tree, which is where new networking features are introduced. The goal is to provide a subnet filtering option for ARP and NDISC during neighbor discovery. ARP and NDISC (in ipv4 and ipv6 respectively) are communication protocols that discovers neighbors and updates their tables/caches.

The neighbor discovery process for both ipv4 and ipv6 maps layer 3 (network-layer; IP addresses) to layer 2 (link-layer; ethernet MAC). But they differ in someways. I'll explain ARP and NDISC, and talk about their differences in accepting unsolicited na (neighbor advertisements).

**ipv4 vs ipv6 analogies**:

<!-- | ipv4          | ipv6                   |
|:------------: |:----------------------:|
| ARP           | NDISC / ND             |
| gratuious     | unsolicited            |
| garp          | unsolicited NA         |
|               |(neighbor advertisement)| -->

| version   | protocol                           | response                              | sysctl     |
|:---------:|:----------------------------------:|:-------------------------------------:|:----------:|
| ipv4      | ARP  (address resolution protocol) |garp (gratuious arp)                   | arp_accept |
| ipv6      | NDISC / ND (neighbor discovery)    |unsolicited na (neighbor advertisement)| drop_unsolicited_na & accept_untracked_na|

This post got too long so I'm splitting it into 3 parts. This post discusses ipv4 ARP. The next posts will discuss ipv6 NDISC and the Linux kernel selftests. Here are links to [part 2](/blog/2022/07/24/linux-kernel-patches-new-feature-in-arp-and-ndisc-neighbor-discovery-part-2-of-3.html) and [part 3](/blog/2022/07/24/linux-kernel-patches-new-feature-in-arp-and-ndisc-neighbor-discovery-part-3-of-3.html).

## ipv4: ARP
The ARP protocol is used to map IP network addresses to the hardware addresses (MAC addresses) used by a data link protocol. I explain a bit more about ARP [here](https://jhpark1013.github.io/blog/2022/06/24/days-10-to-18-arp-neighbor-discovery.html).

ARP is a [request-response](https://en.wikipedia.org/wiki/Request%E2%80%93response) protocol. Here’s what a request-for-MAC-address exchange looks like, in Wireshark:
![network](/blog/assets/network_diagrams/wireshark_arp.png)

The communication goes like:
1. Source Device (Computer): "Who has IP address 192.168.0.1?"
2. Target Device (Router): "It's me! My MAC address is ...dc:51:42."
3. Target Device (Router): "Who has IP address 192.168.0.31? What's your MAC?"
4. Source Device (computer): "My MAC address is ...3a:4e:1d."
(192.168.0.10 is my internal server IP address of my router)

ARP can also be an announcement. When a device announces itself without being prompted by an ARP request, it's sending out a called **garp** (gratuitous ARP). It's also called unsolicited advertisements in ipv6.

By default, devices are configured to drop all gratuitous ARP (garp) frames. But you can enable the device to accept garp with something called the `arp_accept` [sysctl](https://elixir.bootlin.com/linux/latest/source/Documentation/networking/ip-sysctl.rst#L1630). When it's enabled, your ARP table will create new entries.

### 1.1. Garp
**Sender of garp**

When would you send garp?
- Case A: your IP address changed
- Case B: you've joined a network for the first time
- Case C: redundancy

case A: If you've changed your IP address, the first thing you need to do after changing is to broadcast (ffff.ffff.ffff) your IP address and MAC. Your garp would contain your own information in both source and destination MAC addresses (`sip == tip`).

case B: When you join a network for the first time, you'd send a garp to preemptively update their ARP table without requiring them to initiate the traditional ARP request-response protocol.

case C: If two routers in a network shares an IP address, but each have their own MAC address, garp ensures continued ability to communicate with the IP addresses if one router fails. The IP would now be served by the fail-safe router and this is announced to all the hosts with garp. The hosts would update their ARP table so that the IP maps to that fail-safe router's MAC address.

**Receiver of garp**

Learning from garp used to be binary: learn or not learn. But with [this](https://lore.kernel.org/netdev/93cfe14597ec1205f61366b9902876287465f1cd.1657755189.git.jhpark1013@gmail.com/) new patch, we now have the third option the update the table only if the sender of garp is in the same subnet as the receiver of garp.
More specifically, if the garp source IP is in the same subnet as an address configured on your interface.

### 1.2. New arp_accept knob
The subnet filtering option has been added as the third "knob" to the `arp_accept` sysctl.

`arp_accept` is a sysctl but also a funcion in `arp_process()` that takes the garp source IP address as input and compares it with an IP address on the interface that received the garp message. If the source IP matches an address on the interface, `arp_accept` [here](https://elixir.bootlin.com/linux/latest/source/net/ipv4/arp.c#L875) would be set to 1.

If `arp_accept` is set to 1, `__neigh_lookup()` is called in this [if condition](https://elixir.bootlin.com/linux/latest/source/net/ipv4/arp.c#L888). And when `__neigh_lookup()` is called with its last input `int creat` set to 1, it will create a neigh entry in the ARP table if it doesn't already exist.

In [part 2](/blog/2022/07/24/linux-kernel-patches-new-feature-in-arp-and-ndisc-neighbor-discovery-part-2-of-3.html), I'll talk about how subnet filtering is defined in NDISC!


## References
- [gratuitous arp](https://www.practicalnetworking.net/series/arp/gratuitous-arp/)
