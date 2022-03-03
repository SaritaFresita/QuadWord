---
title: Loading Kernel and Enabling A20 line - Kernel_Development - Part 9
date: 2022-02-02T18:22:00+00:00
draft: false

image: /img/thumbs/kernel.png

description: "Let's load our kernel into memory and let's also enable the A20 line"

categories:
  - Kernel_Development
tags:
  - Programming
  - Assembly
  - Low-level
  - Operating systems
  - Computer_Science

type: post
---

Before we continue writing code, let\'s first organise our project.

## Organising our Project

I want our kernel\'s tree source directory to follow a
FreeBSD/Linux-like style, as I think it is pretty clean and the way it
is structured makes it very easy to work around with.

So first, let\'s create two folders \"arch/i386\" to store our i386
exclusive architecture files, and there we will store our *boot.asm*
file.

``` bash
mkdir -p arch/i386
mv boot.asm arch/i386
```

Now, let\'s update our Makefile. Delete all the contents of it and
let\'s rewrite it.

``` makefile
CROSS_PATH=$(HOME)/opt/cross/bin
CC=$(CROSS_PATH)/i686-elf-gcc
LD=$(CROSS_PATH)/i686-elf-ld

ASM=nasm
ECHO=echo -e
QEMU=qemu-system-i386

OBJS=

all: arch/i386/boot.bin $(OBJS)

arch/i386/boot.bin: arch/i386/boot.asm
    @$(ECHO) "ASM\t\t"$<
    @$(ASM) -f bin $< -o $@

run: all
    @$(QEMU) -hda arch/i386/boot.bin

clean:
    @rm -Rf $(OBJS)

%.asm: ; # This is to fix a circular dependecy bug :p
```

As you can see our Makefile is pretty much different now, there are
several variables at the top \"CROSS~PATH~\", \"LD\", \"CC\" \"ASM\",
\"ECHO\", \"QEMU\" and \"OBJS\".

\"CROSS~PATH~\" is the path where our cross-compiler binaries are stored
in, *CC* is the executable of our cross-compiler and *LD* the linker we
will use to link our kernel together, ASM will be used for the assembler
- all these variables are declared in a way, they could be changed at
runtime for any reason -, ECHO will be used for printing messages, QEMU
will be used for running our kernel and OBJS represents all the object
files that have to be created to compile completely our kernel.

There\'s nothing yet in the *all* label, but there will later be stuff
when we start using C and a linker to align correctly our data.

The *run* label, as it did before runs our kernel in QEMU and finally,
the *clean* label deletes all the compiled object files.

## Separating the bootloader and the kernel code

Before we start working in loading our operating system into memory,
let\'s first move some of the 32-bit code in the *boot.asm* file to an
own assembly file that will be linked with the rest of objects that are
produced when compiling our kernel.

Once we have that big file that contains all the objects linked
together, then it will be dd\'d with our bootloader to occupy the rest
of sectors and *boot.asm* will load them into memory later.

So, create a new file called *kernel.asm* in the *sys* folder and there
just move the *BITS 32* instruction and all the *load32* label
(including the infinite jump).

Now, in *boot.asm* we cannot reference *load32* because our boot.asm
file and kernel.asm are not linked together. What we will do to load our
32 bit code kernel, is to load it into a memory address and then we\'ll
jump to it. For now, comment out the *jmp CODE~SEG~:load32* line in the
*.load~protected~* label.

Now, let\'s make a linker script because soon we will start using C and
Assembly together, and this script will describe how it should link our
kernel, where things should be placed, etc.

So create a new file called *linker.ld* in the root directory of your
kernel:

``` text
ENTRY (_start)
OUTPUT_FORMAT(binary)
SECTIONS
{
  . = 1M;
  .text :
  {
    *(.text)
  }

  .rodata :
  {
    *(.rodata)
  }

  .data :
  {
    *(.data)
  }

  .bss :
  {
    *(COMMON)
    *(.bss)
  }
}
```

Don\'t worry that much if you don\'t understand what any of that means,
we will just write that file once for the entire project.

