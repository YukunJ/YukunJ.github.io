---
permalink: /blogs/opt_mem
title: "Optimize Memory Access"
excerpt: ""
author_profile: false
---

# Optimize Memory Access

Modern computers' memory operations account for the largest portion of performance bottlenecks and power consumptions. In this section we look at a few advices on how to optimize the program when TMA shows `Memory Bound`. Meanwhile, remember that there is always a viable option to buy better hardware: invest in a server with more memory channels or DRAM modules with faster transfer speed.

## Cache-Friendly Data Structures

**Access Data Sequentially**

To exploit spatial locality of the caches, making sequential memory access is very beneficial. It helps to bring in the next chunk of data needed ahead of time and faciliate hardware prefetching mechanism.

```c
// bad
for (col = 0; col < NCOLS; col++) {
    for (row = 0; row < NROWS; row++) {
        matrix[row][col] = row + col;
    }
}
// good
for (row = 0; row < NROWS; row++) {
    for (col = 0; col < NCOLS; col++) {
        matrix[row][col] = row + col;
    }
}
```

**Know and Choose the Right Container**

Understand how the built-in containers are built in the programming language you are using. For example, in C++ `std:list` requires additional pointer chasing in traversal. When appropriate, `std::vector` or `std::array` might yield better cache locality.

**Packing the Data**

By making the data more compact, the cache utilization could be improved. For example, here if we know the member `a`, `b` and `c` of `struct C` is a small enum value, we could use bit fields to pack them from 3 bytes each to just 1 byte each. This reduces the memory transferred and taken in cache.

```c
// S is 3 bytes
struct S {
    unsigned char a;
    unsigned char b;
    unsigned char c;
}

// S is 1 byte 
struct S {
    unsigned char a:4;
    unsigned char b:2;
    unsigned char c:2;
}
```

And also by rearranging the order of the fields in the struct, we could avoid additionally padding inserted by compiler. Or we could explicitly ask compiler to remove any padding by `__attribute__((packed))`.

```c
// S is 12 bytes
// b pad pad pad i i i i s s pad pad
struct S {
    bool b;
    int i;
    short s;
}

// S is 8 bytes
// i i i i s s b pad
struct S {
    int i;
    short s;
    bool b;
}
```

**Gathering Relevant Fields**

Imagine in the game programming, the `Soldier` struct would need access `attack`, `defense` and `health` in the battle stage, `coords` and `speed` in the marching stage. We should group close together those fields that are likely to be accessed together.

```c
struct S {
    unsigned attack;  // battle
    unsigned defense; // battle
    unsigned health;  // battle
    2DCoords coords;  // marching
    unsigned speed;   // marching
}
```

**Dynamic Memory Allocation**

Dynamic memory allocation has quite a few associated costs: 

+ it might need OS to perform a system call
+ in multi-threaded applications, dynamic memory allocation might need synchronization
+ *demand-paging* might incur a cost of page fault for every newly allocated memory page when accessed.

People have come up with different solutions to these issues. One way is to design a user-space memory allocation library that requests a big chunk of heap space from OS and conducts allocation and deallocation in user-space by the library itself. Examples include **jemalloc**, **tcmalloc**, etc.

In realtime-senstitive applications like embedded system or high-frequency-trading, people usually statically allocate all the needed space at the very beginning of the program and only deallocate them by the end of the running program. It requires to know how much space the application would need and reuse space in an efficient manner.

**Reduce DTLB Pressure**

TLB is a fast but small hardware device per-core that performs the virtual-to-physical address translations. Huge page is a mechanism to increase the TLB hit rate by increasing per memory page size. On Linux default page size is **4KB**, and **2MB**/**1GB** huge pages are available as well. There are Explicitly Huge Page (EHP) usage and Transport Huge Page (THP) usage.

**Explicit Huge Page**: the huge pages are reserved during boot time via `hugetlbfs` and could be allocated with `mmap` with `MAP_HUGETLB` option.

**Transparent Huge Page**: it could either be enabled system-wide by setting "always" in `/sys/kernel/mm/transparent_hugepage/enabled` or process-wide by setting "madvise" and `madvise(ptr, size, MADV_HUGEPAGE)` on an allocated heap memory area.

The overall tradeoff is that EHP requires additional system configuration for reserving the huge page upfront while THP incurs non-deterministic overhead from kernel to manage the fragmentation and swapping. So for realtime-critical applications EHP is preferred while for client-facing applications like database THP is more flexibly adopted.

For a more in-depth discussion on TLB, refer to my blog [TLB & Huge Page](/blogs/tlb_hugepage)

**Explicit Memory Prefetching**

By now we know it's expensive and likely lead to stall if the program has to fetch the memory location from DRAM instead of any level of D-cache. Modern CPUs have hardware prefetching mechanism but it suffers to predict correctly if the access pattern is not easily predictable. Alternatively programmer could explicitly hint the CPU to prefetch a memory address ahead of time in the hope that by the time the program needs to access it, it will already be in the D-Cache.

Consider the following example with a huge array `arr` pre-allocated:

```c
for (int i = 0; i < N; ++i) {
    size_t idx = random_distribution(generator);
    int x = arr[idx]; // cache miss
    doSomeExtensiveComputation(x);
}
```

Since the index is a random number, most likely the access `arr[idx]` will hit a cache miss. We can adopt a pipeline approach to prefetch the next one while executing for the current loop.

```c
size_t idx = random_distribution(generator);
for (int i = 0; i < N; ++i) {
    int x = arr[idx];
    idx = random_distribution(generator);
    // prefetch the element for the next iteration
    __builtin_prefetch(&arr[idx]);
    doSomeExtensiveComputation(x);
}
```

However notice there is caveats w.r.t explicit prefetching. Firstly it needs to be distanced properly away from when the memory address will be accessed, usually a couple hundred cycles. If too short, the prefetch might not have finished. If too long, it might pollute the cache and affect other parts of the program performance. And prefetching also consumes CPU resources. A general advice is to always measure the before/after performance metrics when adding explicit prefetching, especially the L3 cache hit rate.