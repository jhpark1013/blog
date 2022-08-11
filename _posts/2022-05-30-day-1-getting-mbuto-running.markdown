---
title: Getting mbuto running (Linux kernel tools)
date: 2022-05-30T23:19:53-04:00
---

It'd be really great to get [mbuto](https://mbuto.sh/) working because it would make kernel testing a lot quicker.
mbuto is a shell script building initramfs. Instead of booting the image in my qemu then starting the selftests manually, I can now simply run mbuto which loads the image and automatically runs the kernel selftests when the custom kernel boots in qemu.  

The goal was to get this command to work without errors. I would run the command below in the directory where the kernel was built.
```
kvm -kernel arch/x86/boot/bzImage -initrd $(../mbuto/mbuto -p kselftests -C timens) \
-m 4096 -cpu host -nodefaults -nographic -append console=ttyS0 -serial stdio
```

First, I had to install a few packages for mbuto. The full [list](https://mbuto.sh/mbuto/tree/mbuto#n192) of package is in the `PROGS` variable in the `profile_kselftest` function. They are commands, not package names, but they're often the same.

Then I was getting some kernel panic errors initially:
```
end Kernel panic - not syncing: No working init found.  Try passing init= option to kernel.
```

So I tried  `mkinitramfs -o ramdisk.img` to create the root file system because the error indicated there was no init binary to run. Creating a ramdisk was the most straight-forward way. I think mbuto does this for me so this is probably not right.

I'm not sure if this solved the problem or maybe pulling new code changes from the mbuto repo solved the error. But it worked!!

```
Press s for shell, any other key to run selftests

TAP version 13
1..7
# selftests: /timens: timens
# 1..10
# ok 1 Passed for CLOCK_BOOTTIME (syscall)
# ok 2 Passed for CLOCK_BOOTTIME (vdso)
# ok 3 Passed for CLOCK_BOOTTIME_ALARM (syscall)
# ok 4 Passed for CLOCK_BOOTTIME_ALARM (vdso)
# ok 5 Passed for CLOCK_MONOTONIC (syscall)
# ok 6 Passed for CLOCK_MONOTONIC (vdso)
# ok 7 Passed for CLOCK_MONOTONIC_COARSE (syscall)
# ok 8 Passed for CLOCK_MONOTONIC_COARSE (vdso)
# ok 9 Passed for CLOCK_MONOTONIC_RAW (syscall)
# ok 10 Passed for CLOCK_MONOTONIC_RAW (vdso)
# # Totals: pass:10 fail:0 xfail:0 xpass:0 skip:0 error:0
ok 1 selftests: /timens: timens
# selftests: /timens: timerfd
# 1..3
# ok 1 clockid=7
# ok 2 clockid=1
# ok 3 clockid=9
# # Totals: pass:3 fail:0 xfail:0 xpass:0 skip:0 error:0
# 1..3
ok 2 selftests: /timens: timerfd
# selftests: /timens: timer
# 1..3
# ok 1 clockid=7
# ok 2 clockid=1
# ok 3 clockid=9
# # Totals: pass:3 fail:0 xfail:0 xpass:0 skip:0 error:0
# 1..3
ok 3 selftests: /timens: timer
# selftests: /timens: clock_nanosleep
# 1..4
# ok 1 clockid: 1 abs:0
# ok 2 clockid: 1 abs:1
# ok 3 clockid: 9 abs:0
# ok 4 clockid: 9 abs:1
# # Totals: pass:4 fail:0 xfail:0 xpass:0 skip:0 error:0
ok 4 selftests: /timens: clock_nanosleep
# selftests: /timens: procfs
# 1..2
# ok 1 Passed for /proc/uptime
# ok 2 Passed for /proc/stat btime
# # Totals: pass:2 fail:0 xfail:0 xpass:0 skip:0 error:0
ok 5 selftests: /timens: procfs
# selftests: /timens: exec
# 1..1
# ok 1 exec
# # Totals: pass:1 fail:0 xfail:0 xpass:0 skip:0 error:0
ok 6 selftests: /timens: exec
# selftests: /timens: futex
# 1..2
# ok 1 futex with the 0 clockid
# ok 2 futex with the 1 clockid
# # Totals: pass:2 fail:0 xfail:0 xpass:0 skip:0 error:0
# 1..2
ok 7 selftests: /timens: futex
All tests passed, shutting down guest...
[   77.942829] sysrq: Power Off
[   77.944800] reboot: Power down

```

The selftests (which are filtered by anything to do with `timens` supplied by the `-C` flag in the mbuto invocation above) would run after booting the kernel in my qemu virtual machine. This filtering capability that Sevinj made is pretty neat because you can focus on testing just the relevant set of tests that I'm interested in. I think it'll help streamline my workflow for testing the Linux kernel selftests.
