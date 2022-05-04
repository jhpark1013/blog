---
title: Building custom Linux kernel and kernel modules
date: 2022-04-30T12:20:03-04:00
---

These are some notes I took while building my own custom Linux kernel and kernel modules. I tried to make the notes more generalizable but it’s mainly a way to summarize my build process and note some confusions I had (and a partial work around). The notes are divided into three sections:

1. Building the Linux kernel
2. Building the kernel modules
3. Initial steps on testing it on the hardware

## Section 1: Building a custom Linux kernel
You can build on a local machine or the target machine. You can choose the version that your modules were developed on and you can also apply additional patches that had not been applied yet to the version of the kernel you are working on. For example, if your kernel version is 5.4 and there are patches that have been applied to 5.5 that did not make it in time for the 5.4 release, you can `git apply patches` before compiling.

Next, you need a kernel config file (`.config` file).
You can copy the kernel config from the existing system to the kernel tree:
```
cp /boot/config-$(uname -r) .config
```

A bare minimum config takes less time to compile. This might be worthwhile to make.

To compile, run make:
```
make -jx
```

Where x is the number of processors (found with `nproc`).

```
make menuconfig
```
gives a text-based interface to select options. You can select 32 vs 64 bit, power management, etc. `make oldconfig` updates the old `.config` file with the new configs. It will ask you for new options which you can select to add to the config.

Before `make`, edit the `.config` file to include your module
```
CONFIG_MY_MODULE=y
```
A kernel module is a piece of code that can be loaded or unloaded into the kernel. The next section delves a little more into building the modules into your kernel.


## Section 2: Building an out-of-tree kernel module
If you have a custom kernel module you’d like to compile into your kernel, it’s a little more work. This involves configuring the kbuild (the build system used by the Linux kernel). To build the external module, first have a prebuilt kernel with the modules enabled following the above section. To understand the build process, let’s go over the build system and its various makefiles.


The makefile has several parts: (1) the top Makefile, (2) the Makefile to provide common rules for all kbuild Makefiles, and the (3) kbuild Makefiles that exist in all subdirectories containing individual drivers.

1. The top Makefile builds the two major products: vmlinux (the resident kernel image) and modules. It builds by recursively descending into the subdirectories of the kernel source tree (ones that are selected in the .config file).

2. The scripts/Makefile contains definitions and rules to build the kernel based on the kbuild Makefiles.

3. Each subdirectory has a kbuild Makefiles that constructs files used by kbuild to build built-in or modular targets.

To compile your own custom module, you’ll need to make the kbuild Makefile. You’ll see lines like
```
obj=y += foo.o
```
Which tells kbuild that there is one object in that directory named `foo.o` and this will be built from `foo.c` or `foo.S`. If `foo.o` is built as a module, a variable `obj-m` is used.
```
obj-$(CONFIG_FOO) += foo.o
```

I was a bit confused here because it’s a bit of a chicken and egg problem; I haven’t built the driver yet so it doesn’t show up in the config file -- so how do I insert it into the config file if I haven’t created the kbuild file? It is in the process of creating the kbuild that the trigger is set; when the first makefile
`$(MAKE) -C $(KDIR) M=$$PWD`
is called, this invokes the kbuild makefile that calls
`$(MY_MODULE)+=my_module.o`
Which creates a kbuild file for our external module The error triggers here if `CONFIG_MY_MOUDLE` isn’t set in the config file. This brings me back to the question I posed before: how do I insert it this custom module in the config file if I haven’t created the kbuild file yet? Simply placing it in the `.config` file doesn’t work because in the build process that manually inserted line gets deleted. Adding `MY_MODULE` stanza in the kconfig file doesn’t work either. We need to add it to the kbuild file itself.

The work around: Instead of putting the `MY_MODULE` stanza in kconfig, I defined that option in the `my_module.h` files. `my_module.o` does not need to be in the Makefiles. So I commented out
```
core-$(CONFIG_MY_MODULE)+=my_module.o
```
in the Makefile. Defining `CONFIG_MY_MODULE` in `core.h` ensures that from the point of that code, the option is always enabled. (Note: This is not a solution and I’m working on resolving the core issue).

If your custom module is reliant on other drivers in the kernel, you need to build the kernel with the relevant drivers compiled in before your driver can be compiled.
It compiles in alphabetical order so, for example, when “a” is done, that means all the “ath” modules are compiled.

After compilation, go to where your module is defined and run
```
Make -C KERNEL_SRC=$(HOME)/build/linux C=1 M=$(PWD)
```
`$HOME/build/linux` is where my Linux kernel code is. Repace that with where you’re kernel source code is.

If your driver is successfully compiled, the relevant `.ko` files should be created.


## Section 3: Testing on hardware
Next, make the deb-pkg to produce the `.dev` files and `scp` the files over to your test hardware.
```
make deb-pkg
```
Your hardware should already have an operating system. You can install your custom kernel with the file that was scp’d over.  Install the kernel package and modules (you can do it over `picocom` or `ssh`, as you wish).

You can find the correct grub-reboot invocation. Or you can reboot the hardware and catch the boot menu. Make sure to boot exactly the version you just built. After boot, try `insmod` inserting into your kernel to see if your module works.
```
insmod ~/your_module.ko
dmesg | tail
```

This was a brief overview of the Linux kernel's elaborate build system. There is more to be explored -- specifically, an example Makefile would be useful. I'll try to bring in some specific kernel module examples in future blog posts.

## References
[1] [kerel documentation on kbuild](https://www.kernel.org/doc/html/latest/kbuild/modules.html)
[2] [oreilly on linux device drivers](https://www.oreilly.com/library/view/linux-device-drivers/0596005903/ch02.html)
[3] [stack overflow on "adding new driver code"](https://stackoverflow.com/questions/11710022/adding-new-driver-code-to-linux-source-code)
