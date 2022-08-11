---
title: perf - how I debugged my Linux kernel patch!
date: 2022-07-26T01:09:47-04:00
---

I recently submitted this [patch](https://lore.kernel.org/netdev/93cfe14597ec1205f61366b9902876287465f1cd.1657755189.git.jhpark1013@gmail.com/) to net-next (I wrote a [blog](/blog/2022/07/07/linux-kernel-patches-new-feature-in-arp-and-ndisc-neighbor-discovery-part-1.html) post about it, too). During the development process, I used perf to debug some unexpected behaviors in the kernel caused by my code!

First, let's check the behavior of the code before applying the patch.

When I test with the kernel that does not have the `arp_accept` patch applied, the `arp_accept=2` test does not pass (as expected, since knob '2' doesn't exist yet before this patch).

And I checked with perf (putting probes on `arp_process` and `neigh_update`) and saw that `neigh_update` was called on `arp_accept=1` but not on `arp_accept=0`.

This is the right behavior. If we're accepting garp, we should see an update in the neighbor cache and a call to `neigh_update`. If we're discarding garp, we should **not** see a call to `neigh_update`.

This is how I used perf:

Create perf probes:
```
sudo perf probe neigh_update
```
Record:
```
sudo perf record -e probe:* -ag -- sleep 5
```
See the record:
```
sudo perf script
```
And you can see the neigh_update being called (and other calls in the network stack):
```
arping 46883 [003] 22325.771054: probe:neigh_update: (ffffffffa47d3d10)
	ffffffffa47d3d11 neigh_update+0x1 ([kernel.kallsyms])
	ffffffffa48a7b70 arp_rcv+0x190 ([kernel.kallsyms])
	ffffffffa47c7fdf __netif_receive_skb_one_core+0x8f ([kernel.kallsyms])
	ffffffffa47c8038 __netif_receive_skb+0x18 ([kernel.kallsyms])
	ffffffffa47c8279 process_backlog+0xa9 ([kernel.kallsyms])
	ffffffffa47c98e1 __napi_poll+0x31 ([kernel.kallsyms])
	ffffffffa47c9dbf net_rx_action+0x23f ([kernel.kallsyms])
	ffffffffa4e000cf __softirqentry_text_start+0xcf ([kernel.kallsyms])
	ffffffffa3eaa666 do_softirq+0x66 ([kernel.kallsyms])
	ffffffffa3eaa6d0 __local_bh_enable_ip+0x50 ([kernel.kallsyms])
	ffffffffa47c6738 __dev_queue_xmit+0x388 ([kernel.kallsyms])
	ffffffffa47c6da0 dev_queue_xmit+0x10 ([kernel.kallsyms])
	ffffffffa4958fdc packet_snd+0x34c ([kernel.kallsyms])
	ffffffffa495ab16 packet_sendmsg+0x26 ([kernel.kallsyms])
	ffffffffa479d6a5 sock_sendmsg+0x65 ([kernel.kallsyms])
	ffffffffa479e933 __sys_sendto+0x113 ([kernel.kallsyms])
	ffffffffa479e9d9 __x64_sys_sendto+0x29 ([kernel.kallsyms])
	ffffffffa49f5f01 do_syscall_64+0x61 ([kernel.kallsyms])
	ffffffffa4c0007c entry_SYSCALL_64_after_hwframe+0x44 ([kernel.kallsyms])
	    7fe4b976568a __libc_sendto+0x1a (/usr/lib/x86_64-linux-gnu/libc-2.31.so)
	 1000608d4dd5341 [unknown] ([unknown])
```

Next, I wanted to test my new patch which was built on the newest kernel (5.18). There aren't any perf packages yet for 5.18. And since perf needs to match the kernel version, there was no way (or so I thought) to validate my patch with perf. But Roopa suggested a clever work around!

What you can do is copy the old kernel perf binary to new kernel binary.

In debian:
```
ls -l /usr/bin/perf
```
to see which file it points to, then copy.
```
cp /usr/bin/perf_5.15 /usr/bin/perf_5.18
```

In ubuntu, it would be the same but for `linux-tools`, which is where perf is:
in `/usr/lib`
```
sudo cp -r linux-tools-5.15 linux-tools-5.18
```
in `/usr/lib/linux-tools`
```
sudo cp -r 5.15.0-40-generic 5.18.0-rc4+
```

So now perf works with the newest kernel and I can finally test my new patch.

..And I'm really glad I did, because I found an error in my selftest! I didn't add the subnet mask `/24` when setting `ip addr`. So the test for `arp_accept=2` was failing; it wasn't adding entries to my table even for the same subnets (but on perf I saw `neigh_update` being called). So I knew it was a problem with my selftest.

After adding the subnet mask, the tests ran well. And as an added precaution, I added another if-else statement in the `arp_accept=2` selftest to cover both cases (non-subnet and in subnet).

Phew, perf saved me a lot of hassle!! And observing perf tracepoints is a pretty good way to learn about kernel internals, too!
