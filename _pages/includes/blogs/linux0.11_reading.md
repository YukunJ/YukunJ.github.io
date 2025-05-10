---
permalink: /blogs/linux_source_reading
title: "Linux 0.11 Source Reading"
excerpt: ""
author_profile: false
---

# Linux Source Reading

Table of Contents
+ [Booting](#booting)
+ [Initialization](#initialization)

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

Linux then needs to switch from 16-bit real mode to 32-bit protected mode for historical reasons that we will skim over. Recall so far when we calculate the address, it is `segment address << 4 | offset`. The OS will now set up the `idt` (Interrupte descriptor table) and `gdt` (global descriptor table) to prepare the mode switch, after which the memory calculation process will be: the segment register stores an index into the `gdt` which allows it to find the segment address. And then plus the offset it will get the final memory address.

```assembly
lidt idt_48          # load idt with 0,0
lgdt gdt_48          # load gdt with appropriate

gdt_48:
  .word 0x800        # gdt limit=2048, 256 GDT entries  
  .word 512+gdt, 0x9 # 0x90200 + gdt is its real address in setup.s
```

It also needs to enable the **A20** (Address Line 20 Gate) so that it could start to use the 21st bit of the memory address and beyond 1 MB.

```assembly
inb  al, 0x92          # read current value from port 0x92 (System Control Port A)
orb  al, 0b00000010    # set A20 Gate Enable bit
outb 0x92, al          # write back to port 0x92
```

Now it's ready to switch the mode.

```assembly
mov ax, 0x0001  # protected mode (PE) bit
lmsw ax         # load the value into CR0 register (Machine Status Word)
jmpi 0, 8       # jump to segment 8 of offset 0
```

It load the PE bit into the register and jump to segment **8** and offset 0. Recall it has setup the `gdt`. Segment 8 corresponds to index **1** code segment which has base address **0**. Hence essentially this `jmpi` jumps to address `0x0` which is the beginning of `system` code. We will start to look at `head.s` now as the very beginning of `system` binary.

The first part of `head.s` sets up the kernel stack. 

```assembly
_pg_dir:
_startup_32:
  mov eax, 0x10
  mov  ds, ax
  mov  es, ax
  mov  fs, ax
  mov  gs, ax
  lss esp, _stack_start
```

Notice the label `_pg_dir` is where the page directory will be placed later when Linux starts paging. So the code here will actually be overwritten later as well.

It points the 4 segment register `ds`, `es`, `fs` and `gs` to `0x10`, the 2nd index into the `gdt`, namely the data segment descriptor.

For this `_stack_start`, we need reference into `sched.c`.

```c
long user_stack[4096 >> 2];

struct {
  long *a;
  short b;
} stack_start = {&user_stack[4096 >> 2], 0x10};
```

So `lss esp, _stack_start` sets `ss` (stack segment register) to the same data segment and `esp` to be the address of the last element of `user_stack`. So it means the kernel stack is **4kb** and the stack pointer points to the top of it and will grow downward.

```assembly
call setup_idt # set up 256 descriptor to point to a function 'ignore_int'
call setup_gdt # same as before
```

It then re-setup the `idt` and `gdt`. In particular, the `idt` is setup to have **256** entries all point to a dummy `ignore_int` function that does nothing. Later on each module could overwrite it and provide the handler function for some interrupt descriptor they care about. 

Here comes the last piece before it jumps into the `main` C function and says goodbye to assembly. It needs to enable the paging mechanism.

```assembly
jmp after_page_tables
...
after_page_tables:
  push 0
  push 0
  push 0
  push L6
  push _main
  jmp setup_paging
```

One thing to notice here is that it pushes the address of `main` function as the return address for the call to `setup_paging`. Therefore when it finishes, the control flow will return jump into `main`.

A few words about memory paging: operating system provides each running process the illusion that it has the whole address space. But in reality each process work with virtual memory address. There is a small hardware device called **MMU** (Memory management unit) on CPU board that does the virtual to physical memory address translation. In the software part, Linux just needs to set up the page table for MMU to work on.

Linux 0.11 mandates the maximum memory is 16 MB, aka the biggest address is `0xFFFFFF`. Each page is 4KB. It uses a 2-level paging here. The page directory will contain 4 page tables. Each page table corresponds to 1024 pages. 

Hence `4 * 1024 * 4KB = 16 MB` will be enough to represent the address space.

With all the knowledge in mind, let's look at the final piece of assembly that enables paging mechanism:

```assembly
setup_paging:
  mov ecx, 1024*5      # set counter to be number of 4-byte entries in 5 pages (1 directory + 4 tables)
  xor eax, eax
  xor edi, edi         # pg_dir is 0x000
  pushf
  cld
  rep stosd            # repeat 1024*5 times to store value 0 in eax into edi, basically zeroes out the 5 pages
  mov eax, _pg_dir
  mov [eax], pg0+7     # link page into page directory, 7 = 0b111 set up the present, writer and supervisor bits
  mov [eax+4], pg1+7
  mov [eax+8], pg2+7
  mov [eax+12], pg3+7
  mov edi, pg3+4096    # edi being the last dword in the last page table
  mov eax, 00fff007h   # eax being the last 4KB page in the first 16MB plus 7
  std
  ...                  # fill the page backwards with identity mapping
  or eax, 80000000h    # setup the 31st PG (Paging Enable) bit
  mov cr0, eax         # enable paging
  ret
```

Now let's go to `main.c`.

## Initialization

Let's take a look overall on what does the `main` function of Linux operating system do.

```c
void main(void) {
  /* Part A */
  ROOT_DEV = ORIG_ROOT_DEV;
  drive_info = DRIVE_INFO;
  memory_end = (1<<20) + (EXT_MEM_K<<10);
  memory_end &= 0xfffff000;
  if (memory_end > 16*1024*1024)
    memory_end = 16*1024*1024;
  if (memory_end > 12*1024*1024)
    buffer_memory_end = 4*1024*1024;
  else if (memory_end > 6*1024*1024)
    buffer_memory_end = 2*1024*124;
  else
    buffer_memory_end = 1*1024*1024;
  main_memory_state = buffer_memory_end;
  /* Part B */
  mem_init(main_memory_start, memory_end);
  trap_init();
  blk_dev_init();
  chr_dev_init();
  tty_init();
  time_init();
  sched_init();
  buffer_init(buffer_memory_end);
  hd_init();
  floppy_init();
  /* Part C */
  sti();
  move_to_user_mode();
  if (!fork()) {
    init();
  }
  for(;;) pause();
}
```

It's very concise and clear on what it is doing. We can roughly cut it into 3 parts: Part A calculates some parameters. Part B initializes each module. Part C moves into user mode and starts the first shell. Let's dive into each one of them now.

It first calculates the `memory_end` (which is 1MB + extended memory size) and `buffer_memory_end` (same as `main_memory_begin`). So basically the memory space is divided into 3 chunks right now. The first part is the Linux kernel program binary. The second part is the buffer memory. The third part is main memory.