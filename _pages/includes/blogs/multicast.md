---
permalink: /blogs/multicast
title: "Multicast"
excerpt: ""
author_profile: false
---

Today we will talk about `Multicast`. This network protocol mechanism is used heavily in the trading world. Multicast is especially valuable in market data distribution because all trading servers should receive the exact same packet stream at nearly the same time (fairness), without the exchange having to maintain thousands of independent TCP connections (efficiency).

In particular we will be focusing on **Protocol-Independent IP Multicast-Sparse Mode (PIM-SM)** as a widely deployed multicast routing protocol, although in HFT environments it is often simplified or combined with SSM and L2 optimizations. PIM is called "protocol-independent" because it does not depend on a specific unicast routing protocol. It can leverage existing routing tables learned through OSPF, IS-IS, BGP, or other routing protocols.

Notice we will focus on the high-level idea instead of specific implementation details since I am a software engineer not a network engineer :)

## What is Multicast

<img src="/images/blogs/send-model.png" alt="send-model" width="500">

When we think about sending network packets, the first communication model that usually comes to mind is unicast. For example, TCP establishes a reliable bidirectional connection between a source and destination. However, sometimes 1 source wants to send messages to a group of destinations. The brute-force approach is to establish N separate unicast flows between the source and each destination.

Note the difference between broadcast and multicast: you can send a packet to a group, even if you yourself are not a member of that group.

Below is an illustration of how we could use unicast to implement multicast. We can see for a group of 3 receivers, the identical packet is replicated 3 times and crowded through the 2 switches.

<img src="/images/blogs/unicast.png" alt="unicast" width="500">

We need a more bandwidth-efficient protocol to implement multicast. Ideally, such 2 requirements should be achieved:

+ A router forwards a multicast packet at most once per outgoing interface that has downstream interested receivers 

+ Hosts not in the interested group should not receive the packet

Conceptually, multicast routing builds a distribution tree inside the network. The key idea behind multicast is that packets are replicated only at branching points in the network instead of at the sender itself.

The addresses from `224.0.0.0` to `239.255.255.255` are reserved for multicast addresses. So you cannot use an address in this range as an individual IP address.

## IP Multicast

The IP multicast service model defines three operations for end hosts: 

1. send packets to a group (even if are not a part of that group yourself). 
2. announce that you are joining a group.
3. announce that you are leaving a group.

The end hosts do not need to consider how is the multicast packet delivered to everyone in that group. The multicast routing infrastructure attempts to efficiently distribute packets to all subscribed members of the group.

### Host <-> First-Hop Router

**IGMP** (Internet Group Management Protocol) serves to let first-hop router know which end host is a member of which multicast group.

On a high level, the router will periodically query end host "which group(s) do you belong to?" and end host will reply with a list of multicast groups that it is interested in subscribing to. End hosts could also send unsolicited report about group membership to the router.

In this way, the first-hop router stays informed about the latest multicast group membership subscription of each end host. If the router doesn’t receive an update about a membership for a long time, the router will assume that membership has expired and invalidate it.

### Routing

Now we will talk about how each router learns how and where to route a multicast packet from source to a group of destination in **PIM-SM**. It has 2 phases: rendezvous point routing and SPT switch over.

#### Rendezvous Point Routing

#### SPT Switch Over

## Reference
1. Berkeley CS 168 Textbook (https://textbook.cs168.io/beyond-client-server/intro.html)

2. IGMP protocol (https://en.wikipedia.org/wiki/Internet_Group_Management_Protocol)

3. PIM-SM sparse mode (https://ccnp-sp.gitbook.io/studyguide/routing/multicast/pim-sm-sparse-mode)