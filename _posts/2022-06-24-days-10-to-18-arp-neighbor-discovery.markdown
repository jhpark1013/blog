---
title: Networking concepts - Intro
date: 2022-06-24T14:39:51-04:00
---

In this blog, I will give a brief overview of how devices communicate with each other in networks.
<!-- ![network](/blog/assets/network_diagrams/network_06.PNG) -->

Keywords: ARP, DNS, DHCP, MAC, IP, VLAN

## Networks

Let's start with two devices that communicates with each other. The way you would do that is with a switch, which has ports that makes connections to network devices. This drawing shows two devices connected to a switch.

![network](/blog/assets/network_diagrams/network_01.PNG)

Each device has its own unique identifier called the MAC address. The switch knows which MAC addresses is on which port by keeping track of it in its table.

If a device knows the MAC address of its recipient, it can communicate with it by sending an ethernet frame (message) to the switch, telling it to forward the payload to the recipient.

The device can broadcast its MAC address to everyone on the network (on all the ports).

![network](/blog/assets/network_diagrams/network_02.PNG)

If you have multiple switches connected to each other (like in the drawing above), and your MAC address was broadcasted, the switch will forward to its connected switch. So your MAC address is in the first and second switch's table.  

![network](/blog/assets/network_diagrams/network_03.PNG)

Devices have MAC addresses which are physical addresses, and they also have logical addresses called IP addresses.

Every device has an ARP table that has the association between IP addresses and MAC addresses. When you start your device, its ARP table is empty. It fills entries in the ARP table by sending out a broadcasted ARP request.

It goes like:
1. Source Device: "Who has IP address 10.1.1.11?"
2. Target Device: "It's me! My MAC address is AB-13-34-...DE"
3. Source Device: Updates its ARP table.

We don't want to assign an IP address statically. This is why we have DHCP servers (Dynamic Host Configuration Protocols) that assigns IP addresses to your devices dynamically. So when your device joins a network at first, it broadcasts a DHCP request, and asks for an IP address. The DHCP offers an IP address that's not already claimed and sends this address to you. Then you can acknowledge that you have accepted the IP address. There is a lease time for each IP address to keep track of which addresses are still actively being use.

With dynamic IP addresses, every change of IP addresses in the network needs to be alerted. This is managed by the DNS server (Domain Name System). It gives every device in your network a hostname.

When device joins at first:
1. Device -> DHCP: Request for IP address to DHCP
2. DHCP -> Device: Here's your IP address. And here's the DNS server and default gateway's IP address.  
3. Device -> DNS: My hostname is client1, my IP address is 10.1.1.10
4. DNS: stores that info

Device asks DNS for client 1's IP, communicates with client 1, and updates ARP table:
1. Device -> DNS: "Who is client1?"
2. DNS -> Device: "This is the IP address of client1"
3. Device broadcasts: "Who has this IP?"
4. client1 -> Device: "It's me! Here's my MAC address"
5. Device: stores that info

This is for your local network. How does communication work with the outside world, like google.com?

![network](/blog/assets/network_diagrams/network_04.PNG)

When your computer tries to connect to to google with your web browser, your computer asks the DNS server for the IP address. But you don't want to keep track of all the domains in the world locally in your network. So, when your DNS server sees a request for google that is not in the local network, the DNS server forwards the request recursively to some upstream server (like 8.8.8.8). The request goes through your router, to your ISP, to google.   

If your target IP address looks much different from your own IP address, it won't try to resolve it locally, using ARP requests. Instead it send the message for the different-looking IP address to the router and default gateway.

Your router has a public IP address which was assigned by your ISP. When the router forwards your packet upstream, it will replace your local IP address with that public IP address.

The way it decides whether the IP address looks different is by masquerading.

![network](/blog/assets/network_diagrams/network_05.PNG)

Your computer compares the source and target IP addresses using the subnet mask (defined by your DHCP server). For example, the subnet mask 255.255.255.0 in binary is 11111111.11111111.11111111.00000000. So it will do a bitwise AND with your IP address. So in this case, your first three groups of numbers are the bits you are comparing. If them bits in the first 3 groups match, then they are in the same subnet.

## VLANs (virtual networks)

In a large network (with hundreds of thousands of machines) you don't want to allow every machine to talk to every other machine because of potential security breaches. VLANs allow you to subdivide your network and create sub networks. You can control the communication between these networks.

![network](/blog/assets/network_diagrams/network_07.PNG)

How it works: You can assign numbers to the ports of your switch. And if you configure your port to be on VLAN 3, it gets tagged, and that identifier is in your ethernet frame.

The package we're sending gets a VLAN tag (saying it's on VLAN 3). Then when it gets routed to the second switch's VLAN port tagged as 4. Since they are the tags are different labels, the port discards the packet.

If I'm trying to communicate through VLANs across multiple switches, then you have a problem. When a switch sees that the packet has a VLAN tag, it'll immediately dispose the packet for security reasons.  

If a computer is infected, you don't want it to send with the VLAN tag already set. You don't want to allow the computers to decide for themselves which VLAN to communicate on. So instead, you mark the ports on which you allow tagged packets. You can label them as trunking ports.  So you only allow VLAN tagged packets coming from the trunking ports.  

You can think of your network as a logical (vs physical) system. So you can think of your vlan 3 and 4 corresponding to two huge virtual switches spanning across your entire company or building. This allows you to think about your whole network in a different way and control your connection on a logical level; you don't have to have network cables going across the room messily. Just connect them logically in a VLAN setting. Connect your network in clean way, then do all the messy stuff with connections digitally.
