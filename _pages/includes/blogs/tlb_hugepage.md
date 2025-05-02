---
permalink: /blogs/tlb_hugepage
title: "TLB & Huge Page"
excerpt: ""
author_profile: false
---

# TLB & Huge Page

## Introduction

Let's motivate the topic with a simple experiment.

Suppose we have a snippet of code as follows:
```c
const int N = (1 << 13); // 8k
int a[D * N];

for (int i = 0; i < D * N; i += D) {
    a[i] += 1;
}
```

Then we vary the step size `D` and note down the time taken to finish the program. Notice only the array size `a[D * N]` increases. The total iterations remains `N` times.

What would the performance be like as we vary the step `D` from **2** to **1024**?

<img src="/images/blogs/strides.png" alt="performance" width="500"/>

The performance drops quickly around step size `D=256`. The initial thoughts of many people, including myself, is: "Oh this makes sense. As the step size gets bigger, it is using the spatial locality of the data cache poorly. No wonder."

But if we look closer, this [test cpu](https://en.wikichip.org/wiki/amd/microarchitectures/zen_2) has 32KB L1 cache, 512KB L2 cache and 16MB L3 cache. Hence, after step `D >= 16`, each access will be on a difference cache line (16 * 4 = 64 bytes). In total it will retrieve `N` cache lines covering `N * 64 = 512KB` data, which fits exactly into L2 cache.

So the performance plot should have been a flat line after step size `D >= 16`. Something else is misbehaving here.

## Memory Paging 101

In modern OS, each process uses virtual memory for isolation and multiplexing purposes. The **MMU** (memory managament unit) on cpu helps to translate the virtual memory address to physical memory address. A small set-associative hardware device called **TLB** (translation looksaside buffer) helps MMU to cache a few most recently translated mapping from virtual address to physical address. The granularity of mapping is per page, which defaults to **4KB** on Linux x86-64.

<img src="/images/blogs/tlb_cases.jpg" alt="tlb cases" width="800"/>

In the case of a TLB hit, the MMU could save all the works it otherwise needs to perform in the case of a TLB miss. 

Now looking back to the experiment, the [test cpu](https://en.wikichip.org/wiki/amd/microarchitectures/zen_2) has 2048 entries in L2 TLB. With default page size being 4KB, it could cover a range of 2048 * 4KB = `8MB` memory.

When the step size `D=256`, the total size of the array is N * D * sizeof(int) = 2<sup>13+8+2</sup> = `8MB`. It breaches the L2 TLB size and basically lose the ability of caching during memory mapping.

We cannot change the fact that there is 2048 entries in the L2 TLB, unless we buy a new hardware. This leaves us with only one option - enlarge the page size.

## Huge Page Comes to Rescue

## Reference
1. https://www.hudsonrivertrading.com/hrtbeat/low-latency-optimization-part-1/
2. https://www.hudsonrivertrading.com/hrtbeat/low-latency-optimization-part-2/
3. https://en.algorithmica.org/hpc/cpu-cache/paging/
4. https://people.freebsd.org/~lstewart/articles/cpumemory.pdf
5. https://www.cs.cmu.edu/~213/lectures/11-vm-concepts.pdf