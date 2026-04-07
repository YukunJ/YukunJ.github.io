---
permalink: /blogs/mesi
title: "MESI Cache Coherence Protocol"
excerpt: ""
author_profile: false
---

**MESI** protocol, also known as the Illinois protocol, is one of the most common cache coherence protocols used in multi-processor caches in modern shared-memory system. In this blog, we will go through

+ Why cache coherence protocol is needed
+ What is the MESI protocol
+ How the MESI protocol works

## Why cache coherence protocol is needed

Cache in a computer system is a nice thing to have in many aspects, in particular it saves the bandwidth and latency going to the main memory for a piece of data that's recently accessed. 

However, with the introduction of multi-processor systems with shared memory, keeping per-processor private caches (L1 and L2 in major architectures) in sync and aligned with what's in main memory becomes a headache, particularly the write-back cache.

Consider the following example:

| **time** | **event** | **value in C1** | **value in C2** | **value in main memory** |
|----------|-----------|-----------------|-----------------|--------------------------|
| t0       | C1 load   | 1               | -               | 1                        |
| t1       | C2 load   | 1               | 1               | 1                        |
| t2       | C2 write  | 1               | 2               | 1                        |
| t3       | C1 use    | ?               | 2               | ?                        |

If C1 and C2 are loading the same piece of data and C2 makes a write to it, at t3 C1 cannot directly use the previous value 1 in its private cache because it's outdated. There needs to be a mechanism to ensure C1 knows its cache is outdated. 

## What is the MESI protocol

Using 2 bits per cache line to encode, MESI stands for the 4 states each cache line could be in: **M**(Modified), **E**(Exclusive), **S**(Shared) and **I**(Invalid).

+ **M**(Modified): The cache line is present only in the current cache and the value has been modified locally and is dirty with respect to the main memory. The cache is responsible for supplying the data before any other processor can read it.
+ **E**(Exclusive): The cache line is present only in the current cache and consistent with the value in main memory now.
+ **S**(Shared): The cache line may be stored in other processors' caches and it's consistent with the value in main memory now.
+ **I**(Invalid): The cache line is not valid to use now. Needs a reload.

The main advantage of **MESI** over previous protocol like **MSI** is the **E**(Exclusive) state. When a cache line is in the Exclusive state, the local processor could write to this cache line without broadcasting invalidation through the bus since it knows it is the sole owner of this cache line.

## How the MESI protocol works

The MESI protocol works as a finite-state machine that transitions from one state to another based on 2 kinds of stimuli.

The first kind is the process its own operations:
+ PrRd: the processor requests to read a cache line
+ PrWr: the processor requests to write a cache line

The second kind is the process receiving bus side requests from other processors, which is facilitated by the snooping that monitors all the bus transactions:
+ BusRd: there is a read request to a cache line requested by another processor
+ BusRdX: there is a write request to a cache line requested by another processor that **doesn't already have the cache line**.
+ BusUpgr: there is a write request to a cache line requested by another processor that **already has that cache block residing in its own cache**.

**Table for first stimuli**

| Initial State | Operation | Response |
|--------------|----------|----------|
| Invalid (I) | PrRd | Issue BusRd to the bus. <br> Other caches check for a valid copy and inform the requester. <br> Transition to (S) Shared if others have a copy. <br> Transition to (E) Exclusive if none have a copy. <br> Data supplied by another cache if available, otherwise from main memory. |
|  | PrWr | Issue BusRdX on the bus. <br> Transition to (M) Modified in requester cache. <br> Other caches supply data if available, otherwise fetch from memory. <br> Other caches invalidate their copies. <br> Write updates the cache block. |
| Exclusive (E) | PrRd | No bus transaction. <br> State remains (E). <br> Cache hit on read. |
|  | PrWr | No bus transaction. <br> Transition to (M) Modified. <br> Cache hit on write. |
| Shared (S) | PrRd | No bus transaction. <br> State remains (S). <br> Cache hit on read. |
|  | PrWr | Issue BusUpgr on the bus. <br> Transition to (M) Modified. <br> Other caches invalidate their copies. |
| Modified (M) | PrRd | No bus transaction. <br> State remains (M). <br> Cache hit on read. |
|  | PrWr | No bus transaction. <br> State remains (M). <br> Cache hit on write. |

**Table for second stimuli**

| Initial State | Operation | Response |
|--------------|----------|----------|
| Invalid (I) | BusRd | No state change. Signal ignored. |
|  | BusRdX / BusUpgr | No state change. Signal ignored. |
| Exclusive (E) | BusRd | Transition to (S) Shared (another cache is reading). <br> Flush on bus with block data. |
|  | BusRdX | Transition to (I) Invalid. <br> Flush on bus with data from the now-invalidated block. |
| Shared (S) | BusRd | No state change (block remains shared). <br> May Flush on bus with data (design choice). |
|  | BusRdX / BusUpgr | Transition to (I) Invalid (requesting cache becomes Modified). <br> May Flush on bus with data (design choice). |
| Modified (M) | BusRd | Transition to (S) Shared. <br> Flush on bus with data (also written to main memory). |
|  | BusRdX | Transition to (I) Invalid. <br> Flush on bus with data (also written to main memory). |

For illustration purpose, here is the full state transition graph. Left blue arrows corresponds to self-generated stimuli and right red arrows correspond to bus snooping stimuli.

<img src="/images/blogs/mesi.svg" alt="mesi" width="500">

## Conclusion

The MESI protocol efficiently maintains cache coherence in multi-processor systems using four states: Modified, Exclusive, Shared, and Invalid.

By distinguishing between local processor actions and bus-level interactions, it minimizes unnecessary communication while ensuring correctness. Its design remains a cornerstone of modern cache coherence mechanisms.

## Reference
1. Papamarcos, M. S.; Patel, J. H. (1984). "A low-overhead coherence solution for multiprocessors with private cache memories".

2. https://en.wikipedia.org/wiki/MESI_protocol