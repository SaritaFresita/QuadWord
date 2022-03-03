---
title: Introduction - Kernel_Development - Part 1
date: 2022-02-02T18:06:00+00:00
draft: false

image: /img/thumbs/kernel.png

description: "The first entry of the kernel development series on QuadWord"

categories:
  - Kernel_Development
tags:
  - Low-level
  - Operating systems

type: featured
---

## Introduction

In this guide I will teach you all how to develop a functional kernel
from scratch that can be booted from real hardware and you will learn -
*i hope* - a lot about how computers work and what\'s the function of an
operating system.

Take into account that this is not a guide meant for beginners, kernel,
drivers, operating systems and compilers development are - *in my
opinion* - the hardest projects a programmer can write, if you are a
beginner, or if you already have experience but you feel it is not
enough, take a time to write simple Linux modules, write programs in
assembly, try to write a bootloader that outputs \"*Hello, World!*\" to
the screen, read about filesystems, try to write a driver reading the
manufacturer documentation and things like that, so when you want to
write your own operating system, it won\'t feel hard at all.

Throughout this guide, I won\'t explain too much simple C and assembly
things, for example, I won\'t explain the difference between the
\"data\" and \"bss\" sections, differences between 16-bit and 32-bit
assembly, what is a structure, what does uint32~t~ mean, what is
pass-by-reference, pass-by-value and what they are used for, as I said
before, this is not a guide for beginners.

I have planned to write a lot of things regarding kernels, at the
completion of this guide, you should have an operating system with the
following features:

-   Its own bootloader that can boot up from computers with Legacy BIOS
    and load the kernel in memory (reading it from the hard disk).

-   A 32-bit kernel running in protected mode.

-   A graphical kernel that can draw shapes in the screen with a
    resolution of 1024x768 (or a different resolution of your choice).

-   A kernel with its own interrupts.

-   Heap.

-   Paging.

-   A hard disk driver.

-   A FAT16 filesystem implementation.

-   Userland.

-   TSS.

-   Processes.

-   Mouse and keyboard driver.

-   A kernel capable of running ELF files.

-   A kernel with its own C standard library.

-   A shell.

-   Multitasking.

    I think that\'s it. Kernel development is such a broad topic,
    therefore, it is not necessary for you to know and understand all
    the terms that were used in the features list, as they all will be
    studied deeply in the following entries.

    For now, let\'s prepare our kernel development environment.

## Build a GCC Cross Compiler

**Note**: This is taken from
[osdev.org](https://wiki.osdev.org/GCC_Cross-Compiler).

Let\'s create a cross-compiler for the kernel we will develop. This will
compile to a generic target (i686-elf) that will let you leave your host
os behind, that is, no header nor library in your computer will be used.
You\'ll need a cross-compiler, otherwise, several unexpected things
might happen because the compiler will assume you are running that
program in your computer.

### Compiling a Cross-Compiler

Before compiling your cross-compiler, you\'ll need to install the
following dependencies:

```bash
# Debian
sudo apt install build-essential bison flex libgmp3-dev libmpc-dev \
 libmpfr-dev texinfo libcloog-isl-dev libisl-dev

# Gentoo
sudo emerge --ask sys-devel/gcc sys-devel/make sys-devel/bison \
 sys-devel/flex dev-libs/gmp dev-libs/mpc dev-libs/mpfr \
 sys-apps/texinfo dev-libs/cloog dev-libs/isl

# Arch
sudo pacman -S base-devel gmp limpc mpfr base-devel
```

Now, you\'ll need to download the source code and store it in a
directory like \$HOME/src.

Before we start with the compiling process, let\'s first export some
variables that we will use later:

```bash
export PREFIX="$HOME/opt/cross"
export TARGET=i686-elf
export PATH="$PREFIX/bin:$PATH"
```

### Binutils

In order to compile binutils, run the following script:

```bash
cd $HOME/src

wget https://ftp.gnu.org/gnu/binutils/binutils-2.37.tar.xz
tar xvf binutils-2.37.tar.xz
mkdir build_binutils
cd build_binutils
../binutils-2.37/configure --target=$TARGET --prefix="$PREFIX" \
           --with-sysroot --disable-nls --disable-werror
make
make install
```

### GCC

To compile GCC, run the following script:

```bash
cd $HOME/src

wget https://ftp.gnu.org/gnu/gcc/gcc-11.2.0/gcc-11.2.0.tar.xz
tar xvf gcc-11.2.0.tar.xz

# This command should return $HOME/opt/cross/bin/i686-elf-as
which -- $TARGET-as || echo $TARGET-as is not in the PATH

mkdir build_gcc
cd build_gcc
../gcc-11.2.0/configure --target=$TARGET --prefix="$PREFIX" --disable-nls \
            --enable-languages=c,c++ --without-headers
make all-gcc
make all-target-libgcc
make install-gcc
make install-target-libgcc
```

### Using our Cross-Compiler

Once we are done compiling binutils and GCC, we can use it now to build
our future operating system. You can use this compiler, by running a
command like:

```bash
$HOME/opt/cross/bin/$TARGET-gcc --version
```

To run this compiler by just typing \$TARGET-gcc in the terminal, you
can add the \"bin\" directory to the path:

```bash
export PATH="$HOME/opt/cross/bin:$PATH"
```

That was it for this entry, in the next ones we will learn more and more
about kernels and we will build our own and we will also improve it with
time.

I hope the feature list motivated you all, for sure I missed some
features, as long as we keep developing this kernel, I will remember
them and we will do much more things.

Thanks for reading :3