The *. = 1M;* instruction says that, when we link our kernel object
files using that script, it will set their origin at 1 megabyte in
memory, that is, 0x100000, so that\'s the memory address we\'ll be using
when loading our kernel in memory.

Now, go to your *Makefile* and add the following rule:

``` makefile
sys/kernel.asm.o: sys/kernel.asm
        @$(ECHO) "ASM\t\t"$<
        @$(ASM) -f elf -g $< -o $@
```

Don\'t forget to append *sys/kernel.asm.o* to the *OBJS* variable. And
now, let\'s make the *all* label create a new file called \"os.out\"
which will be our operating system executable.

``` makefile
all: arch/i386/boot.bin $(OBJS)
        dd if=arch/i386/boot.bin > arch/i386/os.out
```

As we haven\'t linked our kernel files, we cannot append a kernel file
to it, but we will do it once we have it. Now, if you compile it, you
will see an error in the *kernel.asm* file, we are referencing constants
that do not exist in that file, to fix it, just add this at the
beginning of the file (after \[BITS 32\]):

``` as
CODE_SEG equ 0x08
DATA_SEG equ 0x10
```

If you do *make* again, you\'ll see that there\'s now a file called
*os.out* in the *arch/i386* folder. Note that you will have to add that
file to the ones removed by the *clean* label and also replace it with
the file that *run* executes, your Makefile should look like this now:

``` makefile
CROSS_PATH=$(HOME)/opt/cross/bin
CC=$(CROSS_PATH)/i686-elf-gcc

ASM=nasm
ECHO=echo
QEMU=qemu-system-i386

OBJS=sys/kernel.asm.o

all: arch/i386/boot.bin $(OBJS)
    dd if=arch/i386/boot.bin > arch/i386/os.out

arch/i386/boot.bin: arch/i386/boot.asm
    @$(ECHO) "ASM\t\t"$<
    @$(ASM) -f bin $< -o $@

sys/kernel.asm.o: sys/kernel.asm
    @$(ECHO) "ASM\t\t"$<
    @$(ASM) -f elf -g $< -o $@

run: all
    @$(QEMU) -hda arch/i386/os.out

clean:
    @rm -Rf arch/i386/boot.bin arch/i386/os.out $(OBJS)

%.asm: ;
```

Now, we need to add another rule to compile a *kernel.out* file which
will contain all the objects files of our kernel linked together, that
rule looks like this:

``` makefile
sys/kernel.out: $(OBJS)
    @$(LD) -g -relocatable $(OBJS) -o sys/kernelfull.o
    @$(CC) -T linker.ld -o $@ -ffreestanding -O0 -nostdlib sys/kernelfull.o
```

Once you add that rule, you can delete the *OBJS* dependency in *all*
and instead, add *sys/kernel.out*.

If you now do *make* your project, there will be a warning because we
did not define a *~start~* label, let\'s fix that problem. Go to your
*kernel.asm* file and rename *load32* to *~start~* and also, at the
beginning of the file (after the BITS 32 instruction), define it as a
global label:

``` asm
global _start
```

And now that error should be gone.

Now that we have two binary files, we have to add it to our final
*kinl.bin* file, to so just add the following commands in the *all* rule
of your Makefile:

``` makefile
@dd if=sys/kernel.out >> kinl.bin
@dd if=/dev/zero bs=512 count=100 >> kinl.bin
```

What those commands do is to add the *sys/kernel.out* file to kinl.bin
and also, it appends 100 sectors of zeros to that file, if you don\'t
want to use magic numbers, you can create another variable in your
Makefile like \"NO~SECTORS~\" (number of sectors) and replace it in the
last dd command.

**Note**: Don\'t forget to update the file name in the *run* rule of
your Makefile.

## Enabling A20 line

We need to enable the A20 line before we continue with our kernel, or
there might be problems later, if we don\'t enable it, we won\'t have
access to the 21st bit of any memory access.

Enabling the A20 line is very easy, we just have to read from port 0x92,
then we set a bit in the register and we write it back to port 0x92.

To enable A20 line, just add the following piece of code after we
initialised the registers in the *~start~* label:

