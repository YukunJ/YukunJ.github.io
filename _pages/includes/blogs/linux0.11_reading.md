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

```assembly
load_setup:
  mov dx, 0x0000    # drive 0, head 0
  mov cx, 0x0002    # sector 2,	cylinder 0
  mov bx, 0x0200    # offset address is 512, in es(0x9000)
  mov ax, 0x0200+4  # service 2 (BIOS read_sector function), read 4 sectors
  int 0x13          # interrupt
  jnc ok_load_setup
```

Here it prepares the arguments in `dx`, `cx`, `bx` and `ax` and call the interrupt `0x13`, which results in disk reading of **4** sectors (512 bytes each) and place them at `0x90200`. 

Assuming the disk reading is successful, it jumps to the label `ok_load_setup`. The key operations there are

```assembly
ok_load_setup:
  ...
  mov ax, 0x1000
  mov es, ax
  call read_it
  ...
  jmpi  0, 0x9020
```

It will load the rest of the 256 sectors Linux operating system code (called `system` starting from the 6th sector) at `0x10000`. Then it jumps to `0x90200` which is the **2nd** sector containing the `setup` and start executing from there.

So far all the Linux operating system codes have been loaded from disk into memory with the following structures:
+ `bootsect`: **1** sector beginning at `0x90000`
+ `setup`: **4** sectors beginning at `0x90200`
+ `system`: **256** sectors beginning at `0x10000`

Now the booting process is done with `bootsect` and with the `jmpi  0, 0x9020` the workflow starts into `setup`. The beginning part of it continues to call BIOS interrupts to populate `0x90000` to `0x901FF` with some miscellaneous parameters like cursor, memory size, video card data, etc. Notice some part of the `bootsect` code is overwritten by these.

Then it's followed by another chunk of memory copying:

```assembly
  ...
  cli              # disable interrupts
  mov ax, 0x0000
  cld              # set move direction to be forward
do_move:
  mov es, ax       # set destination segment
  add ax, 0x1000
  cmp ax, 0x9000
  jz  end_move
  mov ds, ax       # set source segment
  sub di, di
  sub si, si
  mov cx, 0x8000   # will move a 2-byte word 32768 times = 64 KB
  rep movsw        # copy ds:si to es:di
  jmp do_move
end_move:
  ...
```

The routine here accomplishes two things:
+ temporarily disable interrupt because it will switch modes and re-setup the interrupt table
+ move the Linux `system` binary from [0x10000, 0x90000] to [0x00000, 0x80000]
