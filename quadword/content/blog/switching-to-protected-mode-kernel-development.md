---
title: Switching to Protected Mode - Kernel Development - Part 8
date: 2022-02-02T18:20:00+00:00
draft: false

image: /img/thumbs/kernel.png

description: "Let's start working in Protected Mode"

categories:
  - Kernel Development
tags:
  - Programming
  - Assembly
  - Low-level
  - Operating systems
  - Computer Science

type: post
---

## What is Protected Mode?

Before we switch to protected mode, let\'s first learn what is it and
what we have to do to switch to it.

Protected mode is a processor state in x86 architectures, which gives
access to memory protection, 4 GiB address space and much more. It also
provides memory and hardware protection, support to different memory
schemes and access to 4 GiB of memory.

### Protection

Protected mode allows you to protect memory from being accessed and it
also allows you to protect hardware from being accessed from user
programs.

![](https://upload.wikimedia.org/wikipedia/commons/thumb/2/2f/Priv_rings.svg/600px-Priv_rings.svg.png)

*Ring Protection image.*

How this works is, you have different protection levels in your
processor. We have what is known as ring zero which is the most
privileged ring, you can do anything with this ring, you can talk with
hardware, you can destroy memory, you can write and read memory anywhere
you want, overwriting things that shouldn\'t be overwritten and much
much more. The kernel runs on ring zero.

You also have Ring 1 and Ring 2, which are generally not used, but they
can be used for device drivers. Finally, you have Ring 3, this is the
least privileged ring, you put the processor into Ring 3 when you are
running user code, this prevents user programs from overriding kernel
memory, talking directly with hardware, accessing memory of other
programs and using privileged instructions such as *sti* to enable
interrupts and *cli* to disable them. In fact, how a user program talk
to the kernel is through interrupts, and through these interrupts the
processor can put itself back into Ring 0 so it can begin running
privileged code, and once it\'s finished, the kernel should put the
processor into Ring 3 back again so user programs can continue their
execution.

### Memory Schemes

There are different memory schemes in protected mode, the segment
registers become selective registers and the other memory scheme we have
available is paging, this memory scheme allows you to remap memory
addresses so address 0x00 points to another address, for example.

#### Selective Memory Scheme

As I said before, the segment registers become selective registers and
the selectors point to data structures that describe memory ranges and
the permissions the ring level required to access a given range.

#### Paging Memory Scheme

The paging memory scheme is the most common scheme for all operating
systems and kernels, in fact, if you would like to support the x64
architecture, you MUST use this memory scheme. In this memory scheme you
have virtual addresses and you can point those to physical addresses and
memory protection is much easier to control.

### Memory Access

While in Protected Mode, we can work with 32-bit registers and
instructions, we can address up to 4 GiB of memory at any time and we
are no longer limited to the 1 MiB of memory provided by Real Mode.

When we switch to Protected Mode, we are actually writing a 32-bit
kernel.

## Switching to Protected Mode {#switching-to-protected-mode-1}

Before switching to protected mode, I recommend you taking a look at
this [osdev page](https://wiki.osdev.org/Protected_Mode) as it contains
a lot of information about protected mode and even a guide to how to
switch to it.

As you can see in the osdev page section \"Entering Protected Mode\", in
the snippet of code, there\'s a new instruction *lgdt* which loads a
Global Descritor Table and then we have to set the protection enable bit
in the cr0 register and then, we jump with the selector and offset to
where we want to load and that\'d be it, we should be now in Protected
Mode.

If you go to the Global Descriptor Table in
[osdev](https://wiki.osdev.org/Global_Descriptor_Table), you\'ll see
that it contains entries telling the CPU about memory segments. The GDT
is loaded using the *lgdt* assembly instruction, it expects the location
of a GDT description structure.

In that same osdev page, you\'ll find more information about GDT, the
flags it takes and much more regarding GDT. We are not going to care
that much about it, we will set the default values as we will not use
GDT but Paging.

Now, go to your *boot.asm* file and delete all what we did in the
previous entry, delete the following code in your file:

``` asm
mov ah, 2
mov al, 1
mov ch, 0
mov cl, 2
mov bx, buffer
int 0x13

jc err

mov si, buffer
call print
```

And also, delete the *err*, *err~msg~*, *buffer*, *print* and
*print~char~* label.

Let\'s now create the Global Descriptor Table. At the end of the *jump
\$* instruction in the *step2* label, add two labels a *gdt~start~* and
a *gdt~end~*, we will need these labels later to load our GDT:

``` asm
gdt_start:
                ; GDT Code goes here
gdt_end:
```

In between those labels, create another label which will represent a
null segment, this label will contain just 64 bits of zeros:

``` asm
gdt_null:
    dd 0x0
    dd 0x0
```

And now, let\'s code the offset 0x8 (you can read about all these
offsets in the section \"Segment Descriptor\" in the GDT page in osdev)
this offset will define our code descriptor, so just do (after
gdt~null~):

``` asm
gdt_code:
    dw 0xffff
    dw 0
    db 0
    db 0x9a
    db 11001111b
    db 0
```

**Note**: Don\'t worry that much about these values, as I mentioned
before, these are just the \"defaults\" as we don\'t want to work with
these descriptors, we will be using paging. Also, on the repository
containing this project (see it at the end of this entry, you will find
comments).

Now, let\'s code the offset 0x10 which is our data segment:

``` asm
gdt_data:
    dw 0xffff
    dw 0
    db 0
    db 0x92
    db 11001111b
    db 0
```

And finally, after the *gdt~end~* label, create another label called
gdt~descriptor~ and add the following code to it:

``` asm
gdt_descriptor:
    dw gdt_end - gdt_start - 1
    dd gdt_start
```

The instruction *dw gdt~end~ - gdt~start~ - 1* gives us the size of the
descriptor and *dd gdt~start~* is the offset.

Note that our origin is 0, that is very important when creating our
descriptor, we can fix it very easily, just change the origin back to
0x7c00 and after our short jump, instead of having *jmp 0x7c0:step2*
change it to *jmp 0:step2*, so the beginning of your code should look
like this:

``` asm
    ORG 0x7c00
    BITS 16

_start:
    jmp short start
    nop

    times 33 db 0

start:
    jmp 0:step2
```

Also, don\'t forget to change the registers we initialised previously
back to 0x00 instead of 0x7c00 (except the stack pointer register *sp*).

Now, after *step2* create a new label called *load~protected~* which
will be the one that performs the switch from Real Mode to Protected
Mode (by the way, delete the infinite jump in *step2* as it will be now
made when we are in protected mode).

Inside of the *load~protected~* label, disable the interrupts using the
/ instruction and load the GDT using the lgdt keyword like this:

``` asm
.load_protected:
    cli
    lgdt[gdt_descriptor]
```

Now, we have to enable the protection bit in the cr0 register, to do,
add the following code to your *load~protected~* label:

``` asm
mov eax, cr0
or eax, 1
mov cr0, eax
```

Once you\'ve added that code, let\'s create another label right before
our boot signature and zero-padding, so there will jump our code once it
is in 32-bit protected mode, I\'ll call it load32

``` asm
[BITS 32]
load32:
```

**Note**: It\'s important to add *\[BITS 32\]* so all code underneath it
is seen as 32-bit code.

And inside that label, just add an infinite jump for now.

Now, at the top of the file (right after BITS 16) let\'s create two
constants, one called CODE~SEG~ and another one called DATA~SEG~, they
will contain the value these two segments should point to.

``` asm
CODE_SEG equ gdt_code - gdt_start
DATA_SEG equ gdt_data - gdt_start
```

And once we have those constants, go back to your *load~protected~*
label and add a jump to load32 specifying the absolute address of
*CODE~SEG~*:

``` asm
jmp CODE_SEG:load32
```

Now, you can *make* your project and *make run* to test it, if it is
stuck at \"Booting from Hard Disk\" it means everything went okay and
nothing crashed (if it didn\'t, compare your code with that in the git
repository provided at the end of the lesson).

If it didn\'t crash, that\'s a very good sign, but now we also have to
be sure it works fine, to test that let\'s use GDB. So, open up a
terminal and execute *gdb* and inside GDB execute *target remote \|
qemu-system-x86~64~ -hda boot.bin -S -gdb stdio*, that command will fire
up a qemu instance but it will be stoped (your bootloader won\'t be
executed immediately) and the control will be back to the GDB shell,
once you are back in the GDB shell press \"c\" to continue with the
execution of the program and once your bootloader is running, wait a
little and press CTRL+C in the terminal GDB is running, once you can
type again in GDB execute *layout asm* and you should be in the infinite
jump that is located in the *load32* label.

![](/img/guides/kernel/protected_gdb_asm.png)

If you are in that infinite jump, it means you are now in protected
mode. Now that we are in protected mode, let\'s setup our registers as
well. You can exit now QEMU and type \"quit\" in GDB to close it and go
back to your boot.asm file. To set the registers just go to the *load32*
label and add the following piece of code before the infinite jump:

``` asm
mov ax, DATA_SEG
mov ds, ax
mov es, ax
mov fs, ax
mov gs, ax
mov ss, ax
mov ebp, 0x00200000
mov esp, ebp
```

Now that we are in protected mode, our kernel is running 32-bit code and
we cannot longer access the BIOS, if we attempt to do it, bad and weird
things are going to happen, so don\'t try it :p.

You can check this entry\'s changes
[here](https://codeberg.org/QuadWord/Kinl/commit/5269ed2ee8a152c4ce05e5c357cf15a0a71beeb9)