``` asm
in al, 0x92
or al, 2
out 0x92, al
```

If you haven\'t seen the *in* or *out* instructions before, what they
basically do, is just read or write to the processor bus.

Once you\'ve enabled A20 line, *make* your project and run it again in
QEMU to check everything has worked successfully.

## Loading the Kernel into Memory

Now that our kernel source is getting bigger, we need to write a simple
disk driver to load it into memory, it will load the number of sectors
written in kinl.bin into memory address 0x100000.

Let\'s get started by uncomming the *jmp CODE~SEG~:load32* line in the
boot.asm file and create again the *load32* label at the end of the file
(before the boot signature).

``` asm
    [BITS 32]
load32:
```

We will use that label to load our kernel into memory, but as we are in
protected mode, we cannot use the BIOS so we will write a simple ATA LBA
driver, in that label add the following code:

``` asm
mov eax, 1
mov ecx, 100
mov edi, 0x0100000
call ata_lba_read
jmp CODE_SEG:0x0100000
```

Those are the parameters of a function we haven\'t created yet called
*ata~lbaread~*, eax is the LBA, ecx is the number of sectors you want to
load into memory, in our case 100, the edi register is the address in
memory where our kernel will be loaded and finally, we are calling that
function and jumping to the 0x0100000 address.

This is the function *ata~lbaread~* (I won\'t explain it as we will use
it only once and when we do it in C I will explain it as good as I can -
I promise -):

``` asm
ata_lba_read:
    mov ebx, eax        ; Backup the LBA

    ;; Send the highest 8 bits of the LBA to the hard disk controller
    shr eax, 24
    or eax, 0xE0
    mov dx, 0x1F6
    out dx, al
    ;; Finished sending the highest 8 bits of the LBA

    ;; Send the total sectors to read
    mov eax, ecx
    mov dx, 0x1F2
    out dx, al
    ;; Finished sending the total sectors to read

;; Send more bits of the LBA
mov eax, ebx          ; Restore the backup LBA
mov dx, 0x1F3
out dx, al
;; Finished sending more bits of the LBA

;; Send more bits of the LBA
mov dx, 0x1F4
mov eax, ebx        ; Restore the backup LBA
shr eax, 8
out dx, al
;; Finished sending more bits of the LBA

;; Send upper 16 bits of the LBA
mov dx, 0x1F5
mov eax, ebx      ; Restore the backup LBA
shr eax, 16
out dx, al
;; Finished sending upper 16 bits of the LBA

mov dx, 0x1F7
mov al, 0x20
out dx, al

;; Read all sectors into memory
.next_sector:
push ecx

;; Checking if we need to read
.try_again:
mov dx, 0x1F7
in al, dx
test al, 8
jz .try_again

;; We need to read 256 words at a time
mov ecx, 256
mov dx, 0x1F0
rep insw
pop ecx
loop .next_sector
;; End of reading sectors into memory

ret
```

If you *make* your kernel again and you do *make run* you probably
won\'t see any differences but let\'s debug it with GDB now.

As we built our *sys/kernelfull.o* file with debugging symbols, let\'s
load those symbols to GDB, we have to load them into address 0x0100000
so they are aligned correctly, you can execute this command to load the
symbols:

``` bash
add-symbol-file sys/kernelfull.o 0x0100000
```

Press \'y\' and now attach GDB as we\'ve been doing it forever:

``` bash
target remote | qemu-system-i386 -hda kinl.bin -S -gdb stdio
```

And there, let\'s create a breakpoint in *~start~* as we know it would
get executed only if our operating system gets loaded successfully,
execute:

``` bash
break _start
```

And finally, press \'c\' to continue. You should see that the execution
stopped at the breakpoint we set, press \'c\' again and after a little
while press CTRL+C again, if you execute \"layout asm\" you should see
that we are in the infinite loop, that means our operating system got
loaded into memory and the A20 line was successfully enabled.

[You can see this entry's changes here](https://codeberg.org/QuadWord/Kinl/commit/4647b33e0c727475dfd0ab141cc8363702f3b332)
