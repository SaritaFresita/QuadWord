---
title: Interrupt Vector Table - Kernel Development - Part 6
date: 2022-02-02T18:17:00+00:00
draft: false

image: /img/kernel.png

description: "Let's learn what interrupts are and how we can create our own"

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


## What are Interrupts?

Interrupts are something like subroutines, but you don\'t need to know
their memory address to invoke them, you call them through the use of
interrupt numbers such as 1, 2, and 3, rather than memory addresses,
interrupts can be set up by the programmer, for example, you could set
the interrupt 0x32 to point somewhere in your code, therefore, when
someone does *int 0x32* it will invoke the defined interrupt.

### What happens when an Interrupt is invoked?

First, the processor gets interrupted, the old state is pushed to the
stack, saving it - in includes things like the return address - and
after that, the interrupt is executed.

## Interrupt Vector Table {#interrupt-vector-table-1}

The Interrupt Vector Table (IVT) is a table describing where the
interrupts are located in memory, we have 256 interrupt handlers and
each of those entries in the table is 4 bytes, the first two bytes
represent the offset in memory, and the next two bytes represent the
segment. Interrupts are also in numerical order in the table.

The Interrupt Vector Table starts at absolute address 0 in RAM, it\'s
the first byte in memory. The first four bytes describe interrupt 0, the
next four bytes describe interrupt 1, the next four interrupt 2 and so
on until interrupt 256. The following is a fictitious IVT:

| Offset | Segment |
| ------ | ------- |
| 0x0000 | 0x7c00  |
| 0x0000 | 0x7f00  |
| 0x0000 | 0x7c10  |
| 0x0000 | 0x7c80  |

As you can imagine, each of these interrupts are called by their number
(1, 2, 3 and 4, in order) and when the offset and the segment addresses
are added, it will return the absolute address of the routine that
executes this interrupt.

### Creating an Interrupt

Let\'s create our own interrupt.

In your boot.asm file, create a label like this:

``` asm
handle_int0:
iret
```

An interrupt can do anything you want, the one we are creating will
print an \'A\' character to the screen, so add the following code in the
*handle~int0~* label:

``` asm
handle_int0:
mov ah, 0eh
mov al, 'A'
mov bx, 0x00
int 0x10
iret
```

***Note**: We use the \'iret\' keyword at the end of any interrupt, as
it represents that\'s where the interrupt ends.*

What we have to do now is to change the IVT, so when we call the
interrupt 0 it will execute our *handle~int0~* interrupt.

With what I mentioned before, we know that the interrupt 0 starts at
absolute address 0x00 in RAM, so the first two bytes of RAM are the
offset for the interrupt, and the next two byets are the segment for the
interrupt 0, so we would only need to do this after we initialise the
registers and enable the interrupts in the *step2* routine:

``` asm
mov word [ss:0x00], handle_int0
mov word [ss:0x02], 0x7c0

int 0
```

**Note:** We use the stack segment (when doing *mov word
\[ss:hex~num~\]*), because if we don\'t tell the assembler to use the
stack segment, it will use the data segment which currently points at
0x7c0 which would be a problem, that\'s why we add *ss:* which means an
address in the stack segment. We could also change the data segment and
set it back with the value it should have, but this is just easier.

Now, if we assemble our program and we execute it, we would get an
output like the following:

![](/img/guides/kernel/ivt1.png)

### Interrupt 0

The Interrupt 0 is an exception when one number is divided by zero, so
what we can do now, is to divide a number by zero and our interrupt will
be called when we do that. You can divide by zero in Assembly doing:

``` asm
mov ax, 0x00
div ax
```

That piece of code will set the value of the *ax* register to zero and
then divide it by itself. Our interrupt will be called again.

You can get more information about the available exceptions
[here](https://wiki.osdev.org/Exceptions).

And this is this lesson\'s commit:
[here](https://codeberg.org/QuadWord/Kinl/commit/cdce8bb70654e33ada0e095a6a60b744d2063f58)
