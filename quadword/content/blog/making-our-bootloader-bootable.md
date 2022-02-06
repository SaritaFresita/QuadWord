---
title: Making our Bootloader Bootable - Kernel Development - Part 5
date: 2022-02-02T18:16:00+00:00
draft: false

image: /img/thumbs/kernel.png

description: "Let's fix some errors before we can boot our bootloader on actual hardware"

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

## Making our Bootloader Bootable

Now, let\'s improve our bootloader to fix some errors and make it
bootable on an actual machine!

What we will do is, set up segment registers and change the origin of
our bootloader. When the BIOS loads our bootloader, we don\'t know what
the segment registers are, because of this, having a bootloader as the
one we have right now, does not guarantee it will successfully boot on
most machines, for example, if the BIOS sets our data segment to 0x7c0
and our program\'s origin is 0x7c00, then the equation would be *ds \*
16 + 0x7c00* so, if ds is 0x7c0 we\'d end up with 0x7c00 + 0x7c00 which
does not point to our message variable.

Because of these types of scenarios, we should initialise the data
segment and all the other segments ourselves, let\'s do it now.

First, let\'s change our program\'s origin to 0, to do so, just change
the value of the *ORG* instruction:

``` asm
ORG 0
```

What we will need to do now, is go to our *start* label and at the
beginning of it, add the following instructions:

``` asm
cli   ; Disable interrupts

sti ; Enable interrupts
```

As the comments explains, the **cli** instruction disables all the
interupts and **sti** enables them all again. The reason we disable our
interrupts is that we will change some segment registers and we don\'t
want any interrupt to happen as the system would panic or there might be
unexpected responses as some segments wouldn\'t be set correctly. The
following piece of code is in between the cli and the sti instruction.

``` asm
mov ax, 0x7c0
mov ds, ax
mov es, ax
```

We are setting the value of *ax* to 0x7c0 and then we are changing the
ds and es segment values to the one in *ax*. We cannot just move
*0x7c00* to either the data or extra segment, that\'s why we first move
it to ax, that\'s how the processor works.

As we have already set up the data segment, the extra segment and our
origin is zero. When we reference our message label, the processor will
assume that we are loaded into address 0x00 into RAM, so its offset will
be pretty low, it will be where in our binary file the message label is
stored. Let\'s assume that this offset is 14 bytes so, when we call
*lodsb* what will happen is, it will use our data segment and the si
register, as we already know we\'ve changed our data segment to 0x7c0,
it will multiply its value by 16 and its result would be 0x7c00 and
then, it will add the offset of our message to that address, *0x7c00 +
14 = 0x7c0e* which would be correct. That\'s why we need to change these
data segments, because if the BIOS sets them for us, it could mean our
origin is set wrong for our program and then it won\'t link up
correctly.

The next segment we want to initialise is the stack segment, we will set
this up differently as we know it grows downwards. So, what we can do is
set the stack pointer equal to 0x7c00 and it will start growing down.
Therefore, we will do:

``` asm
mov ax, 0x00
mov ss, ax
mov sp, 0x7c00
```

What we did, is to set up the stack segment to zero and the stack
pointer to 0x7c00.

The last thing we have to do is to add:

``` asm
jmp 0x7c0:start
```

After the *BITS 16* instruction, so our code segment also becomes 0x7c0.

That should be it, if you assemble your bootloader again and run it with
qemu, it should work as it did before, but now, we are in control of the
situation, we don\'t rely on the BIOS to set everything up for us
anymore.

## Bios Parameter Block

Lastly, we have to do something before our bootloader is completely
bootable from actual hardware, we have to implement a Bios Paramter
Block (BPB) for short.

![](/img/guides/kernel/bootloader3.png)

*(BPB taken from
[osdev](https://wiki.osdev.org/FAT#BPB_.28BIOS_Parameter_Block.29))*

Our bootloader, as we currently have it, will boot fine in some
computers, but in others there might be problems as the BIOS tampers
your data when booting from an USB stick, which is probably the way we
all will test our operating system. The reason this happens is because
of the BPB, some BIOS expect it, those who expect a BPB will corrupt our
data.

The data is corrupted because when we boot from a USB stick, it is doing
something called USB emulation, the BIOS is treating our USB stick as
hard drive and allowing us to talk to it as such. You don\'t need to
know much about the BPB for now, we just need to know that some BIOS
assume it\'s there and will start writing data and overwriting your
code.

We can get around this problem by implementing the BPB, we don\'t need
to have real values, it can be all zeros, we will create a fake BPB to
get around this problem.

The first thing we want have to do is to add up the size of the BPB,
except the first three bytes (because they are a short jump and a nop
and some BIOSes will look for this as well, we will actually write these
three bytes, and we will fill the other ones with zeros).

To do this, go to the start of your code (before *jmp 0x7x0:start*) and
create another label, like this:

``` asm
_start:
```

And what we will add to that label is:

``` asm
jmp short start
nop
```

We are doing the short jump and the nop of the first three bytes
(**Note**: nop means \"No Operation\").

Now we have to create another label under start so we can do a jump to
that new label, so it sets the code segment at 0x7c0, just modify the
beginning of your start label so it looks like this:

``` asm
start:
jmp 0x7c0:step2

step2:
;; Rest of the code here
```

Now, we can put our fake BPB in between *~start~* and *start*. After the
*nop* instruction, let\'s add 33 zeros (that\'s the added num of bytes
occupied by the BPB):

``` asm
times 33 db 0
```

Now you should be able to assemble and boot your bootloader without the
risk of your code being overwritten.

[You can see this lesson\'s first part commit
here](https://codeberg.org/QuadWord/Kinl/commit/46296ae7dfe069a277510f60b5af80161abf6244)

[You can see this lesson\'s BPB part commit
here](https://codeberg.org/QuadWord/Kinl/commit/990fc2a9b1a382ade30902198a76b890fb119d83)
