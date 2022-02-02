---
title: Using C Code - Kernel Development - Part 10
date: 2022-02-02T18:24:00+00:00
draft: false

image: /img/kernel.png

description: "We don't want to write our kernel only in Assembly, right? let's use C instead"

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

## Cleaning Objects

Before we continue, let's first update the _clean_ rule in our Makefile so it
cleans all the objects properly, as more objects are generated we have to add
them to the objects to clean.

```make
clean:
	@rm -Rf arch/i386/boot.bin kinl.bin sys/kernel.out sys/kernelfull.o $(OBJS)
```

That's how our clean rule should look for now.

## Aligning C and Assembly Code

As we specified in our linker script, we want our kernel to be linked as a
binary file, so far, this is completely file, but when we start using C and
Assembly, it can cause alignment problems. C requires functions to be aligned in
a certain way, when we are writing Assembly and we link it with object files
created with a C compiler our Assembly code might not align correctly and it
means that all the functions that were compiled assume that it will always be
aligned, but we didn't do it in our assembly, there can be problems such as call
instructions because of the alignments.

Why does it happen? Basically, because our assembly shares the same section as
our object files created with the C compiler, you could change the alignment of
the C compiler, but to keep this simpler, let's use the compiler's default and
let's create a new section for our assembly files.

To create a new section, go to your _linker.ld_ file and add the following piece
of code at the bottom (after we define the bss section). We do it at the bottom
so the C code in the text section will never have the alignment damaged because
of our assembly code:

```ld-script
.asm :
{
	*(.asm)
}
```

Once we have created the new section, we have to make sure all our sections are
16-bit aligned, we will align them to 4096 (as 4096 % 16 = 0, it will be very
useful when we start working with pages later), add `ALIGN (4096)` after every
section, the final code should look like this:

```text
ENTRY (_start)
OUTPUT_FORMAT(binary)
SECTIONS
{
	. = 1M;
	.text : ALIGN (4096)
	{
		*(.text)
	}

	.rodata : ALIGN (4096)
	{
		*(.rodata)
	}

	.data : ALIGN (4096)
	{
		*(.data)
	}

	.bss : ALIGN (4096)
	{
		*(COMMON)
		*(.bss)
	}

	.asm :
	{
		*(.asm)
	}
}
```

Note that we won't set the section of our _kernel.asm_ file to be the one we
just created (.asm) as it needs to be the first object linked in our kernel, so
it is executed when we jump to 0x0100000 (otherwise, another object would get
executed and that's not the idea).

Another thing we have to do, is to pad our _kernel.asm_ file with 512 bytes of
zeros, why? As we won't set this file to be aligned in the .asm section, it
might break the alignment of a C file, for example, as it will be always first,
and as 512 % 16 = 0, it means that this file will be perfectly aligned. Add this
line of code at the end of the _kernel.asm_ file:

```asm
times 512-($-$$) db 0
```

**Note**: When you want to align an Assembly file in the asm section, you just
have to add `section .asm` at the beginning of it.

## Mixing C and Assembly

Let's start using C for developing our kernel, because let's be honest, it would
be a complete pain if we try to do it using only Assembly.

Let's get started by creating a `kernel.c` file in the _sys_ folder and in that
file you will only have to add two things: an include and a void function. We
will create a `kernel.h` as well, where we will store some structures, and
definitions and some other functions such as `panic`, that's why we need to
create it, and the void function could be our C entry point, as it will get
executed first, the file then looks like this:

```c
#include <kernel.h>

void
kmain ()
{
}
```

Note that we are doing `<kernel.h>` that's because of the flags we will set
later on when compiling our code.

And in the `kernel.h` file, only add header guards and the declaration of our
`kmain` function, that file looks like this:

```c
#ifndef __KERNEL_H
#define __KERNEL_H

void kmain ();

#endif
```

And now, we have to tell to the Makefile to compile this C code and link it with
our other objects as well, to do so, first create a new variable in it called
`CFLAGS`, this variable will contain all the flags we will set when compiling
our C files. This variable looks like this:

```make
CFLAGS=-g -ffreestanding -falign-jumps -falign-functions -falign-labels \
	-falign-loops -fstrength-reduce -fomit-frame-pointer \
	-finline-functions -Wno-unused-function -fno-builtin -Werror \
	-Wno-unused-label -Wno-cpp -Wno-unused-parameter -nostdlib \
	-nostartfiles -nodefaultlibs -Wall -O0
```

It is split into several lines, so they are less than 80 characters long (for
good practise).

With those flags we are telling the compiler not to look for an `int main`, to
align jumps, functions labels, loops, we are telling it to give us some
additional warnings and to turn them into errors, we are telling it not to link
with an standard library, we are enabling debugging symbols, and finally we are
disabling optimisations (so it is easier to debug for us).

Now, let's create another variable - right after CFLAGS - called `INCLUDES`, which
will define all the include flags and we will use them when compiling. We are
making it a different variable, as there might be several includes in the
future. For now, we'll just add the `sys` folder:

```make
INCLUDES=-Isys
```

Now, what we have to do, is to add a new object file in the `OBJS` variable,
this will be our `kernel.c` object. Take into account that we MUST add it after
`kernel.asm.o`, as it should be the first object as explained before. This
variable now should look like this:

```make
OBJS=sys/kernel.asm.o sys/kernel.o
```

And now, we should add a new rule after `sys/kernel.asm.o` to compile our C
file. This rule looks like this:

```make
sys/kernel.o: sys/kernel.c
	@$(ECHO) "CC\t\t"$<
	@$(CC) $(FLAGS) $(INCLUDES) -std=gnu99 -c $< -o $@
```

That rule will ensure our C file gets compiled and our sys/kernel.out rule, will
ensure they get linked properly, there's one last thing we must do, and we must
add the `FLAGS` variable to that rule where we use our cross compiler, that rule
should look like this:

```make
sys/kernel.out: $(OBJS)
	@$(LD) -g -relocatable $(OBJS) -o sys/kernelfull.o
	@$(CC) $(FLAGS) -T linker.ld -o $@ -ffreestanding -O0 -nostdlib sys/kernelfull.o
```

Now, let's call our C function, in our `kernel.asm` file. To do so, just declare
a label as extern after the bits instruction like this:

```asm
extern kmain
```

And after we enable A20 line, we must call the `kmain` function:

```asm
call kmain
```

Now if you compile your project, you run it with GDB:

```bash
add-symbol-file 0x0100000
target remote | qemu-system-i386 -hda kinl.bin -S -gdb stdio
```

And if we set a breakpoint at our `kmain` function:

```bash
break kmain
```

When we continue the execution of our program (by entering `c`), you will see
that this breakpoint will get reached, that means, we have successfully mixed C
and Assembly code.

[You can see this entry's changes here](https://codeberg.org/QuadWord/Kinl/commit/c6e62ba97fd59c7edd7489e0d8acf08056f01ad7)
