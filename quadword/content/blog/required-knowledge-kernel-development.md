---
title: Required Knowledge - Kernel Development - Part 2
date: 2022-02-02T18:09:00+00:00
draft: false

image: /img/kernel.png

description: "Let's talk a little about the required knowledge to develop a kernel"

categories:
  - Kernel Development
tags:
  - Low-level
  - Operating systems
  - Computer Science

type: post
---

## Required Knowledge

In this entry we will learn about two basic things that are important
when developing kernels, we\'ll see what memory is and what does a
computer do when booting.

## Memory

Memory is a piece of hardware that allows computers to store
information, RAM (Random Access Memory) is the main memory of your
computer, programs can store and read information there.

ROM memory (Read-Only Memory) is a form of memory that allows to only
read its information, it cannot be modified.

Something very important to know about memory, is that, information is
accessed in a linear way, for example, if you tell a program \"read the
information in address 0xff781a\", the program will read the information
that is stored in 0xff781a, if you ask the program to read what\'s
stored in address 0x0af9, it will read what\'s stored there, information
is written in a linear way as well.

Even though memory is read from and written to in a linear way, when the
processor access it and reads what\'s stored there, it will do it in a
different way as the processor has different memory scheme, but, when we
read directly from the memory, we will read in a linear way.

### RAM Memory

In RAM you can write and read information, but RAM can only be used as
temporary storage, once your computer shuts down, its content will be
flushed, it will be gone.

### ROM memory

In ROM memory, you cannot write information - *through conventional
manners* -, but this is permanent, that is, even if you turn off your
computer there will be the information.

This kind of memory, is generally used to store permanent programs that
will never change, for example, embedded devices or your computer\'s
BIOS.

## Boot process

Let\'s take a look at the steps your computer take before executing your
operating system:

-   First, BIOS is executed from ROM memory in which it is stored.
-   Then, the BIOS will look for an operating system in all the storage
    devices connected to the PC, and once it finds it, it will load it
    in memory address 0x7c00.
-   The bootloader of the operating system loads the kernel into memory
    and then, it is executed.

### What is a bootloader?

A bootloader is a small program, that loads the kernel of an operating
system into memory.

When the computer boots, the processor will be in a compatibility state
called \"Real Mode\" and this gives us access to 1 MiB of memory and
also, executes only 16-bit code. In this mode, we will be fairly
limited, but then our bootloader will switch to a mode called
\"Protected Mode\" and this one will give us access to 4 GiB of memory
and we will be executing 32-bit code.

### Regarding BIOS...

BIOS is executed directly from ROM, therefore, once we turn on the
computer, the BIOS will load itself into memory - so it is faster - and
it is executed then.

BIOS is also in charge of initialise important hardware, this will
provice functions to communicate with the hard disks, print text to the
screen, draw graphics, etc.

When the BIOS is finished initialising the computer\'s hardwarwe, it
will look in all the connected hard disks that the last 2 bytes of the
first sector are 0x55AA, once it finds these bytes in a disk, it means
it has an operating system and BIOS will load it to memory address
0x7c00.

Something that is pretty important to know, is that BIOS is almost a
kernel itself, it has several routines that will help our bootloader to
execute our kernel, it executes 16-bit code and these routines are
generic and part of a standard.
