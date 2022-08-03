---
title: Linux kernel patches - new feature in ARP & NDISC neighbor discovery - part 2 (of 3)
date: 2022-07-24T16:45:17-04:00
---

In the previous blog (part 1), I talked about the new knob in `arp_accept` that creates neighbors from garp only if the source ip is in the same subnet as an address configured on the interface receiving the garp message. In this blog, I will explain the new subnet-filtering feature we introduced in NDISC in [this patch](https://lore.kernel.org/netdev/56d57be31141c12e9034cfa7570f2012528ca884.1657755189.git.jhpark1013@gmail.com/). And in the next blog (part 3), I will go through the Linux kernel selftest that was used to test the new features.

Here are links to [part 1](/blog/2022/07/07/arp-and-ndisc-neighbor-discovery.html) and [part 3](/blog/2022/07/24/linux-kernel-patches-new-feature-in-arp-and-ndisc-neighbor-discovery-part-3-of-3.html).

**ipv4 vs ipv6 analogies**:

| version   | protocol                           | response                              | sysctl     |
|:---------:|:----------------------------------:|:-------------------------------------:|:----------:|
| ipv4      | ARP  (address resolution protocol) |garp (gratuious arp)                   | arp_accept |
| ipv6      | NDISC / ND (neighbor discovery)    |unsolicited na (neighbor advertisement)| drop_unsolicited_na & accept_untracked_na|


## ipv6: NDISC
(Neighbor discovery is called NDISC or ND).

ipv4 ARP relies on layer 2 broadcast: A host learns the IP address of the device it wants to send messages to via DNS, but doesn't know the link-layer address yet. It needs to broadcast an ARP request frame with the target IP address as the destination. Finally, that target device responds back with the MAC address (it fills the destination MAC address section of the frame).

The downside to this is that every host must process broadcast frames, even that frame isn't for that host. So a lot of compute resources are wasted by processing ARP requests.

ipv6 neighbor discovery improves on ipv4. Instead of broadcasting the destination address, the node **multicasts** a neighbor solicitation message. The solicited node multicast address is derived from the layer 3 unicast address on the target host. And only the host listening to the multicast group address will respond with the MAC address.

Both ARP tables and NDISC caches have ways of keeping their entries up-to-date when networks change. (i.e. when there are devices going offline and new devices coming online).

Entries periodically go through garbage collection to flush the tables. The garbage collection times are determined by the neigh/default/gc_thresh sysctl.

And there's also a system to keep the neighbor entries up-to-date. Entries will always go through following state transitions, INCOMPLETE-->STALE->DELAY->PROBE->REACH (RFC 2461)

1. incomplete: router has sent message and address resolution is in progress (MAC address not yet received).
2. stale: The stale state signifies that address resolution is needed. Stale entries are kept in the ND cache for 4 hours before being removed. Packets destined for a
neighbor in the stale state will still be forwarded by the router. Once
forwarded the neighbor unreachability detection (NUD) mechanism begins,
and the neighbor reachability state transitions to the delay state.
3. delay: router is waiting for packet response.
4. probe: If there is no response after 5 seconds, the neighbor reachability state
changes to probe and three NS messages are sent for address resolution.
If a response is received from the neighbor the state changes to
reachable again. If there is no response to the three NS requests, the
neighbor entry is deleted.
5. reachable: neighbor's responses are received within the "reachable time."

### 2.1. Unsolicited NA
<!-- Unsolicited NA is the garp for ipv6. -->

Prior to the patch that introduced `accept_untracked_na`, NDISC didn't create neighbor entries from unsolicited neighbor advertisements.

The sysctl options for accepting gratuitous/unsolicited na for ipv4 and ipv6:
- ipv4: `arp_accept`
- ipv6: `drop_unsolicited_na` and `accept_untracked_na`

Side note: I thought about combining `drop_unsolicited_na` and `accept_untracked_na` into one sysctl. I thought combining knobs that are complements of each other eliminates redundancies of turning one knob off to turn on another. But this is considered an API change which is highly discouraged. `drop_unsolicited_na` has been around for a long time, so we cannot change that. This is because once a sysctl is out, people start to depend on it and removing it will cause their systems to not work.

Also, `drop_unsolicited_na` and `accept_untracked_na` are not actually complete complements of each other. `accept_untracked_na` encompasses are wider range (unsolicited na **and** solicited-but-not-in-cache). This is explained by the RFC 9131 (details below).


### 2.2. Accepting unsolicited NA: RFC 9131
The functions where neighbors are created (i.e. `arp_process()` and `ndisc_recv_na()` functions) operate differently.

<!-- In the `ndisc_recv` function in ndisc.c, the default behavior is to **not** drop unsolicited neighbor advertisements. However, in the `arp_process` function in arp.c, the default behavior is to drop gratuitous arp. So the default behaviors are different. -->

<!-- But even when the default (in NDISC) is to accept unsolicited, -->

For ARP there was already an [if condition](https://elixir.bootlin.com/linux/latest/source/net/ipv4/arp.c#L875) in `arp_process()` that accepts unsolicited advertisements (controlled by the `arp_accept` sysctl).

In NDISC's `ndisc_recv_na()` function, if all the if..return conditions passes, then in the end it calls `ndisc_update()` which calls `neigh_update()`, which creates a new entry in the neighbor cache.

Among all the if...return conditions, there is only an if condition that blocks unsolicited if `drop_unsolicited_na` was on (which was always off). And there was an [if condition](https://elixir.bootlin.com/linux/latest/source/net/ipv6/ndisc.c#L1044) that only allows cache updates in very specific cases (which excluded any cases where the address wasn't already in the cache).

ndisc.c code with comments:
```c
static void ndisc_recv_na(struct sk_buff *skb)
{
        if (/* condition */) {
                /* don't accept */
                return;
        }

        ...
        /* bunch of if...return conditions  */
        ...

        /* Include code to accept untracked_na here */

        /* If we make it to here, and neighbor exists,
         * update cache
         */
        neigh = neigh_lookup(&nd_tbl, &msg->target, dev);
        if(neigh) {
                ndisc_update(...);
        }
}
```

Recently there has been a new change. An RFC came out in October 2021 that adds a behavior to accept unsolicited NA.
RFC 9131 (section 4.2) says:

> "routers create a new neighbor cache entry upon receiving an
unsolicited neighbor advertisement for an address that does
not already have a neighbor cache entry. These changes do not
modify the router behavior specified in RFC4861 for the
scenario when the corresponding neighbor cache entry already
exists"


So Arun's [patch](https://lore.kernel.org/lkml/642672cb-8b11-c78f-8975-f287ece9e89e@gmail.com/T/), following RFC 9131, created another if condition that creates a `neigh` for addresses that are not in the cache (unsolicited or solicited), right before the if condition that actually creates the cache with `ndisc_update()`, upon checking that a `neigh` exists.


Prior to this recent patch, there was no condition in the `ndisc_recv_na()` function that creates new entries in the neighbor cache from unsolicited or untracked neighbor advertisements.

So when I add a new knob '2' for `accept_untracked_na`, it's subnet filtering from both unsolicited and absent-in-cache solicited. [Here's](https://lore.kernel.org/netdev/56d57be31141c12e9034cfa7570f2012528ca884.1657755189.git.jhpark1013@gmail.com/) my patch.

<!-- I think it's best to follow the RFC's way of accepting untracked (unsolicited + solicited-but-absent-in-cache) when we create the `neigh`. -->

For the case that benefits from the `accept_untracked_na` described in
RFC 9131, is in this scenario:
1. Host joins network and receives router advertisement.
2. Host populates its neighbor cache with the default router ipv6 and link-layer addresses (lladdr) and can send traffic to off-link dst.
3. But the router doesn't have any cache entries for the host yet! The router has to complete address resolution. and all the packets that arrive before the resolution are dropped.

So it's beneficial to have a usable cache entry for the host address by the time the router receives the first packet for that address. So `accept_untracked_na` will create that usable cache entry in STALE
state.

In [part 3](/blog/2022/07/24/linux-kernel-patches-new-feature-in-arp-and-ndisc-neighbor-discovery-part-3-of-3.html), I'll go through the Linux kernel selftest that was used to test these new features!

## References
- [ipv6 neighbor discovery](https://blogs.infoblox.com/ipv6-coe/ipv6-neighbor-discovery-cache-part-1-of-2/)
