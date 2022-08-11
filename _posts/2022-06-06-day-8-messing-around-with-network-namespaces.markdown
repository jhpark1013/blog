---
title: Messing around with network namespaces
date: 2022-06-06T17:25:27-04:00
---

I haven't worked with namespaces yet so today I'm going to try some experiments to try to familiarize myself with network interfaces.

## Namespaces
I'm going to first make a namespace called `ns1`:
```
sudo ip netns add ns1
```
Running commands within a network namespace goes like this:
```
sudo ip netns exec ns1 <command>
```
If I run `bash` as the command, and run `nc -l 8900` in the namespace, `netstat` shows that the server exists.
```
$ sudo ip netns exec ns1 bash

root@jaehee:/home/jaehee# netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:8900            0.0.0.0:*               LISTEN
```
It's listening on `0.0.0.0:8900` which means it's listening all network interfaces. But since I haven't set up any interfaces yet, when I `curl`, no packets would be detected. And that is inline with what I see:
```
root@jaehee:/home/jaehee# curl localhost:8900
curl: (7) Couldn't connect to server
```
But if I setup a loopback `lo` interface, `curl` should send packets this time. and `nc` should receive them.   

Let's setup the loopback interface (in the namespace)
```
ip link set dev lo up
```
Now let's try `curl localhost:8900` from another instance of the namespace and we should see a packet show up. And it does!
```
root@jaehee:/home/jaehee# ip link set dev lo up
root@jaehee:/home/jaehee# nc -l 8900
GET / HTTP/1.1
Host: localhost:8900
User-Agent: curl/7.68.0
Accept: */*
```
So, when we add a network interface, the server starts on `0.0.0.0`.


## Routing
We have interfaces/devices that want to send data to eachother. And there are paths or routes connecting these interfaces where traffic flows. The packets of data are assigned a network device.

**Step 1** iptables has prerouting rules

iptables lets you create rules to match network packets and accept/drop/modify them. It's used for firewalls and NAT. Every iptable rule has a target (what to do with the matching packets. e.g. accept/drop/modify). The iptables have chains, and chains have rules. There are tables for filter, nat, etc. The chains in the table has input, forward, prerouting options. And the rules are what looks like `-s 172.17.0.0 -j DROP`

**Step 2** The packets get routed

You can see the routes on the route table.
On my computer, I have a few interfaces. When I run `ifconfig` I can see a list of my interfaces. I have `docker`, `lo`, `enp`, `wlp`, `wg`, `virbr`, etc. And if I run `sudo ip route list table all` I can see my routes.
```
$ sudo ip route list table all
default via 192.168.0.1 dev wlp0s20f3 proto dhcp metric 600
169.254.0.0/16 dev virbr0 scope link metric 1000 linkdown
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.64.2.0/24 dev wg0 scope link
192.168.0.0/24 dev wlp0s20f3 proto kernel scope link src 192.168.0.31 metric 600
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 linkdown
broadcast 127.0.0.0 dev lo table local proto kernel scope link src 127.0.0.1
local 127.0.0.0/8 dev lo table local proto kernel scope host src 127.0.0.1
local 127.0.0.1 dev lo table local proto kernel scope host src 127.0.0.1
broadcast 127.255.255.255 dev lo table local proto kernel scope link src 127.0.0.1
broadcast 172.17.0.0 dev docker0 table local proto kernel scope link src 172.17.0.1 linkdown
local 172.17.0.1 dev docker0 table local proto kernel scope host src 172.17.0.1
... etc ...
```
On the last line it says `local` in the front, so the packets sent to `172.17.0.1` goes to `local`.

**Step 3** tcpdump gets the packet

Once there is a network interface associated with the packet, the packet gets sent there and we can observe that the packet gets sent with tcpdump.

## Summary
Before routing, there is a prerouting chain in the iptables that happens.
When you send a packet to an IP address, the route table decides which network interface the packet goes through.
For the packets to go through the Linux kernel network stack, you need network interfaces. Once routed thorugh an interface, you can use tcpdump to capture packets.

<!--
```
root@jaehee:/home/jaehee# netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      77725/nc            
```

```
Check to see if the loopback interface was added properly by running:
```
ip netns exec ns1 ip address
``` -->
