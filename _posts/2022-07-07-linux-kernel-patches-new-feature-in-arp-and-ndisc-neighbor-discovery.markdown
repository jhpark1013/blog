---
title: Linux kernel patches - new feature in ARP & NDISC neighbor discovery - part 1 (of 2)
date: 2022-07-07T11:45:46-04:00
image: https://jhpark1013.github.io/blog/assets/twitter_cards/arp_code.png
description: In this blog post, I talked about adding the subnet filtering feature in ARP in the Linux kernel networking stack.
---
<!-- /assets/twitter_cards/arp_code.png -->

I recently submitted a [patchset](https://lore.kernel.org/netdev/cover.1657755188.git.jhpark1013@gmail.com/) that adds a feature to the ARP (address resolution protocol) and NDISC (neighbor discovery) process. I've learned a bunch of new things which I'll try to document here.

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

This post got too long so I'm splitting it into two parts. This post discusses ipv4 ARP and a new feature introduced in [patch](https://lore.kernel.org/netdev/93cfe14597ec1205f61366b9902876287465f1cd.1657755189.git.jhpark1013@gmail.com/). In the next post, I will discuss ipv6 NDISC. Here is the link to [part 2](/blog/2022/07/24/linux-kernel-patches-new-feature-in-arp-and-ndisc-neighbor-discovery-part-2.html).

<!-- and [part 3](/blog/2022/07/24/linux-kernel-patches-new-feature-in-arp-and-ndisc-neighbor-discovery-part-3-of-3.html). -->

## ipv4: ARP
The ARP protocol is used to map IP network addresses to the hardware addresses (MAC addresses) used by a data link protocol. I explain a bit more about ARP [here](/blog/2022/06/24/days-10-to-18-arp-neighbor-discovery.html).

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

`arp_accept` is a sysctl but also a funcion in `arp_process()`. Previously, the code directly called `IN_DEV_ARP_ACCEPT` which has now been replaced by the `arp_accept()` function in my patch to include more than just 2 features. Because the 3rd knob has been added, there needed to be a switch statement to output 0 or 1 based on the conditions.

```c
static int arp_accept(struct in_device *in_dev, __be32 sip)
{
	struct net *net = dev_net(in_dev->dev);
	int scope = RT_SCOPE_LINK;

	switch (IN_DEV_ARP_ACCEPT(in_dev)) {
	case 0: /* Don't create new entries from garp */
		return 0;
	case 1: /* Create new entries from garp */
		return 1;
	case 2: /* Create a neighbor in the arp table only if
		 * sip is in the same subnet as an address
		 * configured on the interface that received
                 * the garp message
		 */
		return !!inet_confirm_addr(net, in_dev, sip, 0, scope);
	default:
		return 0;
	}
}
```
For the new case '2', it would take the garp source IP address as input and compare it with an IP address on the interface that received the garp message. If the source IP matches an address on the interface, the `if (arp_accept(in_dev, sip))` in `arp_process()` would be set to 1 which would create a neighbor.

And Stefano pointed out that this would work even when there are multiple addresses configured on the interface because `confirm_addr_indev()`, called by `inet_confirm_addr()` (which I use in `arp_accept()`), matches any subnet configured for the given interface -- see the `for_ifa()` loop. Here's a [link](https://elixir.bootlin.com/linux/latest/C/ident/confirm_addr_indev) to confirm_addr_indev().


```c
if (arp_accept(in_dev, sip)) {
        /* Unsolicited ARP is not accepted by default.
           It is possible, that this option should be enabled
           for some devices (strip is candidate)
         */
        if (!n &&
            (is_garp ||
             (arp->ar_op == htons(ARPOP_REPLY) &&
              (addr_type == RTN_UNICAST ||
               (addr_type < 0 &&
                /* postpone calculation as late as possible */
                inet_addr_type_dev_table(net, dev, sip) ==
                        RTN_UNICAST)))))
                n = __neigh_lookup(&arp_tbl, &sip, dev, 1);
}
```

If `arp_accept(in_dev, sip)` is 1, `__neigh_lookup()` is called. When `__neigh_lookup()` is called with its last input `int creat` set to 1, it will create a neigh entry in the ARP table if it doesn't already exist. (Defined [here](https://elixir.bootlin.com/linux/latest/C/ident/__neigh_lookup)).

```c
static inline struct neighbour *
__neigh_lookup(struct neigh_table *tbl, const void *pkey,
               struct net_device *dev, int creat)
{
	struct neighbour *n = neigh_lookup(tbl, pkey, dev);

	if (n || !creat)
		return n;

	n = neigh_create(tbl, pkey, dev);
	return IS_ERR(n) ? NULL : n;
}
```

In this blog post, I talked about adding the subnet filtering feature in ARP in the Linux kernel networking stack. In [part 2](/blog/2022/07/24/linux-kernel-patches-new-feature-in-arp-and-ndisc-neighbor-discovery-part-2.html), I'll describe how subnet filtering is defined in NDISC!


## References
- [gratuitous arp](https://www.practicalnetworking.net/series/arp/gratuitous-arp/)
