---
title: The Linux kernel networking selftests
date: 2022-05-18T13:59:20-04:00
---

I had an opportunity to write a [patch](https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net-next.git/commit/?id=a313f858ed36) to improve the functionality of one of the tests their selftests. I got a lot of help from Roopa Prabhu who was mentoring the Linux kernel Outreachy contributors this year. This patch, although simple, helped me get comfortable with the net-dev tree and the available selftests. It was very interesting to look at the tests and the network topologies defined in them, and I wanted to give a brief overview of the system and write about potential future improvements.

## Selftests and network topologies
The netdev development, which happens in the [net-next](https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net-next.git/) kernel networking tree, has a set of Linux kernel selftests which are growing by the day.

Most kernel trees include the same tests. The net-next tree is simply where new networking features and tests are introduced first. These selftests are located in net-next/tools/testing/selftests/net. Here's the [link](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/tools/testing/selftests/net).  

There is overlap of topologies used across tests. When we setup a network topology, we are setting up namespaces and policy routing (or simply routing) between them.

Every new test copies an existing test and starts from it. So there are groups of tests and their topologies that look similar today. There's a handful of topologies that can serve all tests and it would be helpful to extract the common code from tests into its own libraries and modules.

## Networking concepts
Here's a few networking concepts that's helpful to define to understand the tests.
I am still learning and a lot of this may not be correct.

**Network virtualization** refers to combining hardware and software network resources and divides the network into segments and create software networks between those virtual machines.

**Namespace** is a copy of the network stack from the host system. Network namespaces are useful for setting up containers or virtual environments. Each namespace has its own IP addresses, network interfaces, routing tables and so forth.

So say we want to have something like a virtual machine, one feature we might want is to separate my processes from the other processes on the computer. And we might not want the network interfaces and routing table entries to be shared across the entire OS.

You can do this with namespaces. In a networking namespace, you can run programs on any port you want without it conflicting with what's already running.

**Socket transport/socket protocols** provide the network transportation of application data from one machine to another (or from one process to another within the same machine).

**Tap interfaces** are kernel virtual network devices. Being network devices supported entirely in software, they differ from ordinary network devices which are backed by physical network adapters. They are special software entities which tell the Linux bridge to forward ethernet frames as it is. In other words, the virtual machines connected to tap interfaces will be able to receive raw ethernet frames.

For example, we can operate directly with [Layer](https://en.wikipedia.org/wiki/OSI_model) 4 sockets.

NB: The selftests are not necessarily operating at Layer 4 (TCP, UDP, etc.), but also at Layer 3 (IP, IPv4, IPv6).

A cool thing I learned is [**passt**](https://passt.top/passt/about/) which uses socket transport without needing to create interfaces on the host, and **pasta** which uses a tap interface to a network namespace (not a VM).
