---
permalink: /blogs/tma
title: "Top-down Microarchitecture Analysis"
excerpt: ""
author_profile: false
---

Just as doing anything else complex in the world, performance engineering needs a good methodology to start with. A lot of time people have a spark in their mind "Oh it must be that {cache/IO/network/threading/anything} that my code is running slow" and head down immeidately into optimization, only to find out that it is not the bottleneck of the program in the end.

The **TMA** (Top-down Microarchitecture Analysis) is a good methodology proposed by Intel in 2014. It helps to identify ineffective usage of CPU microarchitecture of a program on a high level and could be used to establish a target direction for performance optimization.

<img src="/images/blogs/tma.png" alt="tma" width="500"/>

The above is a breakdown diagram of how TMA methodology analyze the execution of a program. Let's walk through it step by step:
