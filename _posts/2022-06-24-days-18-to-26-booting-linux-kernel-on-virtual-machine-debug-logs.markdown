---
title: Days 18 to 26 - booting custom Linux kernel on virtual machine (debug logs)
date: 2022-06-24T14:42:14-04:00
---

I've created a selftest for testing the new arp_accept test. I am currently trying to boot the new kernel (with the new arp_accept knob) on my qemu debian but am a little stuck here. The qemu won't load a stable kernel build either. I'm running debian on my qemu.

I've tried 4 different ways of testing on virtual machine:
1. booting on qemu with mbuto creating my initrd: mbuto -p kselftests -C net, mbuto -p kselftests -T net:arp_accept_test.sh -- these didn't work.
mbuto -p kselftest -C timens works though

2. booting the kernel from qemu: I created a bare bones ram disk image that can be used boot into a busybox shell. It's stuck on "clocksource: Switched to clocksource tsc."

3. create a deb-pkg -- this failed to make .deb files (I was planning to scp it over)

4. haven't tried building it on the vm because the storage. (I had run out of storage on my image when I tried building it) I've increased the image but it wasn't enough. So I need to try increasing it more and build it again (haven't had the chance to this yet)

Note: I've gotten method 4 to work now. Methods 1-2 are still unsolved. This post is mostly to document my WIP debug process for method 1.  

Ideally I'd like to work with mbuto or boot from qemu. Like, I'd like this command to work
```
qemu-system-x86_64 \
- kernel git/kernels/net-next/arch/x86_64/boot/bzImage \
- initrd git/kernels/kernel-utils/initramfs.cpio.gz  \
- nographic -append "console=ttyS0"
```

I've been focused on debugging the first method involving mbuto which is a tool for testing custom kernels.  

Status right now with mbuto (for kernel v5.18-rc4):
- [x] all missing modules have been enabled in configs
- [x] kernel boots without kernel panics and is able to find rootfs (indicates that initrd is working)
- [x] mbuto runs without missing module errors
- [x] mbuto `make install` succeeds
- [x] mbuto creates an image
- [ ] kernel boots without errors (modprobe errors)
- [ ] kernel selftests runs without errors

For the most recent kernels, v5.18-rc5 onwards, the `bpf_helper_defs.h` not found error exists. More detail on this in the logs (6/22).

Side note: I've tried 2 different ways for initrd (mbuto does this for me but when I was working on debugging the second method I was working on creating my own initrd)
1. mkinitramfs -o ramdisk.img
2. using busybox following these two references [ref 1](https://ops.tips/notes/booting-linux-on-qemu/), [ref 2](https://www.youtube.com/watch?v=asnXWOUKhTA)
I get the errors noted in the logs on day 6/21.

## Logs

**6/16** - Installed debian qemu again.

**6/17** - Made a shared folder in qemu to run selftests qemu. The qemu ran out of space when cloning the net-next tree and building it.

**6/18** - Virtual machine runs into issues finding initrd when booting. Created initramfs using busybox.

**6/20** - mbuto debug: debug using `make .. install` and reading full bash output using `-x` debug mode.
```
/usr/bin/make SKIP_TARGETS=alsa arm64 bpf breakpoints capabilities cgroup clone3 core cpufreq cpu-hotplug drivers/dma-buf efivarfs exec filesystems filesystems/binderfs filesystems/epoll firmware fpu ftrace futex gpio intel_pstate ipc ir kcmp kexec kvm landlock lib livepatch lkdtm membarrier memfd memory-hotplug mincore mount mount_setattr move_mount_set_group mqueue nci net/af_unix net/forwarding net/mptcp netfilter nsfs pidfd pid_namespace powerpc proc pstore ptrace openat2 rlimits rseq rtc seccomp sgx sigaltstack size sparc64 splice static_keys sync syscall_user_dispatch sysctl tc-testing timens timers tmpfs tpm2 user vDSO vm x86 zram cpu-hotplug memory-hotplug -C ./tools/testing/selftests/ install
```

**6/21** - All the kernel panics: kernel can't find rootfs and initrd.
I got a kernel panic error when it couldn't find the init.
The "No init found" error is output when the kernel can't find `/sbin/init`, `/etc/init`, or `/bin/init` on the rootfs.
The "Failed to execute /init" error is output when the kernel can't find `/init` in the `initramfs`, whether built-in or external.
This shouldn't happen since, upon startup, I'm creating an init file using busybox and mounting proc, sys, and dev..
```
[    0.556637] x86/mm: Checked W+X mappings: passed, no W+X pages found.
[    0.557244] Run /init as init process
[    0.557645] Failed to execute /init (error -2)
[    0.558066] Run /sbin/init as init process
[    0.558444] Run /etc/init as init process
[    0.558842] Run /bin/init as init process
[    0.559233] Run /bin/sh as init process
[    0.559614] Kernel panic - not syncing: No working init found.  Try passing init= option to kernel.
```
I also got a kernel panic error "vfs" not finding the rootfs and asking me to supply `root=`
```
[    2.130276] Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
[    2.130618] CPU: 0 PID: 1 Comm: swapper/0 Not tainted 5.18.0-rc3 #10
[    2.130846] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS 1.13.0-1ubuntu1.1 04/01/204
[    2.131182] Call Trace:
[    2.131635]  <TASK>
[    2.131803]  dump_stack_lvl+0x49/0x5f
[    2.132272]  dump_stack+0x10/0x12
[    2.132358]  panic+0x10c/0x29e
[    2.132477]  mount_block_root+0x14b/0x1f1
[    2.132609]  mount_root+0x147/0x153
[    2.132725]  prepare_namespace+0x13f/0x170
[    2.132838]  kernel_init_freeable+0x25e/0x284
[    2.132965]  ? rest_init+0xe0/0xe0
[    2.133067]  kernel_init+0x1a/0x130
[    2.133180]  ret_from_fork+0x22/0x30
[    2.133327]  </TASK>
[    2.133856] Kernel Offset: 0x12600000 from 0xffffffff81000000 (relocation range: 0xffffffff8000000)
[    2.134347] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0-
```

Reinstalling qemu worked only for initrd=initramfs where initramfs was made with `mkinitramfs`. I deleted the debian install on qemu in the process, which I think was causing problems.. or maybe it was the additional inputs to the kvm like `-cpu host` etc)

