---
permalink: /blogs/double_mmap_trick
title: "Double mmap Virtual Memory Mapping Trick"
excerpt: ""
author_profile: false
---

In this blog, we will explore a very smart double `mmap` virtual memory mapping trick. In industry, a lot of ring buffer implementations exploit this trick to ease the implementation and boost performance.

Let's first look at a very vanilla implementation of ring buffer using `mmap`.

```c
struct circular_queue {
    size_t size;
    size_t write_index;
    size_t read_index;
    char *buffer;
};

struct circular_queue *create_queue(size_t size) {
    assert(size % getpagesize() == 0);
    struct circular_queue *queue = malloc(sizeof(struct circular_queue));
    queue->size = size;
    queue->write_index = 0;
    queue->read_index = 0;
    queue->buffer = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    return queue;
}

void enqueue(struct circular_queue *queue, char *data, size_t len) {
    size_t remain = queue->size - queue->write_index;
    size_t copy_to_tail = (remain >= len) ? len : remain;
    size_t overflow = len - copy_to_tail;
    memcpy(queue->buffer + queue->write_index, data, copy_to_tail);
    memcpy(queue->buffer, data + copy_to_tail, overflow); // wrap around
    queue->write_index = (queue->write_index + len) % queue->size;
}
```

`mmap` creates a virtual memory mapping of requested `size` into this process's address space starting at `queue->buffer`. The caveat here for the ring buffer is that when it reaches the end of the buffer, it needs to take modulo operation and wraps around, which is annoying in implementation and not great for performance. An illustration is provided as follows:

<img src="/images/blogs/single_mmap.jpg" alt="media 1" width="600">