---
layout: article
uuid: aee20b73-82c1-4a37-a788-edee98ff4fe8
title: omap davinci linux cmem ddralg ddr2 dsplink
categories: omap
updated_at: 2010-08-23
---
Goal
====

Explain the physical allocation of DSP / ARM memory on the OMAP / DaVinci

Overview
====
In order to get DSP applications to work, you have to allocate actual physical memory for use by

  * The Operating System (such as the Linux Kernel)
  * CMEM (shared contiguous memory between DSP and ARM)
  * DDRALGHEAP / DDR2 / DSPLink

Pre-req:
-----

You should know the base address of the RAM device. For the OMAP3530, for example, the RAM starts at `0x80000000`, which is `2048MB`, or `2GB`.

The address space for a 32-bit system is up to `0xFFFFFFFF`, which is `4096MB`, or `4GB`.

On a 32-bit desktop system (or a 64-bit system with a 32-bit memory bus) the theoretical limit for memory is 4GB, but in practice 3GB is the limitation, maybe 3.5 if you're lucky. This is due to the fact that RAM isn't the only device on the system. There are many other devices.

In the case of the OMAP3530, it would not be theoretically possible to have more than 1GB of memory since the first 2GB and last 1GB of address space are reserved for other devices. The most RAM on an OMAP3530 kit right now is 512MB. I'm guessing that RAM + NAND is probably the 1GB.

OS Memory
====

Linux (the ARM OS)
------

With the default system I believe that `120MB` of memory is allocated to linux.

You can find out how much OS memory is currently allocated like so:

    free -m
    # or
    cat /dev/meminfo

Since the kernel takes some memory unto itself, you're likely to see less than what _should_ be physically available.

If you would like to modify this to be another amount, say `160MB` you should add the parameter `MEM=120M` to the u-boot `bootargs`.

TI's BIOS (the DSP OS)
-------

`BIOS` refers to TI's `B? I? Operating System` - similar to `Linux` or `Windows`, but without a user interface.
It schedules, manages memory, runs applications, and even can use device drivers.

CMEM Memory
====

[TI wiki CMEM Overview](http://processors.wiki.ti.com/index.php?title=CMEM_Overview)

Just like disk fragmentation occurs on a filesystem, memory fragmentation occurs in `virtual memory` in the `heap`.

For example, if you allocate 40MB of memory dynamically, the virtual address space will always be contiguous, 
but the physical address space could potentially be split up into blocks of 20MB, 10MB, and 10MB.
As the OS runs for longer periods of time and more apps request and release various memory sizes, the risk of fragmentation becomes greater.

This is introduces latency, as the processor has to spend extra clock cycles switching back and forth between physical memory may be hundreds of megs in address space apart.

`CMEM` allocates memory which is guaranteed to be contiguous, not fragmented.
If you allocate 8MB, you get 8MB of physical memory. Period.

This memory is shared between the DSP and the ARM.

DDRALGOHEAP
====

[TI wiki DDRALGHEAP](http://processors.wiki.ti.com/index.php?title=DDRALGHEAP)

This is codec memory, as allocated by `BIOS`. This should be the size

DDR2
====

This is `BIOS` memory.

Setting memory sizes
====

TODO

These are set in `.tcf` and `.tci` files. The [TI wiki](http://processors.wiki.ti.com/index.php/DDRALGHEAP#DDRALGHEAP_Limitations) suggests that you don't need to know the base address - that XDC will calculate that based on the desired size - but I've only made changes by calculating everything myself.

Resources
=====

[Pixhawk Memory Map Tutorial](http://pixhawk.ethz.ch/tutorials/omap/dsplink/memorymap)
[TI wiki DDRALGHEAP](http://processors.wiki.ti.com/index.php?title=DDRALGHEAP)
[TI wiki CMEM Overview](http://processors.wiki.ti.com/index.php?title=CMEM_Overview)
[UUG Phsyical vs Virtual Memory allocation and fragmentation](http://uug.byu.edu/pipermail/uug-list/2010-August/004036.html)
[LWN Avoiding - and fixing - memory fragmentation](http://lwn.net/Articles/211505/)

appendix - mailing list discussion
------
> On the OMAP3530 each core runs it's own OS.
>
> The ARM runs Linux.
>
> The DSP runs a special TI OS with the horrible name BIOS (also called DSPBIOS).
> I think TIOS would have been about 1000x better, to avoid confusion for
> people skimming the documentation for the first (or second, or third) time.
>
> There is a special boot loader, u-boot, which tells Linux how much physical
> memory it is allowed to use.
> The rest of the memory is split between CMEM and BIOS (and or unallocated).
>
> The kernel module `cmem.ko` accesses physical memory, past the size that
> Linux is aware of, by its physical address with a given size at module load
> time.
>
> To allocate the memory in the CMEM area, I include TI's cmem C library and
> use their own `alloc_contig` function, which then gives direct access to
> physical memory which is guaranteed to be physically contiguous.
>
> Another kernel module, `dsplink.ko` allows communication between the two
> OSes. For example, I `contig_alloc()` on the Linux side, but then pass that
> address to the DSP side so that BIOS knows that both the dsp algorithm and
> the arm application have shared access to this memory.
>
> Otherwise, BIOS would only know about the memory that it has been allocated
> with (which is setup in special TI makefile-thing for DSP executables).
>
> BIOS also uses `malloc()`, but it has the caveat that in their flavor of C,
> any memory allocation must be done before any executable statement.
