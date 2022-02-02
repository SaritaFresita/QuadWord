---
title: Real Mode - Kernel Development - Part 3
date: 2022-02-02T18:11:00+00:00
draft: false

image: /img/kernel.png

description: "Learn what Real Mode is, and why it's important to know when developing a kernel"

categories:
  - Kernel Development
tags:
  - Low-level
  - Operating systems
  - Computer Science

type: post
---

## Real Mode

In previous entries, I've explained briefly that 'Real Mode' is a
compatibility mode that all modern intel processes have, and this is the
mode in which they start, it mimics the processor from several years
ago.

In Real Mode we can only access 1 MiB of RAM, it doesn't matter if you
have four or more gigabytes plugged into your computer, only 1 MiB will
be accessible, also, memory is accessed through the segmentation memory
model, simply put, memory can be accessed through the use of segments
and offsets.

Real Mode is based on the original x86 design, it acts like older intel
processors from the 70s (such as the 8086) and we also have the same
limitations as these. All the code we write while in real mode, is
required to be written in 16 bits.

When the processor is in Real Mode, there is no memory nor hardware
security, which means, programs that are run could destroy our computer
with no way for us to stop them, the reason why in modern operating
systems a process can't just destroy your computer is that the kernel
tells the processor (this is made in the processor itself) which
programs cannot access certain parts of memory or directly the hardware.

And finally, when we are in Real Mode, only 8-bit and 16-bit registers
are accessible, which means, we can only request memory address offsets
up to 65535 for our given segment, there can't be decimal numbers larger
than 65535.

## Segmentation Memory Model

Memory segmentation is an operating system memory management technique
of division of a computer's primary main memory into segments or
sections. In a computer system using segmentation, a reference to a
memory location includes a value that identifies a segment and an offset
(memory location) within that segment. Segments or sections are also
used in object files of compiled programs when they are linked together
into a program image and when the image is loaded into memory.

Segments usually correspond to natural divisions of a program such as
individual routines or data tables so segmentation is generally more
visible to the programmer than paging alone. Different segments may be
created for different program modules, or for different classes of
memory usage such as code and data segments. Certain segments may be
shared between programs.

Segmentation was originally invented as a method by which system
software could isolate different tasks and data they are using. It was
intended to increase reliability of the systems running multiple
processes simultaneously. In a x86-64 architecture, it is considered
legacy and most x86-64 based modern system software don't use memory
segmentation. Instead they handle programs and their data by utilising
memory paging which also serves as a way of memory protection. However
most x86-64 implementations still support it for backward compatibility
reasons.

Source [Wikipedia](https://en.wikipedia.org/wiki/Memory_segmentation).
