---
title: Where to start in contributing to the Linux kernel
date: 2022-05-19T11:22:51-04:00
---

This is a work-in-progress. I will update this post soon!

## Where does Linux kernel development even happen?
Initially when I started to read the Linux kernel code, I was confused about where the development happened. I had thought it was on GitHub and that the directories like staging are submodules.
It turns out that kernel development does not happen on Github. We see some repositories on GitHub because they are provided as mirrors of the original repositories.

And the staging tree is [here](ttps://git.kernel.org/pub/scm/linux/kernel/git/gregkh/staging.git/). It's not a sub-repository of any sort; it's just another tree. It merges (following pull requests) "between" trees. [Here's](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=dfdc1de64248b5e1024d8188aeaf0e59ec6cecd5) an example.
The staging tree contains the whole kernel, not just staging drivers. But this staging tree is the place where development happens for staging drivers.
Similarly, the netdev development has its own tree called [net-next](https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net-next.git/), so does the wireless development [wireless-next](https://git.kernel.org/pub/scm/linux/kernel/git/wireless/wireless-next.git/) and so on.


## How I started contributing
A good first place to start is the staging directory. [Here](https://git.kernel.org/pub/scm/linux/kernel/git/gregkh/staging.git/log/?h=staging-testing&qt=grep&q=Jaehee) are my patches in staging.
The staging directory contains drivers that are in progress.

## Resources
- ["first kernel patch"](https://kernelnewbies.org/FirstKernelPatch) is a reference that I consistently referred to as I was making patches.
- [Outreachy](https://www.outreachy.org/) is a great program that encourages people to work on open source. The Linux kernel is one of the open source communities that work with Outreachy.

The mentors at Outreachy are immensely knowledgeable and extremely nice! The positive nature of the Linux kernel community, and mentors and contributors at Outreachy, is amazing. It's easy to criticize harshly at novice mistakes, but they chose not to; the constructive feedback was extremely valuable and I learned a lot. And, most people in tech will realize that everyone starts from somewhere. 

## Patch advice
The ["patch philosophy"](https://kernelnewbies.org/PatchPhilosophy) is a good reference.
Before I forget, here's a log of feedback I had received after submitting my initial kernel patches.

- `difforderfile` can sequence the order the files appear in the diff.
- Have a concise subject line. The log message should give a more complete description.
- Instead of repeating what the checkpatch tells you, explain the reasoning behind the fix in your case.

I'll come back to updating this log.
