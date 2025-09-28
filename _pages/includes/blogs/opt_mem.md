---
permalink: /blogs/opt_mem
title: "Optimize Memory Access"
excerpt: ""
author_profile: false
---

# Optimize Memory Access

Modern computers' memory operations account for the largest portion of performance bottlenecks and power consumptions. In this section we look at a few advices on how to optimize the program when TMA shows `Memory Bound`.

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