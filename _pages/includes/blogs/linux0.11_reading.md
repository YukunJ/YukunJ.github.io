---
permalink: /blogs/linux_source_reading
title: "Linux 0.11 Source Reading"
excerpt: ""
author_profile: false
---

# Linux Source Reading

Table of Contents
+ [Booting](#booting)

In this long blog, we will read over the source code for **Linux 0.11**, which is the first self-hosted version published in 1991 by Linus Torvalds.

## Booting

When we click the power button to start the computer, the PC (program counter) is initialized to be `0xFFFF0` as required by Intel manuals and starts executing there. This address contains a jump instruction to the **BIOS** (Basic Input/Output System) stored in the ROM (Read-Only Memory) chip. 

Linux's entry code is `boot/bootsect.s` which is compiled and placed at the first disk sector. The BIOS will move the binary placed at the first disk sector (512 bytes) to `0x7c00`, which is another hardcoded address that every operating system using BIOS for booting could assume to be placed at.

```assembly
# boot/bootsect.s # first 2 lines
mov ax,0x07c0
mov ds, ax
```

The first 2 lines of `bootsect.s` moves `0x07c0` into `ds` (Data Segment Register). This `0x07c0` left shifts 4 bits to become `0x7c00` for historical reasons. Once `ds` is set, all the future memory access will be taken by default as offset with respect to `ds`. This makes sense since BIOS forces all the operating system binary to be loaded at `0x7c00` to begin with. This simplifies all the future RAM access.

```assembly
# boot/bootsect.s # next 6 lines
mov ax, 0x9000
mov es, ax
mov cx, 256
sub si, si
sub di, di
rep movw
```

The next 6 lines of `bootsect.s` sets `es` to be `0x9000`, `cx` to be 256, and zeros out the `si` and `di`. 

A bit of assembly code explanation here: `rep` will repeat the following instruction `movw` `cx` times and decrements `cx` by 1 each time it finishes the instruction, until `cx` reaches zero. The `movw` instruction moves a **word** (2 bytes) from `ds:si` to `es:di` and increments `si` and `di` by 2.

So this part of assembly actually means copying the first `512` bytes of `bootsect.s` from `0x7c00` to `0x9000`. It copies the boot sector to a more controllable location in memory (in this case `0x9000`) so that it won't overwrite or being overwritten by other things.

And then it's followed by far jump instruction to the new location of the same `bootsect.s` binary starting with offset at the `go` label.

```assembly
jmpi go,0x9000 # pseudo code: cs = 0x9000; ip = go;
go:
  mov ax, cs
  mov ds, ax
  mov es, ax
  mov ss, ax
  mov sp, 0xff00
```

The far jump instruction sets the code segment `cs` to be `0x9000`. The rest 5 lines in `go` label set the data segment `ds` and stack segment `ss` to be `0x9000`, and stack pointer `sp` to be offset `0xff00`. The current stack location `ss:sp` is `(ss << 4) | sp = 0x9ff00`. On a high level, Linux is doing preparing work here to get ready to access

+ **code**:  `cs:ip`
+ **data**:  `ds:[address]`
+ **stack**: `ss:sp`

Recall so far only the first 512 bytes of the Linux code is loaded from disk into memory, the rest also needs to be loaded.