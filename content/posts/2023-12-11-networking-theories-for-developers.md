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

## Types Of IP traffic
Hosts in networking means any connecting device to a certain network. This includes PC, printers, mobiles etc.

- Unicast -> Goes to single destination host
- Broadcast -> Goes to all hosts in on the subnet. It only works within the subnet. It is intentionally dropped by the router otherwise a single message will be flooded over to the entire internet.
- Multicast -> Goes to interested multiple hosts. Think of a radio channel. The radio station sends the audio content to all the hosts who tuned in to that particular radio channel. The condition here that the receiver must request the traffic.

## IPv4
To view your IP address you can use `ifconfig` command in UNIX based system and `ipconfig` in Windows. Usually in a typical setup we have different devices connected to internet via a Router. This is called a default gateway. The devices within a subnet can have communciation based on their private IP addresses. If they ever have to reach to internet it will go via router. The router or default gateway IP address can be checked via command `ip route`.

The router does something called NAT (Network Address Translation). It knows which devices is trying to access which destination on internet. It takes those details, Masks the IP of itself as a source IP and reach to the internet. Upon response, it knows the original device who has requested the content, so it returns back the response to the device.

To verity this part, I checked how my request will reach google in my OSX system.
```
$ route -n get www.google.com
   route to: 216.239.38.120
destination: default
       mask: default
    gateway: 192.168.0.1
  interface: en0
      flags: <UP,GATEWAY,DONE,STATIC,PRCLONING,GLOBAL>
 recvpipe  sendpipe  ssthresh  rtt,msec    rttvar  hopcount      mtu     expire
       0         0         0         0         0         0      1500         0
```

Here it tells me that in order to reach google, gateway is the the IP of my router.

One more thing to hightlight here is that we have limited number of IPv4 addresses. If we assign a unique IP addresses to all the devices in the world, we will exhaust all the available addresses in no time. That is why the router and the NAT mechanism plays a very important role in utilizing the availbel IP addresses

I am about to prove why there is such a limiatation of IPv4 addresses. To solve this we have to first understand the basic strucutre of IPv4 addresses. 

The IP address usually looks like this `192.168.0.4` or `216.239.38.120`. We know that these are the decimal representation of IP addresses. In reality, the IPv4 address is a 32 bit sequence of 0s and 1s. For now lets park this question that why they have chose 32 bits in the first place.

These 32 bits are divided into what we call and Octet. We take chunk of 8 bits. So in order to make 32 bits we multiply 8 bits into 4. That is why you see 4 parts in the decimal representation of IP addresses seperated by a dot (.)

some example of how an IP address could look like in binary formats are 
```
192.168.0.4 -> 11000000.10101000.00000000.00000100
216.239.38.12 -> 11011000.11101111.00100110.00001100
```

Now lets break down the limitation on the number of possible IPv4 addresses. We know that each part in an IPv4 address is 8 bits. Each bit can have either a value 0 or 1 which means there are 2 possiblities for each bit. By doin the maths we can say
```
8 bits means 2^8 (2 raised to the power of 8)= 256
```
This means that an octet can have maximum value 255 (start from 0 to 256). Now if we do this for the 3 remaining octects (Remember IPv4 has 4 octets) we will get some numbers like 
```
2^8.2^8.2^8.2^8
```
We can also write the above equation like
```
2^(8 x 4) or simply 2^32
```
When we explain the 2 ^ 32 we get a number approximately 4.3 billions.

This concludes that the design that our ancestors has used to distribute IPv4 address has this limitation. This was not foreseen because initally the internet was designed to be used in corporates or schools etc. They did not realize that one day it will become a necessity of every household and every person

## Subnetting
Managing a single network is difficult. What if there is a host down, how will a network engineer know unless he goes to each host one by one.
To solve this problem the network designed divides a single network into smaller manageable networks called Subnets.

## Subnet Mask
Before understanding subnet mask there is one point that is required to be understood is that how one device can communicate to another device.
We know that if we want to communicate to a device in the same network or specifically in the same subnet, it can reach that device directly. But if those two devices are not in the same subnet, than the request will be router through a router / default gateway which I have explained above.

Now the question that comes to mind is that how do the network identifies that the request is for the same subnet or differnt. To understand this we have to understand how the IPv4 address is constructed.

Imagine that we have two hosts

- 192.168.1.10
- 192.168.1.20

Now how do these two devices will know wether they are in same network or not. The answer is Subnet Mask. Just like IPv4 addresses, the subnet mask is also a 32 bit sequence of 1s and 0s. The condition is that the subnet mask always have contiguous 1s and 0s. We know that the highest number in an IPv4 address for an octet is 255 which have discussed above. The 1s in the subnet mask defines the network portion and the 0s represent the host portion. So in order to differentiate the network and host portions of the above two IPv4 addresses we must convert them to binary.
```
192.168.1.10 -> 11000000.10101000.00000001.00001010
192.168.1.20 -> 11000000.10101000.00000001.00000010
```

Now consider that the subnet mask we have applied to this network is 255.255.255.0
Again, the number 255 if converted to binary will result in `11111111` So the binary representation of this will be

```
255.255.255.0 -> 11111111.11111111.11111111.00000000
```


This shows that the first 3 octets of those ip addresses belongs to network portion and the last octet is host portion. Meaning that we are saying here that given this subnet mask, any device whose IPv4 address has first 3 octets same, it means they are on same subnet. In our example both  the ip addresses has `192.168.1` has the same values in first 3 octets.If one of these devices has any other value in the first 4 octets, this tells us that the devices are not in the same network and a default gateway must be used to communicate between them.

In summary, when we apply a subnet mask to a ip address on a system. We tell the system that top x number of octets are representing the network portion.

The subnet mask can also be defined in a / slash notation. for example 
`192.168.1.10 255.255.255.0 -> 11111111.11111111.11111111.00000000`
can also be written as `192.168.1.1/24`

It is simple as counting the number of consecutive 1s in the subnet mask Lets see few more examples.
`192.168.1.10 255.255.0.0` can be written as `192.168.1.1/16` -> This tells that the network portion is first 2 octets and last 2 are hosts portions
`192.168.1.10 255.0.0.0` can be written as `192.16f.1.1/8` -> This tells that the network portion is first 1 octet and last 3 are hosts portions

Another thing I have to remember is that in any subnet the 1st and last ip address is reserved. For example if I have a following network
`192.168.0.1/24` This means I can have IP addresses from `192.168.0.0` to `192.168.0.255` for the host devices. 

But I cannot assign an IP address `192.168.0.0` or `192.168.0.255` to any hosts. The former is reserved for the network address and the latter is the broadcast address for this subnet. A broadcast address is used to send a packet to all connected hosts. If I send a packet to this broadcast address, All the hosts in this subnet will receive that packet. This is also discussed in start of this post.






