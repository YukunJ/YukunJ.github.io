---
permalink: /blogs/reservoir_sampling
title: "Reservoid Random Sampling Algorithm"
excerpt: ""
author_profile: false
---

In this blog, we will look at a very classic algorithm -- **Reservoir sampling** to solve such a problem:

+ Choose a random sample ***k*** items without replacement from a population of unknown size ***n*** in a single pass.

It has the following constraints:

+ the size of ***n*** items are too big to fit into main memory
+ the items are presented in a streaming fashion and the algorithm cannot look back
+ at any point, the algorithm needs to be able to present a random sample of size ***k*** based on the data it processes so far

We will first introduce a basic **Algorithm R** by Jeffrey Vitter and then see an optimized version of it named **Algorithm L**.

<img src="/images/blogs/reservoir.png" alt="media 1" width="300">

**Algorithm R**

**Algorithm L**


**Reference**

+ [Random Sampling with a Reservoir](https://www.cs.umd.edu/~samir/498/vitter.pdf)

+ [Reservoir sampling wiki](https://en.wikipedia.org/wiki/Reservoir_sampling#:~:text=Reservoir%20sampling%20is%20a%20family,to%20fit%20into%20main%20memory.)
