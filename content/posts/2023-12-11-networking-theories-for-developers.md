---
title: Networking theories for developers
date: 2023-12-11
categories:
---

In this post, I am writing down some notes from the lecture I am watching on Udemy for CCNA preparation. I am not from the networking background but it is a critical domain that no one can skip if he is serious about computers.
The course link is https://www.udemy.com/course/ccna-complete/

## Transport Layer - OSI Layer 4

The transport layer's main responsibilty is to prepare the data packets. It uses either TCP or UDP protocol for this.
TCP is sequential. This means that when on a TCP connection a sender sends a packet, each packet has a sequence number attached to the header. The receiver sends acknowledgement for each packet to the sender. In a case where a packet is dropped midway, the receiver will not be able to send acknowledgement for that particular packet, hence the sender will know that the packet is dropped and it will resend that specific packet. In contrast, UDP does not care about the sequencing. TCP also does flow control which means if the receiver is not able to cope up with the packets being sent, the TCP protocol slows the sending of packets to maintain the flow.

## Network Layer - OSI Layer 3 - IP 

The most common protocol in this layer is the IP (Internet Protocol). The responsiblity of this layer is to route traffic between different devices. These devices can be part of the same network or other network. I get this example from cloudfare.

Imagine if Alice wants to send a message to Bob. If they are on the same network, the message can be send directly. It still uses the IP protocol but within the same network. If they both are on different networks, the responsiblity of sending Alice's packet to the network and then routing it to the other network on which Bob is connected. All of this happens in the IP protocol.

Also understand that the packet is a unit in which we measure the data that is being sent over the network. All packets have some metadata associated. We call it Headers. The networking software in the devices capture the data from the upper layer i.e Transport layer and attaches the headers into those packets. The fields in headers are source and destination IP addresses, packet lenght, IP version, Some information about if the packet is fragmented and so on. On the receving end, the networking software reads the headers and take actions accordingly.

I find this analogy easier to understand.

The network layer is a postal company. It carries all the envolopes for a company. Each envelope has the address of the sender and receiver. Receiver in this example is the company office. The postal company sort all the envolopes for a particular company and hand it over to the company's mail room. It does not bother about which deparment the envelopes are for.

Once the envelopes are present in the company's mail room, the Transport layers kicks in. It knows the department (Ports and Protocols). It takes the envelope, see the port number (department) and route the envelopes to those deparatments.

### Types of IP traffic
Hosts in networking means any connecting device to a certain network. This includes PC, printers, mobiles etc.


#### Unicast
Goes to single destination host
#### Broadcast
Goes to all hosts in on the subnet. It only works within the subnet. It is intentionally dropped by the router otherwise a single message will be flooded over to the entire internet.
#### Multicast
Goes to interested multiple hosts. Think of a radio channel. The radio station sends the audio content to all the hosts who tuned in to that particular radio channel. The condition here that the receiver must request the traffic.

## Subnetting
Managing a single network is difficult. What if there is a host down, how will a network engineer know unless he goes to each host one by one.
To solve this problem the network designed divides a single network into smaller manageable networks called Subnets.

to be continued...
