---
title: Thinking of the internship project tasks
date: 2022-06-02T10:23:07-04:00
---

Yesterday we had a project meeting where we discussed the tasks. There are close to 12 new tasks to choose from so I'm spending some time today to think how I'll divide my time. The goal/expectation is to choose one or two to work on during the next 3 months.

I'd like to try to understand how to setup the topologies and start thinking of how I would design a network topology for a specific scenario. And I see 3 tasks that will get me to understand network topologies: (1) providing filters for arp during neighborhood discovery, (2) fixing VLAN bridge binding, and (3) refactoring vxlan code.

A couple of terms to note:

**VLAN**: one cable, multiple LANs.
- VLAN is an extra tag on ethernet frames indicating that they belong on a different LAN than the default.
- Why is it used? VLANs make it easy for network admins to partition a single switched network to match the functional and security requirements of their system without having to run new cables. Devices will appear to be on the same LAN despite them being in different geographical locations.  
- VLAN allows a network of computers and users to communicate in a simulated environment as if they exist in a single LAN and share a single broadcast and multicast domain.  
- There is something called the BRIDGE_BINDING flag

**bridge**: a bridge is a network device that connects multiple LANs together to form a larger LAN.

**vxlan**: virtual extensible LAN. It's a network virtualization technology that attempts to address the scalability problems associated with large cloud computing deployments.

**fdb**: forward database

Creating network topologies involves setting up the namespaces and "routing" for the interfaces in the namespaces.  I'm not familiar with namespaces and routing. But I think a good place to start is observing the selftests and going over the links that Andy sent carefully.  

Roopa suggested running the selftests in debug mode to learn the network topologies. You can debug with the `-x` flag which I didn't know about. So I can try `bash -x ./fcnal-test.sh` to pause the test and observe what's going on.
`tools/testing/selftests/net/fib_tests.sh` is a good one to understand.

Here are some links that Andy suggested
- Very quick [intro](https://linuxhint.com/use-linux-network-namespace/) to network namespace.
- Thorough [step-by-step](https://linux-blog.anracom.com/2017/10/30/fun-with-veth-devices-in-unnamed-linux-network-namespaces-i/) and analysis of network namespace with bridge and VLANs (7 parts blog series).


Back in April, I was applying my understanding of hardware testing and trying to transfer the same testing process to the selftests. For example, I was trying to capture pcaps and analyzing the packets in my hypothetical test. I don't think this hardware analogy is not the right way of approaching the self tests. It's better to think about these tests as simulating virtual devices, and performing tests on these virtual interfaces.
