Kernel booting process. Part 4.
================================================================================

Transition to 64-bit mode
--------------------------------------------------------------------------------

It is the fourth part of the `Kernel booting process` and we will see first steps in the [protected mode](http://en.wikipedia.org/wiki/Protected_mode), like checking that cpu supports the [long mode](http://en.wikipedia.org/wiki/Long_mode) and [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions), [paging](http://en.wikipedia.org/wiki/Paging) and initialization of the page tables and transition to the long mode in in the end of this part.

**NOTE: will be much assembly code in this part, so if you have poor knowledge, read a book about it**

In the previous [part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md) we stopped at the jump to the 32-bit entry point in the [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S):

```assembly
jmpl	*%eax
```

Recall that `eax` register contains the address of the 32-bit entry point. We can read about this point from the linux kernel x86 boot protocol:

```
When using bzImage, the protected-mode kernel was relocated to 0x100000
```

And now we can make sure that it is true. Let's look on registers value in 32-bit entry point:

```
eax            0x100000	1048576
ecx            0x0	    0
edx            0x0	    0
ebx            0x0	    0
esp            0x1ff5c	0x1ff5c
ebp            0x0	    0x0
esi            0x14470	83056
edi            0x0	    0
eip            0x100000	0x100000
eflags         0x46	    [ PF ZF ]
cs             0x10	16
ss             0x18	24
ds             0x18	24
es             0x18	24
fs             0x18	24
gs             0x18	24
```

We can see here that `cs` register contains - `0x10` (as you can remember from the previous part, it is the second index in the Global Descriptor Table), `eip` register is `0x100000` and base address of the all segments include code segment is zero. So we can get physical address, it will be `0:0x100000` or just `0x100000`, as in boot protocol. Now let's start with 32-bit entry point.

32-bit entry point
--------------------------------------------------------------------------------

We can find the definition of the 32-bit entry point in [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S):

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
....
....
....
ENDPROC(startup_32)
```

First of all why `compressed` directory? Actually `bzimage` is a gzipped `vmlinux + header + kernel setup code`. We saw the kernel setup code in all of the previous parts. So, the main goal of the `head_64.S` is to prepare for entering long mode, enter into it and decompress the kernel. We will see all of these steps besides kernel decompression in this part.

Also you can note that there are two files in the `arch/x86/boot/compressed` directory:

* head_32.S
* head_64.S

We will see only `head_64.S` because we are learning linux kernel for `x86_64`. `head_32.S` even not compiled in our case. Let's look at [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/Makefile). We can see there the following target:

```Makefile
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
	$(obj)/string.o $(obj)/cmdline.o \
	$(obj)/piggy.o $(obj)/cpuflags.o
```

Note on `$(obj)/head_$(BITS).o`. It means that compilation of the head_{32,64}.o depends on value of the `$(BITS)`. We can find it in the other Makefile - [arch/x86/kernel/Makefile](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/Makefile):

```Makefile
ifeq ($(CONFIG_X86_32),y)
	    BITS := 32
        ...
		...
else
		...
		...
        BITS := 64
endif
```

Now we know where to start, so let's do it.

Reload the segments if needed
--------------------------------------------------------------------------------

As I wrote above, we start in [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S). First of all we can see before the `startup_32` definition:

```assembly
    __HEAD
	.code32
ENTRY(startup_32)
```

`__HEAD` is defined in [include/linux/init.h](https://github.com/torvalds/linux/blob/master/include/linux/init.h) and looks like:

```C
#define __HEAD		.section	".head.text","ax"
```

We can find this section in the [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/vmlinux.lds.S) linker script:

```
SECTIONS
{
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
	}
```

Note on `. = 0;`. `.` is a special variable of linker - location counter. The value assigned to it is an offset relative to the offset of the segment. As we assign zero to it, we can read from comments:

```
Be careful parts of head_64.S assume startup_32 is at address 0.
```

Ok, now we know where we are, and now is the best time to look inside the `startup_32` function. 

In the start of `startup_32` we can see the `cld` instruction which clears the `DF` flag. After this, string operations like `stosb` and others will increment the index registers `esi` or `edi`.

Next we can see the check of the `KEEP_SEGMENTS` flag from `loadflags`. If you remember we already saw `loadflags` in the `arch/x86/boot/head.S` (there we checked flag `CAN_USE_HEAP`). Now we need to check the `KEEP_SEGMENTS` flag. We can find a description of this flag in the linux boot protocol:

```
Bit 6 (write): KEEP_SEGMENTS
  Protocol: 2.07+
  - If 0, reload the segment registers in the 32bit entry point.
  - If 1, do not reload the segment registers in the 32bit entry point.
    Assume that %cs %ds %ss %es are all set to flat segments with
	a base of 0 (or the equivalent for their environment).
```
	
and if `KEEP_SEGMENTS` is not set, we need to set `ds`, `ss` and `es` registers to a flat segment with base 0. That we do:

```C
	testb $(1 << 6), BP_loadflags(%esi)
	jnz 1f

	cli
	movl	$(__BOOT_DS), %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
```

remember that `__BOOT_DS` is `0x18` (index of data segment in the Global Descriptor Table). If `KEEP_SEGMENTS` is set, we jump to the label `1f` or update segment registers with `__BOOT_DS` if this flag is not set.

If you read the previous [part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md), you can remember that we already updated segment registers in the [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/pmjump.S), so why do we need to set up it again? Actually linux kernel also has the 32-bit boot protocol, so `startup_32` can be the first function which will be executed right after a bootloader transfers control to the kernel.

As we checked the `KEEP_SEGMENTS` flag and put the correct value to the segment registers, the next step is to calculate difference between where we loaded and compiled to run (remember that `setup.ld.S` contains `. = 0` at the start of the section):

```assembly
	leal	(BP_scratch+4)(%esi), %esp
	call	1f
1:  popl	%ebp
	subl	$1b, %ebp
```

Here the `esi` register contains the address of the [boot_params](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L113) structure. `boot_params` contains a special field `scratch` with offset `0x1e4`. We are getting the address of the `scratch` field + 4 bytes and puting it in the `esp` register (we will use it as stack for these calculations). After this we can see the call instruction and `1f` label as its operand. What does `call` mean? It means that it pushes the `ebp` value into the stack, then the `esp` value, then the function arguments and returns the address in the end. After this we pop return address from the stack into `ebp` register (`ebp` will contain return address) and subtract address of the previous label `1`. 

After this we have address where we loaded in the `ebp` - `0x100000`.

Now we can setup the stack and verify that the CPU supports long mode and [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions).

Stack setup and CPU verification
--------------------------------------------------------------------------------

Next we can see assembly code which sets up a new stack for kernel decompression:

```assembly
	movl	$boot_stack_end, %eax
	addl	%ebp, %eax
	movl	%eax, %esp
```

`boots_stack_end` is in the `.bss` section. We can see the definition of it in the end of `head_64.S`:

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

First of all we put the address of `boot_stack_end` into the `eax` register and add to it value of `ebp` (remember that `ebp` now contains the address where we loaded - `0x100000`). In the end we just put the `eax` value into `esp` and that's all, we have a correct stack pointer.

The next step is CPU verification. We need to check that the CPU supports `long mode` and `SSE`:

```assembly
	call	verify_cpu
	testl	%eax, %eax
	jnz	no_longmode
```

It just calls `verify_cpu` function from [arch/x86/kernel/verify_cpu.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/verify_cpu.S) which contains a couple of calls to the `cpuid` instruction. `cpuid` is the instruction which is used for getting information about the processor. In our case it checks long mode and SSE support and returns `0` on success or `1` on fail in the `eax` register. 

If `eax` is not zero, we jump to the `no_longmode` label which just stops the CPU with a `hlt` instruction while any hardware interrupt will not happen.

```assembly
no_longmode:
1:
	hlt
	jmp     1b
```

We set stack, checked CPU and now can move on to the next step.

Calculate relocation address
--------------------------------------------------------------------------------

The next step is calculating relocation address for decompression if needed. We can see the following assembly code:

```assembly
#ifdef CONFIG_RELOCATABLE
	movl	%ebp, %ebx
	movl	BP_kernel_alignment(%esi), %eax
	decl	%eax
	addl	%eax, %ebx
	notl	%eax
	andl	%eax, %ebx
	cmpl	$LOAD_PHYSICAL_ADDR, %ebx
	jge	1f
#endif
	movl	$LOAD_PHYSICAL_ADDR, %ebx
1:
	addl	$z_extract_offset, %ebx
```

First of all note on `CONFIG_RELOCATABLE` macro. This configuration option is defined in the [arch/x86/Kconfig](https://github.com/torvalds/linux/blob/master/arch/x86/Kconfig) and as we can read from it's description:

```
This builds a kernel image that retains relocation information
so it can be loaded someplace besides the default 1MB.

Note: If CONFIG_RELOCATABLE=y, then the kernel runs from the address
it has been loaded at and the compile time physical address
(CONFIG_PHYSICAL_START) is used as the minimum location.
```

In short words, this code calculates the address to move the kernel to for decompression put it into the `ebx` register if the kernel is relocatable or bzimage will decompress itself above `LOAD_PHYSICAL_ADDR`.

Let's look at the code. If we have `CONFIG_RELOCATABLE=n` in our kernel configuration file, it just puts `LOAD_PHYSICAL_ADDR` into the `ebx` register and adds `z_extract_offset` to `ebx`. As `ebx` is zero for now, it will contain `z_extract_offset`. Now let's try to understand these two values.

`LOAD_PHYSICAL_ADDR` is the macro which defined in [arch/x86/include/asm/boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/boot.h) and it looks like this:

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

Here we calculate the aligned address where the kernel is loaded (`0x100000` or 1 megabyte in our case). `PHYSICAL_ALIGN` is an alignment value to which the kernel should be aligned, and it ranges from `0x200000` to `0x1000000` for x86_64. With the default values we will get 2 megabytes in the `LOAD_PHYSICAL_ADDR`:

```python
>>> 0x100000 + (0x200000 - 1) & ~(0x200000 - 1)
2097152
```

After we retrieve the alignment unit, we add `z_extract_offset` (which is `0xe5c000` in my case) to the 2 megabytes. In the end we will get 17154048 bytes offset. You can find `z_extract_offset` in `arch/x86/boot/compressed/piggy.S`. This file is generated in compile time by the [mkpiggy](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/mkpiggy.c) program.

Now let's try to understand the code if `CONFIG_RELOCATABLE` is `y`. 

First of all we put the `ebp` value into `ebx` (remember that `ebp` contains address where we loaded) and `kernel_alignment` field from kernel setup header into the `eax` register. `kernel_alignment`  is a physical address of alignment required for the kernel. Next we do the same as in the previous case (when kernel is not relocatable), but we just use the value of the `kernel_alignment` field as the align unit and `ebx` (address where we loaded) as the base address instead of `CONFIG_PHYSICAL_ALIGN` and `LOAD_PHYSICAL_ADDR`. 

After we calculate the address, we compare it with `LOAD_PHYSICAL_ADDR` and add `z_extract_offset` to it again or put `LOAD_PHYSICAL_ADDR` in the `ebx` if the calculated address is less than we need.

After all of this calculation we will have `ebp` which contains the address where we loaded and `ebx` with the address to which the kernel will be moved for decompression.

Preparation before entering long mode
--------------------------------------------------------------------------------

Now we need to do the last preparations before we can see the transition to 64-bit mode. At first we need to update the Global Descriptor Table for this:

```assembly
	leal	gdt(%ebp), %eax
	movl	%eax, gdt+2(%ebp)
	lgdt	gdt(%ebp)
```

Here we put the address from `ebp` with `gdt` offset into the `eax` register, next we put this address into `ebp` with offset `gdt+2` and load the Global Descriptor Table with the `lgdt` instruction. 

Let's look at the Global Descriptor Table definition:

```assembly
	.data
gdt:
	.word	gdt_end - gdt
	.long	gdt
	.word	0
	.quad	0x0000000000000000	/* NULL descriptor */
	.quad	0x00af9a000000ffff	/* __KERNEL_CS */
	.quad	0x00cf92000000ffff	/* __KERNEL_DS */
	.quad	0x0080890000000000	/* TS descriptor */
	.quad   0x0000000000000000	/* TS continued */
```

It is defined in the same file as the `.data` section. It contains 5 descriptors: null descriptor, for kernel code segment, kernel data segment and two task descriptors. We already loaded the GDT in the previous [part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md), we're doing almost the same here, but descriptors with `CS.L = 1` and `CS.D = 0` for execution in 64 bit mode.

After we have loaded the Global Descriptor Table, we must enable [PAE](http://en.wikipedia.org/wiki/Physical_Address_Extension) mode by putting the value of the `cr4` register into `eax`, setting 5 bit in it and loading it again into `cr4` :

```assembly
	movl	%cr4, %eax
	orl	$X86_CR4_PAE, %eax
	movl	%eax, %cr4
```

Now we are almost finished with all preparations before we can move into 64-bit mode. The last step is to build page tables, but before that, here is some information about long mode.

Long mode
--------------------------------------------------------------------------------

Long mode is the native mode for x86_64 processors. First of all let's look at some differences between `x86_64` and `x86`.

It provides features such as:

* New 8 general purpose registers from `r8` to `r15` + all general purpose registers are 64-bit now
* 64-bit instruction pointer - `RIP`
* New operating mode - Long mode
* 64-Bit Addresses and Operands
* RIP Relative Addressing (we will see an example if it in the next parts)

Long mode is an extension of legacy protected mode. It consists of two sub-modes:

* 64-bit mode
* compatibility mode

To switch into 64-bit mode we need to do following things:

* enable PAE (we already did it, see above)
* build page tables and load the address of the top level page table into the `cr3` register 
* enable `EFER.LME`
* enable paging

We already enabled `PAE` by setting the PAE bit in the `cr4` register. Now let's look at paging.

Early page tables initialization
--------------------------------------------------------------------------------

Before we can move into 64-bit mode, we need to build page tables, so, let's look at the building of early 4G boot page tables. 

**NOTE: I will not describe theory of virtual memory here, if you need to know more about it, see links in the end**

The Linux kernel uses 4-level paging, and generally we build 6 page tables:

* One PML4 table
* One PDP  table
* Four Page Directory tables

Let's look at the implementation of it. First of all we clear the buffer for the page tables in memory. Every table is 4096 bytes, so we need 24 kilobytes buffer:

```assembly
	leal	pgtable(%ebx), %edi
	xorl	%eax, %eax
	movl	$((4096*6)/4), %ecx
	rep	stosl
```

We put the address stored in `ebx` (remember that `ebx` contains the address to relocate the kernel for decompression) with `pgtable` offset to the `edi` register. `pgtable` is defined in the end of `head_64.S` and looks:

```assembly
	.section ".pgtable","a",@nobits
	.balign 4096
pgtable:
	.fill 6*4096, 1, 0
```

It is in the `.pgtable` section and its size is 24 kilobytes. After we put the address in `edi`, we zero out the `eax` register and write zeros to the buffer with the `rep stosl` instruction.

Now we can build the top level page table - `PML4` - with:

```assembly
	leal	pgtable + 0(%ebx), %edi
	leal	0x1007 (%edi), %eax
	movl	%eax, 0(%edi)
```

Here we get the address stored in the `ebx` with `pgtable` offset and put it in `edi`. Next we put this address with offset `0x1007` in the `eax` register. `0x1007` is 4096 bytes (size of the PML4) + 7 (PML4 entry flags - `PRESENT+RW+USER`) and puts `eax` in `edi`. After this manipulation `edi` will contain the address of the first Page Directory Pointer Entry with flags - `PRESENT+RW+USER`.

In the next step we build 4 Page Directory entries in the Page Directory Pointer table, where the first entry will be with `0x7` flags and the others with `0x8`:

```assembly
	leal	pgtable + 0x1000(%ebx), %edi
	leal	0x1007(%edi), %eax
	movl	$4, %ecx
1:  movl	%eax, 0x00(%edi)
	addl	$0x00001000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

We put the base address of the page directory pointer table in `edi` and the address of the first page directory pointer entry in `eax`. Put `4` in the `ecx` register, it will be a counter in the following loop and write the address of the first page directory pointer table entry  to the `edi` register. 

After this `edi` will contain the address of the first page directory pointer entry with flags `0x7`. Next we just calculate the address of following page directory pointer entries with flags `0x8` and write their addresses to `edi`.

The next step is building the `2048` page table entries by 2 megabytes:

```assembly
	leal	pgtable + 0x2000(%ebx), %edi
	movl	$0x00000183, %eax
	movl	$2048, %ecx
1:  movl	%eax, 0(%edi)
	addl	$0x00200000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

Here we do almost the same as in the previous example, except the first entry will be with flags - `$0x00000183` - `PRESENT + WRITE + MBZ` and all other entries with `0x8`. In the end we will have 2048 pages by 2 megabytes.

Our early page table structure are done, it maps 4 gigabytes of memory and now we can put the address of the high-level page table - `PML4` - in `cr3` control register:

```assembly
	leal	pgtable(%ebx), %eax
	movl	%eax, %cr3
```

That's all. Now we can see transition to the long mode.

Transition to long mode
--------------------------------------------------------------------------------

First of all we need to set the `EFER.LME` flag in the [MSR](http://en.wikipedia.org/wiki/Model-specific_register) to `0xC0000080`:

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_LME, %eax
	wrmsr
```

Here we put the `MSR_EFER` flag (which is defined in [arch/x86/include/uapi/asm/msr-index.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/msr-index.h#L7)) in the `ecx` register and call `rdmsr` instruction which reads the [MSR](http://en.wikipedia.org/wiki/Model-specific_register) register. After `rdmsr` executes, we will have the resulting data in `edx:eax` which depends on the `ecx` value. We check the `EFER_LME` bit with the `btsl` instruction and write data from `eax` to the `MSR` register with the `wrmsr` instruction.

In the next step we push the address of the kernel segment code to the stack (we defined it in the GDT) and put the address of the `startup_64` routine in `eax`.

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
```

After this we push this address to the stack and enable paging by setting `PG` and `PE` bits in the `cr0` register:

```assembly
	movl	$(X86_CR0_PG | X86_CR0_PE), %eax
	movl	%eax, %cr0
```

and call:

```assembly
lret
```

Remember that we pushed the address of the `startup_64` function to the stack in the previous step, and after the `lret` instruction, the CPU extracts the address of it and jumps there. 

After all of these steps we're finally in 64-bit mode:

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
....
....
....
```

That's all!

Conclusion
--------------------------------------------------------------------------------

This is the end of the fourth part linux kernel booting process. If you have questions or suggestions, ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create an [issue](https://github.com/0xAX/linux-insides/issues/new).

In the next part we will see kernel decompression and many more.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Intel® 64 and IA-32 Architectures Software Developer’s Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [GNU linker](http://www.eecs.umich.edu/courses/eecs373/readings/Linker.pdf)
* [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)
* [Paging](http://en.wikipedia.org/wiki/Paging)
* [Model specific register](http://en.wikipedia.org/wiki/Model-specific_register)
* [.fill instruction](http://www.chemie.fu-berlin.de/chemnet/use/info/gas/gas_7.html)
* [Previous part](https://github.com/0xAX/linux-insides/blob/master/Booting/linux-bootstrap-3.md)
* [Paging on osdev.org](http://wiki.osdev.org/Paging)
* [Paging Systems](https://www.cs.rutgers.edu/~pxk/416/notes/09a-paging.html)
* [x86 Paging Tutorial](http://www.cirosantilli.com/x86-paging/)