I tried a using busybox, and understood the init script in initrd, but -- none of it really mattered because, now, mbuto successfully created my initrd.

Debug configs: I noticed this new path when I enable `CONFIG_DEBUG_INFO_BTF`
```
./tools/bpf/resolve_btfids/libbpf/bpf_helper_defs.h
```
`make mrproper` and try `make`. Still unable to get past the kernel panics.

**6/22** - found a [bug report](https://lore.kernel.org/netdev/20220527085351.GC11731@xsang-OptiPlex-9020/t/) with the same error message I'm getting. Tried using an older version of the kernel. mbuto booted.

The slightly older version of the kernel, v5.18-rc4 works. For v5.18-rc5 onwards, I get the "bpf_helper_defs.h not found" issue -- this, i'm pretty sure, is because of this commit: edae34a3ed9293b5077dddf9e51a3d86c95dc76a ("selftests net: add UDP GRO fraglist + bpf self-tests")  mentioned in this bug report. It's adding more paths in /tools/testing/selftests/net/bpf/Makefile to find the bpf includes. And this change is causing the "not found" issue.
```
clang -O2 -target bpf -c bpf/nat6to4.c -I../../bpf -I../../../lib -I../../../../../usr/include/ -o /home/jaehee/git/kernels/net-next/tools/testing/selftests/net/bpf/nat6to4.o
In file included from bpf/nat6to4.c:43:
../../../lib/bpf/bpf_helpers.h:11:10: fatal error: 'bpf_helper_defs.h' file not found
#include "bpf_helper_defs.h"
^~~~~~~~~~~~~~~~~~~
1 error generated.
```

Since the older version v5.18-rc4 builds and works on mbuto, I'm applying my arp_accept test on this kernel version and running it on mbuto.  
There are two issues I'm running into with mbuto right now: (1) test is not being found, and (2) all the selftests aren't running properly.
The first issue, is solved now. I needed to add `TEST_PROGS += arp_accept_test.sh` to mbuto's makefile.

I think the second issue is related to the the modprobe error (see day 6/23). The interfaces of the namespaces are not showing up. mbuto boots and runs selftests but they are full of errors. I see "cannot find device <interface>" in all the tests that involve interfaces in namespaces.
I'm going to take a wild guess and say the ip commands aren't working properly maybe because I don't have something configured properly?

**6/23** - mbuto may not be working because of missing kernel modules. The modules listed in `__kmods_needed` in mbuto have been installed. The process goes like, enable in menuconfig or directly enable in .config, `make`, run mbuto with `-x` debug mode and search for the missing modules listed after `WARNING: the following modules are: ...`, repeat until I don't see errors anymore. Now I am able to boot, but the "modprobe: FATAL: Module iptable_nat not found in directory /lib/modules/5.18.0-rc4+" is still there. And the net selftests aren't running properly.

```
modprobe: FATAL: Module iptable_nat not found in directory /lib/modules/5.18.0-rc4+
modprobe: FATAL: Module nf_nat not found in directory /lib/modules/5.18.0-rc4+
modprobe: FATAL: Module nf_tables not found in directory /lib/modules/5.18.0-rc4+
modprobe: FATAL: Module nft_nat not found in directory /lib/modules/5.18.0-rc4+
modprobe: FATAL: Module nf_conntrack not found in directory /lib/modules/5.18.0-rc4+
modprobe: FATAL: Module nft_chain_nat not found in directory /lib/modules/5.18.0-rc4+
```


**6/24** - I thought forgot `make modules_install` afterwards. But mbuto does this for me. I still get "modprobe: FATAL: Module iptable_nat not found in directory /lib/modules/5.18.0-rc4+" issue still exists. But there I see iptable_nat in the net-next/.mbuto_mods/lib/modules/5.18.0-rc4+/kernel/net. I'm not sure why it's not detecting it?

My current output: https://0x0.st/oSM0.txt

**6/25** - I am using qemu virtual machine to test (method 4 from the very beginning of this blog). I've compiled net-next from there, `make modules_install`, `make install` and boot (select the new kernel I just built in grub). Then, use `perf` to trace kernel functions that I've edited.

```
qemu-system-x86_64 -enable-kvm -nic user,hostfwd=tcp::2222-:22 -daemonize -m 4G -smp cores=4,cpus=4 net-test

```

## Thoughts

The `timens` selftests work -- i.e. when I run `mbuto -p kselftests -C timens`. So this indicates that there are some dependencies specific to `net` selftests that are not working.

Although it's still unsolved, incremental progress has been made and I'm learning a lot. I'm trying to get better at reading the error messages, finding where in the code it's causing errors, and identifying the source of the problems.
