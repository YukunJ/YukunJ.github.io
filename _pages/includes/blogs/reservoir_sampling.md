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

1. Initialize an array ***R*** containing the first ***k*** items of the input. This serves as the initial reservoir.
2. For each new input $x_i$, generate a random integer $j  \in ( 1, \cdots, i ) $. If $j$ happens to be within $ \in ( 1, \cdots, k ) $, then $x_i$ will replace $R[j]$. otherwise $x_i$ is discarded.
3. Repeat step (2) until all inputs are processed.

This basic algorithm R looks simple enough. Here is the mathematical proof by induction. 

We will prove that during the processing of each element, after processing $i$ elements, each processed element has a uniform probability of $\frac{k}{i}$ to be retained in the sample.

+ Base: when $i$ is $k$, all the elements are included in the sample with probability 1.
+ Induction: 
    + assume after processing $m$ elements, each elements seen so far have a probability of $\frac{k}{m}$ to be retained in the sample. 
    + The next element $x_{m+1}$ has a probability $\frac{k}{m+1}$ to be included in the reservoir. So the chance that it replaces on particular element already in the reservoir is $\frac{k}{m+1}\frac{1}{k} = \frac{1}{m+1}$
    + So for any prior element $x_i$ where $i < m+1$, the new probability that it is retained in the sample is P(it is in the sample before $m+1$) * P(it is not replaced by $x_{m+1}$) = $\frac{k}{m}(1 - \frac{1}{m+1})=\frac{k}{m+1}$

Assume the average time complexity to generate a random number is $O(1)$, this basic algorithm R has an asymptotic complexity of $O(n)$.

**Algorithm L**


**Reference**

+ [Random Sampling with a Reservoir](https://www.cs.umd.edu/~samir/498/vitter.pdf)

+ [Reservoir sampling wiki](https://en.wikipedia.org/wiki/Reservoir_sampling#:~:text=Reservoir%20sampling%20is%20a%20family,to%20fit%20into%20main%20memory.)
