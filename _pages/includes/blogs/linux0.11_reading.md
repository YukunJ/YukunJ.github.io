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
  + [mem_init](#mem_init)
  + [trap_init](#trap_init)
  + [blk_dev_init](#blk_dev_init)
  + [tty_init](#tty_init)
  + [time_init](#time_init)
  + [sched_init](#sched_init)
  + [buffer_init](#buffer_init)
  + [hd_init](#hd_init)
+ [First Process](#firstprocess)
  + [move to user mode](#move_to_user_mode)
  + [scheduling process](#scheduling_process)
  + [fork() system call](#system_call)

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

### mem_init

`mm/memory.c` module is responsible for managing the main memory region. Its initialization is as follows:

```c
#define LOW_MEM 0x100000
#define PAGING_MEMORY (15*1024*1024)
#define PAGING_PAGES (PAGING_MEMORY>>12)
#define MAP_NR(addr) (((addr)-LOW_MEM)>>12)
#define USED 100

static long HIGH_MEMORY = 0;
static unsigned char mem_map [ PAGING_PAGES ] = {0,};

void mem_init(long start_mem, long end_mem)
{
  int i;

  HIGH_MEMORY = end_mem;
  for (i=0 ; i<PAGING_PAGES ; i++)
    mem_map[i] = USED;
  i = MAP_NR(start_mem);
  end_mem -= start_mem;
  end_mem >>= 12;
  while (end_mem-->0)
    mem_map[i]=0;
}
```

The meat of this init function is to properly initialize the page mapping array `mem_map` which indicates if a 4KB page is used or not. 

It first initializes each page to be "**used**". And then it sets the page index corresponding to the main memory region `[start_mem, end_mem]` for be `0` to indicate it's eligible for allocation.

In the `mm/memory.c` module another key function `get_free_page()` uses this `mem_map` key data structure. It's an inline assembly function so we will directly describe its procedure:

+ scan `mem_map` backward to find a free page
+ mark it as in-use and zero out the page
+ compute its physical address and return

### trap_init

Recall in the booting process we temporarily disable interrupts. Now in `trap_init` it will reset and register interrupt handler.

```c
// kernel/traps.c
void trap_init(void)
{
  int i;

  set_trap_gate(0,&divide_error);
  ...
  set_system_gate(4,&overflow);
  ...
  set_trap_gate(14,&page_fault);
  set_trap_gate(15,&reserved);
  set_trap_gate(16,&coprocessor_error);
  for (i=17;i<48;i++)
    set_trap_gate(i,&reserved);
  ...
}
```

It registers a bunch of interrupt descriptors to its corresponding handler functions. Number 17 to 48 is batch set to `reserved` because they will be overwritten later by each hardware device's initialization.

Both `set_trap_gate` and `set_system_gate` call into the inline assembly macro `_set_gate` in `asm/system/h`, which registers the interrupt handler into the `idt` table. The slight difference between these 2 set methods is that `set_trap_gate` registers the gate as kernel mode (only code running in kernel mode can use this gate) while `set_system_gate` registers the gate as user mode.

At this point all the interrupts handlers Linux registered are not yet taking effect. Later in the `main.c` function, `sti()` calls into assembly command `__asm__ ("sti"::)` which finally enables the interrupts.

### blk_dev_init

Coming next is the `blk_dev_init` in `main()` function, which stands for block device. 

```c
// kernel/blk_drv/blk.h
#define NR_REQUEST 32
struct request {
  int dev;         /* -1 if no request */
  int cmd;         /* READ or WRITE */
  int errors;
  unsigned long sector;
  unsigned long nr_sectors;
  char * buffer;
  struct task_struct * waiting;
  struct buffer_head * bh;
  struct request * next;
}

// kernel/blk_drv/ll_rw_blk.c
struct request request[NR_REQUEST];
void blk_dev_init(void)
{
  int i;

  for (i=0; i<NR_REQUEST ; i++) {
    request[i].dev = -1;
    request[i].next = NULL;
  }
}
```

This init function looks simple: just zero-initialize the array of `struct request`. Each field for the `struct request` is self-explanatory:

+ `dev`: device id, -1 if no request
+ `cmd`: READ or WRITE
+ `errors`: number of errors happen during operation
+ `sector`: the beginning sector
+ `nr_sectors`: for how many sectors
+ `buffer`: the place to put data in memory after READ
+ `waiting`: which process initiates this request
+ `bh`: head pointer for buffering area
+ `next`: links to the next pending `struct request` 

What is worth mentioning is that this `request` array makes the device I/O operation "**asynchronous**". The function `ll_rw_block` calls into `make_request` and returns from there. It adds an entry into the `request` array and delegates the work to the low-level device I/O driver to finish the task. And it could use the `wait_on_buffer(bh)` mechanism to check if the device has finished the I/O operations.

`make_request` will lock this buffer to ensure exclusive ownership and then search for an empty slot in the `request` array. Notice the last one third of the array is reserved for **READ**s. so for **WRITE**s it only searches the first two third of the array. and then it fills this `struct request` and add it to the tail of the linked list of requests which is currently waiting to be serviced.

### tty_init

Now it's time to make the keyboard input and display output working. There is a region of memory that's directly mapped to the display. Where is this region depends on which type of display and mode the system is using. For example the following will show "hello" on the screen. 

```assembly
# one byte for the character, the other byte for the color
mov [0xb8000], 'h'
mov [0xb8002], 'e'
mov [0xb8004], 'l'
mov [0xb8006], 'l'
mov [0xb8007], 'o'
```

The main meat is the `con_init()` function that `tty_init()` calls into. It first retrieves a bunch of information about display mode and parameters from `0x90006` (Recall in `setup.s` we place these information there), finds out the direct mapping region for the current display. And then it locates the cursor and enable keyboard interrupts as follows:

```c
#define ORIG_X  (*(unsigned char *)0x90000)
#define ORIG_Y  (*(unsigned char *)0x90001)
void con_init(void) 
{
  ...
  gotoxy(ORIG_X, ORIG_Y);
  set_trap_gate(0x21, &keyboard_interrupt);
  // toggle the bit and reset the keyboard interface
  outb_p(inb_p(0x21)&0xfd, 0x21);
  a=intb_p(0x61);
  outb_p(a|0x80, 0x61);
  outb(a, 0x61);
}

static inline void gotoxy(unsigned int new_x, unsigned int new_y)
{
  ...
  x=new_x;
  y=new_y;
  pos=origin + y*video_size_row + (x<<1);
}
```

The calling trace is a bit long for the keyboard input interrupte handler `keyboard_interrupt` which is an assembly function and it calls into `do_tty_interrupt` then `copy_to_cooked`, which eventually ends up in `con_write` in the case of display being the terminal.

```c
void con_write(struct tty_struct * tty) {
  ...
  __asm__("movb _attr,%%ah\n\t"
    "movw %%ax,%l\n\t"
    ::"a" (c),"m" (*(short *)pos)
    :"ax");
  pos += 2;
  x++;
  ...
}
```

This assembly code basically writes the keyboard input character into memory pointed by `pos` and then adjusts the cursor position by incrementing `pos` and `x`.

### time_init

How does the operating system know about the current time? In old days the computer is not always connected to the Internet. Let's dive into this part.

```c
#define CMOS_READ(addr) ({ \
  outb_p(0x80|addr, 0x70); \
  inb_p(0x71); \
})

#define BCD_TO_BIN(val) ((val)=((val)&15) + ((val)>>4)*10)

static void time_init(void)
{
  struct tm time;
  do {
    time.tm_sec = CMOS_READ(0);
    time.tm_min = CMOS_READ(2);
    time.tm_hour = CMOS_READ(4);
    time.tm_mday = CMOS_READ(7);
    time.tm_mon = CMOS_READ(8);
    time.tm_year = CMOS_READ(9);
  } while (time.tm_sec != CMOS_READ(0)); // to get a consistent snapshot
  BCD_TO_BIN(time.tm_sec);
  BCD_TO_BIN(time.tm_min);
  BCD_TO_BIN(time.tm_hour);
  BCD_TO_BIN(time.tm_mday);
  BCD_TO_BIN(time.tm_mon);
  BCD_TO_BIN(time.tm_year);
  time.tm_mon--;
  startup_time = kernel_mktime(&time);  
}
```

The way Linux knows about time to through interaction with hardware. Specifically, the **CMOS** device which is a RAM chip on the motherboard. It keeps track of time via a real-time clock (**RTC**), and is powered by a small battery that would last for several years. Hence even when the computer is powered off, the clock is still ticking for time keeping. `CMOS_READ` specifies the address to read into port `0x70` and read out the data from port `0x71`.

It fills up the `struct tm` and eventually calculate out `startup_time`, which stands for how many seconds has passed since 1970 Jan 1 0'o clock.

### sched_init

Now this is the important part of Linux: scheduling. Let's breakdown the `sched_init` piece by piece.

```c
// kernel/sched.c
void sched_init(void) {
  ...
  set_tss_desc(gdt+FIRST_TSS_ENTRY,&(init_task.task.tss));
  set_ldt_desc(gdt+FIRST_LDT_ENTRY,&(init_task.task.ldt));
  ...
}
```

Recall Linux sets up the `gdt` global descriptor table with code segment and data segment. Now it is setting up the **tss** (task state segment) and **ldt** (local descriptor table) for the first task. `struct tss_struct` stores all the register information for a process, which is primarily used when OS does context switching between different processes. Code running in kernel mode uses the code segment and data segment in `gdt`, while code running in user mode uses each process' code segment and data segment in `ldt`.

```c
struct desc_struct {
  unsigned long a,b;
}

struct task_struct * task[64] = {&(init_task.task), };

void sched_init(void) {
  ...
  int i;
  struct desc_struct * p;
  p = gdt + 6;
  for (i=1; i < 64; i++) {
    task[i] = NULL;
    p->a=p->b=0;
    p++;
    p->a=p->b=0;
    p++;
  }
  ...
}
```

Now it zero-initializes the rest 63 tasks's `struct task_struct` and their `tss` and `ldt` entries in `gdt`. Technically the task 0 `init_task` has not run yet. But Linux has already set it up so that in the future the "current" running code will become the task 0. We will copy paste the definition for the `struct task_struct` here for reference, which contains a lot of necessary information for managing each individual process.

```c
struct task_struct {
  long state; /* -1 unrunnable, 0 runnable, >0 stopped */
  long counter;
  long priority;
  long signal;
  struct sigaction sigaction[32];
  long blocked; /* bitmap of masked signals */
/* various fields */
  int exit_code;
  unsigned long start_code,end_code,end_data,brk,start_stack;
  long pid,father,pgrp,session,leader;
  unsigned short uid,euid,suid;
  unsigned short gid,egid,sgid;
  long alarm;
  long utime,stime,cutime,cstime,start_time;
  unsigned short used_math;
/* file system info */
  int tty;  /* -1 if no tty, so it must be signed */
  unsigned short umask;
  struct m_inode * pwd;
  struct m_inode * root;
  struct m_inode * executable;
  unsigned long close_on_exec;
  struct file * filp[NR_OPEN];
/* ldt for this task 0 - zero 1 - cs 2 - ds&ss */
  struct desc_struct ldt[3];
/* tss for this task */
  struct tss_struct tss;
};
```

Next, it sets up the `tr` register and `ldt` register to inform CPU about the location of `tss` and `ldt` in memory for the current process.

```c
#define ltr(n) __asm__("ltr %%ax"::"a" (_TSS(n)))
#define lldt(n) __asm__("lldt %%ax"::"a" (_LDT(n)))

void sched_init(void) {
  ...
  ltr(0);
  lldt(0);
  ...
}
```

Finally it enables the timer interrupt via **PIC** (programmable interrupt controller), which sends interrupt to Linux on a regular basis (100 times per second via `#define HZ 100`) and helps Linux to schedule process. And also the `system_call` is enabled so that user process could use low-level kernel function like `sys_read`.

```c
void sched_init(void) {
  ...
  outb_p(0x36,0x43);
  outb_p(LATCH & 0xff, 0x40); 
  outb(LATCH >> 8, 0x40);
  set_intr_gate(0x20, &timer_interrupt);
  outb(inb_p(0x21)&~0x01, 0x21);
  set_system_gate(0x80, &system_call);
  ...
}
```

### buffer_init

```c
// fs/buffer.c
extern int end;
struct buffer_head * start_buffer = (struct buffer_head *) &end;

void buffer_init(long buffer_end) {
  struct buffer_head * h = start_buffer;
  void * b = (void *) buffer_end;
  while ( (b -= 1024) >= ( (void *) (h+1)) ) {
    h->b_dev = 0;
    ...
    h->b_date = (char *) b;
    h->b_prev_free = h-1;
    h->b_next_free = h+1;
    h++;
  }
  h--;
  free_list = start_buffer;
  free_list->b_prev_free = h;
  h->b_next_free = free_list;
  for (int i=0; i<307; i++) {
    hash_table[i] = NULL;
  }
}
```

That's all for the `buffer_init`. Let's analyze it line by line. The extern variable `end` is set by the linker, which indicates the end of the Linux kernel program in memory (aka `[0, end]` is the Linux kernel program). Earlier in the `main` function it calculates the `buffer_end` and pass it in here as the argument. 

So the range `[start_buffer, buffer_end]` is what to be managed by the `buffer` module. It uses a doubly linked list `struct buffer_head`. Each manages a **1024**-byte block of buffer memory. The memory layout is as follows:

```bash
block_1 <--------- buffer_end
block_2
block_3
...
block_n
[wasted_space]
head_n
...
head_3
head_2
head_1 <--------- start_buffer
```

Finally, the **307** hashtable is used to help find if a block of data is in buffer space when the operating system needs to read data from block device. Iterating through the whole doubly linked list is not efficient. It uses `dev^block %307` as the hashing to find the buffer head directly.

```c
#define _hashfn(dev,block) (((unsigned)(dev^block))%307)
#define hash(dev,block) hash_table[_hashfn(dev,block)]
```

### hd_init

floppy disk is retried by now. so we will skip `floppy_init()` and only take a look at the hardware disk `hd_init()`.

```c
void main(void) {
  ...
  hd_init();
  floppy_init();
  ...
}

// kernel/blk_drv/hd.c
#define MAJOR_NR 3
#define NR_BLK_DEV 7
#define DEVICE_REQUEST do_hd_request

struct blk_dev_struct {
  void (*request_fn)(void);
  struct request * current_request;
};

extern struct blk_dev_struct blk_dev[NR_BLK_DEV];

void hd_init(void)
{
  blk_dev[MAJOR_NR].request_fn = DEVICE_REQUEST;
  set_intr_gate(0x2E,&hd_interrupt);
  outb_p(inb_p(0x21)&0xfb,0x21);
  outb(inb_p(0xA1)&0xbf,0xA1);
}
```

Linux uses an array of `struct blk_dev_struct` to manage different types of device. Each of it contains a function pointer `request_fn` recording how to interact with that device. This is the C/Linux way of polymorphism. Here it initializes the index 3 (`hd`)'s function pointer to `do_hd_request`.

Aferwards it registers the `0x2E` interrupt handler to be `hd_interrupt`. The 2 last lines of IO ports writing and reading enables hardware disk controller to send interrupt to the CPU.

## FirstProcess

### move_to_user_mode

```c
// include/asm/system.h
#define move_to_user_mode() \
_asm { \
  _asm mov eax,esp \
  _asm push 00000017h \
  _asm push eax \
  _asm pushfd \
  _asm push 0000000fh \
  _asm push offset l1 \
  _asm iretd \
_asm l1: mov eax,17h \
  _asm mov ds,ax \
  _asm mov es,ax \
  _asm mov fs,ax \
  _asm mov gs,ax \
}
```

Let's put the conclusion first: 
+ **Code**: could only jump to code in same privilege level via a regular `jmp` or `call` (kernel mode to kernel mode, user mode to user mode)
+ **Data**: could access data in same or lower privilege level (kernel mode can access user mode data, but not vice versa)

Recall when interrupt happens, CPU will push 5 variables into the stack for us. And upon returning from the interrupt, it will pop back the 5 variables. The order is as follows:

```bash
Upon interrupt:
push 
+ ss (stack segment )
+ esp (stack pointer)
+ eflags (flag register)
+ cs (code segment)
+ eip (instruction register)

Upon return from the interrupt:
pop them back in reverse order
```

Here Linux plays a nice trick of "faking an interrupt". It manually pushes the 5 variables into the stack as of an interrupt has happened. 

In x86, the bottom 2 bits of a segment selector are the Requested Privilege Level (**RPL**). Here the last 2 bits of `00000017h` and `0000000fh` (for `ss` and `cs` respectively) are `0b11` which stands for ring 3 user mode. And the relative address of tag `l1` is pushed in as the return address. Hence, when `iretd` is executed, the code flow actually continues to move to the next line with privilege level switched to user mode already.

### scheduling_process

Now we will look at the workflow of how Linux schedules and switches between different processes. Recall it enables the clock time interrupt earlier in `sched_init()`. So every time the time interrupt happens, the CPU could decide to switch to a different process to run.

A few things to consider first is: What it takes to context switch between different processes?

+ **context**: one CPU only has 1 set of registers. So every process has to "share" these registers upon switching. 

In each process' `struct task_struct` there is a `struct tss_struct tss` which contains all the value copy of the set of registers upon context switching.

```c
// include/linux/sched.h
struct task_struct {
  ...
  struct tss_struct tss;
}

struct tss_struct {
  long  back_link;	/* 16 high bits zero */
  long  esp0;
  long  ss0;		/* 16 high bits zero */
  long  esp1;
  long  ss1;		/* 16 high bits zero */
  long  esp2;
  long  ss2;		/* 16 high bits zero */
  long  cr3;
  long  eip;
  long  eflags;
  long  eax,ecx,edx,ebx;
  long  esp;
  long  ebp;
  long  esi;
  long  edi;
  long  es;		/* 16 high bits zero */
  long  cs;		/* 16 high bits zero */
  long  ss;		/* 16 high bits zero */
  long  ds;		/* 16 high bits zero */
  long  fs;		/* 16 high bits zero */
  long  gs;		/* 16 high bits zero */
  long  ldt;		/* 16 high bits zero */
  long  trace_bitmap;	/* bits: trace 0, bitmap 16-31 */
  struct i387_struct i387;
};
```

+ **time**: CPU needs to know how long each process has run to be able to make a decision about context switching.

```c
struct task_struct {
  ...
  long counter;
  long priority;
  ...
}

void do_timer(long cpl) {
  ...
  if ((--current->counter)>0) return;
  schedule();
}
```

This `counter` will decrement every time interrupt happens for the current running process. If it reaches 0, then CPU will schedule another process to run to be fair.

And every time a new process is scheduled to run, its `counter` is initialized to its `priority` field. Intutively, the higher the priority, the longer it could own the CPU and run its task before the CPU is given away to another process.

+ **state**: not every process is able to utilize the CPU at any given time. For example it might be blocked waiting on I/O read/write from the disk.

Hence Linux also needs to keep track of "if this process is willing/able to utilize the CPU now". It keeps track of each process' readiness via the `state` variable.

```c
#define TASK_RUNNING  0
#define TASK_INTERRUPTIBLE  1
#define TASK_UNINTERRUPTIBLE  2
#define TASK_ZOMBIE 3
#define TASK_STOPPED  4

struct task_struct {
  ...
  long state;
  ...
}
```

Now let's take a look at the `schedule()` when the current process' `counter` decrements to zero and the operating system decides to switch to a different process.

```c
void schedule(void)
{
  int i, next, c;
  struct task_struct ** p;
  ...
  while (1) {
    c = -1;
    next = 0;
    i = NR_TASKS;
    p = &task[NR_TASKS];
    while (--i) {
      if (!*--p)
        continue;
      if ((*p)->state == TASK_RUNNING && (*p)->counter > c)
        c = (*p)->counter, next = i;
    }
    if (c) break;
    for(p = &LAST_TASK; p > &FIRST_TASK; --p)
      if (*p)
        (*p)->counter = ((*p)->counter >> 1) + (*p)->priority;
  }
  switch_to(next);
}
```

Basically Linux is doing 3 simple things here:

1. Find the `TASK_RUNNING` state process with maximum remaining `counter`, call it `next`.
2. If all `TASK_RUNNING` processes' remaining `counter` are 0, reinitialize each process' to be `counter = counter/2 + priority`.
3. call the assembly function `switch_to(next)`.

There is one magic part `ljmp` long jump in the `switch_to` call. The CPU will recognize it if `ljmp` jumps to a TSS (Task State Segment) and will automatically save the old task's state and load the new task's state.

### system_call

Now we will take a look at how system call works in Linux through the example of `fork()` from the `main` function.

The following macro template constructs the function defintions for all system calls that do not take in any parameter. `fork` is one of them.
```c
static _inline _syscall0(int, fork)

#define _syscall0(type, name) \
type name(void) \
{ \
long __res; \
__asm__ volatile ("int $0x80" \
  : "=a" (__rse) \
  : "0" (__NR_##name)); \
if (__res >= 0) \
  return (type) __res; \
errno = -__res; \
return -1; \
}
```

Notice the key line here is `int 80h` with parameter `__NR_fork = 2`. Recall in `sched_init` we set the interrupt number `0x80`'s handler:

```c
// kernel/sched.c
set_system_gate(0x80, &system_call);
```

And this `system_call` function will step into the `sys_call_table` array to retrieve the corresponding system call handler function.

```assembly
// kernel/system_call.s
_system_call:
  ...
  call [_sys_call_table + eax*4]
  ...
```

This `sys_call_table` defines all the system call functions that Linux provides users with.

```c
// include/linux/sys.h
typedef int (*fn_ptr)();
fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, sys_read,
sys_write, sys_open, sys_close, sys_waitpid, sys_creat, sys_link,
sys_unlink, sys_execve, sys_chdir, sys_time, sys_mknod, sys_chmod,
sys_chown, sys_break, sys_stat, sys_lseek, sys_getpid, sys_mount,
sys_umount, sys_setuid, sys_getuid, sys_stime, sys_ptrace, sys_alarm,
sys_fstat, sys_pause, sys_utime, sys_stty, sys_gtty, sys_access,
sys_nice, sys_ftime, sys_sync, sys_kill, sys_rename, sys_mkdir,
sys_rmdir, sys_dup, sys_pipe, sys_times, sys_prof, sys_brk, sys_setgid,
sys_getgid, sys_signal, sys_geteuid, sys_getegid, sys_acct, sys_phys,
sys_lock, sys_ioctl, sys_fcntl, sys_mpx, sys_setpgid, sys_ulimit,
sys_uname, sys_umask, sys_chroot, sys_ustat, sys_dup2, sys_getppid,
sys_getpgrp, sys_setsid, sys_sigaction, sys_sgetmask, sys_ssetmask,
sys_setreuid,sys_setregid, sys_iam, sys_whoami };
```

Here we finally find the kernel-level function `sys_fork`.

```assembly
// kernel/system_call.s
_sys_fork:
  call _find_empty_process
  testl %eax,%eax
  js lf
  push %gs
  pushl %esi
  pushl %edi
  pushl %ebp
  pushl %edi
  pushl %ebp
  pushl %eax
  call _copy_process
  addl $20, %esp
1: ret
```