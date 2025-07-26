---
title: "Code Models and Relocation Types"
date: 2025-07-26
categories:
  - "Programming"
tags:
  - "ELF"
  - "LLVM"
comments: true # Enable Disqus comments for specific page
toc: true
---

本文主要探讨 code models, relocation 与 relocation types。文章首先解释不同的 code models 如何影响编译器的代码生成，然后介绍 relocation 这一机制和常见的 relocation type。文章结合多个代码示例说明在不同场景下（如不同 code models 下的函数调用、全局变量访问）生成的汇编代码以及所使用的 relocation types。

<!--more-->

## ABI

本文绝大部分内容来源于 ELF ABI，ELF ABI 包含两部分：generic specification (gABI) 和 processor-specific extensions (psABI)。

1. **gABI**(SYSTEM V APPLICATION BINARY INTERFACE): https://refspecs.linuxfoundation.org/elf/gabi41.pdf

2. ELF x86-64-ABI **psABI** (System V Application Binary Interface AMD64 Architecture Processor Supplement): https://gitlab.com/x86-psABIs/x86-64-ABI/-/jobs/artifacts/master/raw/x86-64-ABI/abi.pdf?job=build

## Position-Independent Code

[Wikipedia](https://en.wikipedia.org/wiki/Position-independent_code) 对 position-independent code 的介绍：

> In computing, position-independent code (PIC) or position-independent executable (PIE) is a body of machine code that executes properly **regardless of its memory address**. PIC is **commonly used for shared libraries**, so that the same library code can be loaded at a location in each program's address space where it does not overlap with other memory in use by ...

ELF x86-64-ABI psABI 关于 GOT 和 PLT 的描述：

> Global Offset Table (GOT)
>
> **Position-independent code cannot, in general, contain absolute virtual addresses.** Global offset tables hold absolute addresses in private data, thus making the addresses available without compromising the position-independence and shareability of a program's text. A program references its global offset table using position-independent addressing and extracts absolute values, thus redirecting position-independent references to absolute locations.

> Procedure Linkage Table (PLT)
>
> Much as the global offset table redirects position-independent address calculations to absolute locations, **the procedure linkage table redirects position-independent function calls to absolute locations**. The link editor cannot resolve execution transfers (such as function calls) from one executable or shared object to another. Consequently, the link editor arranges to have the program transfer control to entries in the procedure linkage table. On the AMD64 architecture, procedure linkage tables reside in shared text, but they use addresses in the private global offset table. The dynamic linker determines the destinations's absolute addresses and modifies the global offset table’s memory image accordingly. The dynamic linker thus can redirect the entries without compromising the position-independence and shareability of the program’s text. Executable files and shared object files have separate procedure linkage tables.

1. Position-independent code (PIC) 可以被加载至任意内存地址执行。所以称作地址无关。

2. Position-independent code 通常被用于 shared libraries。

3. 完全由 position-independent code 组成的 executable 是 position-independent executables (PIE)，PIE 常用于 ASLR (Address space layout randomization)。

4. Position-independent code 中函数调用通常通过 PLT。

5. Position-independent code 中访问全局变量通常通过 GOT 间接进行，GOT 中保存了全局变量的地址。

6. Position-independent code 可以被加载至任意内存地址执行，所以 position-independent code 无法通过绝对地址进行函数调用和访问全局变量。通过引入 PLT 和 GOT 将对函数和全局变量的直接访问变为对 PLT 和 GOT 的间接访问，dynamic loader 负责填充 PLT 和 GOT。

编译器指定生成 position-independent code 的编译选项见 [Code Gen Options (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html#index-fpic)。可以看到，编译器有两套与 position-independent code 相关的编译选项，`-fpic/-fpie` 和 `-fPIC/-fPIE`，这与不同架构对 GOT 的大小限制有关。x86 对于 GOT 大小没有限制，所以 `-fpic/-fpie` 和 `-fPIC/-fPIE` 的效果没有区别。

## Why Code Models

本节讨论为什么需要 code models。

### Addressing Modes

寻址模式 (addressing modes) 是指令集架构 (ISA) 中定义的一种规则，它规定了如何解释指令中的操作数，从而计算出要访问的有效地址 (effective address, computed address used to access memory)。

可以非严格将 x86-64 的 addressing modes 分为两类：Base-Index-Scale-Displacement Addressing 和 PC-Relative Addressing。

#### Base-Index-Scale-Displacement Addressing

有效地址 Effective Address = Base + (Index * Scale) + Displacement

公式中 Base, Index, Scale, Displacement 都是可选的，含义如下（详见 Intel SDM Vol. 1 "3.7.5 Specifying an Offset"）：

- *Base* — The value in a 64-bit general-purpose register.

- *Index* — The value in a 64-bit general-purpose register.

- *Scale factor* — A value of 2, 4, or 8 that is multiplied by the index value.

- *Displacement* — An 8-bit, 16-bit, or 32-bit value.

注：只使用 displacement 进行寻址时，称作 absolute addressing。

举例说明：

```gas
# Displacement
# This form of an address is sometimes called an absolute or static address.
movl $1, 0x604892         # direct (address is constant value)

# Base
movl $1, (%rax)           # indirect (address is in register %rax)

# Base + Displacement
movl $1, -24(%rbp)        # indirect with displacement

# (Index ∗ Scale) + Displacement
movl $1, 0x8(, %rdx, 4)   # (special case scaled-index, base assumed 0)

# Base + Index + Displacement
movl $1, 0x4(%rax, %rcx)  # (special case scaled-index, scale assumed 1)

# Base + (Index ∗ Scale) + Displacement
movl $1, 8(%rsp, %rdi, 4) # indirect with displacement and scaled-index
movl $1, (%rax, %rcx, 8)  # (special case scaled-index, displ assumed 0)
```

#### PC-Relative Addressing

注：PC-relative addressing, RIP-relative addressing 和 instruction pointer relative addressing 描述的是同一个概念。x86-64 下 PC (Program Counter) / IP(Instruction Pointer) 就是 RIP (64-bit Instruction Pointer) 寄存器。

Intel SDM Vol. 1 3.7.5.1 "Specifying an Offset in 64-Bit Mode"：

> RIP + Displacement ⎯ In 64-bit mode, RIP-relative addressing uses a signed 32-bit displacement to calculate the effective address of the next instruction by sign-extend the 32-bit value and add to the 64-bit value in RIP.

Intel SDM Vol. 2 2.2.1.6 "RIP-Relative Addressing"：

> A new addressing form, RIP-relative (relative instruction-pointer) addressing, is implemented in 64-bit mode. An effective address is formed by adding displacement to the 64-bit RIP of the next instruction.
>
> RIP-relative addressing allows specific ModR/M modes to address memory relative to the 64-bit RIP using a signed 32-bit displacement. This provides an offset range of ±2GB from the RIP.

有效地址 Effective Address= RIP + Displacement

PC-relative addressing 可以看作是 Base-Index-Scale-Displacement Addressing 的一种特例，RIP 是 Base，配合 signed 32-bit displacement。PC-relative addressing 可以访问 RIP ± 2GB 的地址范围。

举例说明：

```gas
mov $0x1, 0x10(%rip)
```

### Instruction Format

了解 addressing modes 后，我们再以 CALL 和 MOV 指令为例说明指令格式的要求和限制。

注：本节用于描述 CALL 和 MOV 指令的表格 instruction 列使用的是 intel 汇编语法格式。该表格各个字段的完整含义，详见 Intel SDM Vol. 2A "3.1.1 Instruction Format"。

#### CALL — Call Procedure

https://www.felixcloutier.com/x86/call

| Opcode | Instruction | Op/En | 64-bit Mode | Compat/Leg Mode | Description                                                                                                                  |
| ------ | ----------- | ----- | ----------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| E8 cw  | CALL rel16  | D     | N.S.        | Valid           | Call near, relative, displacement relative to next instruction.                                                              |
| E8 cd  | CALL rel32  | D     | Valid       | Valid           | Call near, relative, displacement relative to next instruction. 32-bit displacement sign extended to 64-bits in 64-bit mode. |
| FF /2  | CALL r/m64  | M     | Valid       | N.E.            | Call near, absolute indirect, address given in r/m64.                                                                        |

Opcode 列中出现的 "cw", "cd" 含义如下：

> **cb, cw, cd, cp, co, ct** — A 1-byte (cb), 2-byte (cw), 4-byte (cd), 6-byte (cp), 8-byte (co) or 10-byte (ct) value following the opcode. This value is used to specify a code offset and possibly a new value for the code segment register.

Instruction 列中出现的 "rel16", "rel32", "r/m64" 含义如下：

> **rel16, rel32** — A relative address within the same code segment as the instruction assembled. The rel16 symbol applies to instructions with an operand-size attribute of 16 bits; the rel32 symbol applies to instructions with an operand-size attribute of 32 bits.
>
> **r/m64** — A quadword general-purpose register or memory operand used for instructions whose operand-size attribute is 64 bits when using REX.W. Quadword general-purpose registers are: RAX, RBX, RCX, RDX, RDI, RSI, RBP, RSP, R8–R15; these are available only in 64-bit mode. The contents of memory are found at the address provided by the effective address computation.

**Call near, relative direct `CALL rel32`**

> A relative offset (*rel16* or *rel32*) is generally specified as a label in assembly code. But at the machine code level, it is encoded as a signed, 16- or 32-bit immediate value. This value is added to the value in the EIP(RIP) register.
>
> In 64-bit mode the relative offset is always a 32-bit immediate value which is **sign extended** to 64-bits before it is added to the value in the RIP register for the target calculation.

当我们在汇编上写 `call label` 时，实际在机器代码中 label 是编码为相对偏移的。举例说明：

```Bash
$ echo 'call 0xdeadbeef' | llvm-mc --triple=x86_64 --filetype=obj -o call-deadbeef.o

$ ld.lld call-deadbeef.o --noinhibit-exec --emit-relocs
ld.lld: error: call-deadbeef.o:(.text+0x1):
relocation R_X86_64_PC32 out of range: 3733827018 is not in [-2147483648, 2147483647];
references ''
>>> defined in call-deadbeef.o

$ llvm-objdump -dr a.out
0000000000201120 <.text>:
  201120: e8 ca ad 8d de  callq        0xffffffffdeadbeef <.text+0xffffffffde8dadcf>
       0000000000201121:  R_X86_64_PC32        *ABS*+0xdeadbeeb

# 0xdeadbeef - (0x201120 + 5) = 3733827018 = 0xde8dadca
```

`call 0xdeadbeef` 的编码是 `e8 ca ad 8d de`，对应的 instruction 及 opcode 是 `CALL rel32` 和 `E8 cd`。这里 opcode 中的 "cd" 指是 4-byte value following the opcode。对于 `call 0xdeadbeef` 这条指令 "cd" 就是 `ca ad 8d de` 即 0xde8dadca，是相对偏移：`0xde8dadca = 0xdeadbeef - (0x201120 + 5)`。

**Call near, absolute indirect `CALL r/m64`**

> For a near call absolute, an absolute offset is specified indirectly in a general-purpose register or a memory location (*r/m16*, *r/m32, or r/m64*). The operand-size attribute determines the size of the target operand (16, 32 or 64 bits).
>
> When in 64-bit mode, the operand size for near call (and all near branches) is forced to 64-bits. Absolute offsets are loaded directly into the EIP(RIP) register.

在上例 a.out 中 0x201120 处的指令 `call 0xdeadbeef`，其相对偏移 0xde8dadca=3733827018 超出了 signed 32-bit 能表示的范围，所以会链接时报错 "3733827018 is not in [-2147483648, 2147483647]"。因此这里如果想要实现对 `0xdeadbeef` 的调用，只能先将 `0xdeadbeef` 保存至 `r/m64` (A quadword general-purpose register or memory operand used for instructions whose operand-size attribute is 64 bits) 中，然后 indirect call `CALL r/m64`：

```Bash
$ cat indirect-call-deadbeef.s
.globl _start
        .text
_start:
        movabsq $0xdeadbeef, %rax
        callq   *%rax

$ llvm-mc --triple=x86_64 --filetype=obj -o indirect-call-deadbeef.o indirect-call-deadbeef.s
$ ld.lld indirect-call-deadbeef.o
```

#### MOV — Move

https://www.felixcloutier.com/x86/mov

通常指令中的 displacement 和 immediate 都是 32-bit 的。比较特殊的是 MOV 指令，MOV 指令的操作数支持 64-bit 的 displacement 和 immediate。

| Opcode            | Instruction      | Op/En | 64-bit Mode | Compat/Leg Mode | Description                       |
| ----------------- | ---------------- | ----- | ----------- | --------------- | --------------------------------- |
| REX.W + A1        | MOV RAX, moffs64 | FD    | Valid       | N.E.            | Move quadword at (offset) to RAX. |
| REX.W + A3        | MOV moffs64, RAX | TD    | Valid       | N.E.            | Move RAX to (offset).             |
| REX.W + B8+ rd io | MOV r64, imm64   | OI    | Valid       | N.E.            | Move imm64 to r64.                |

Instruction 列中出现的 "moffs64", "imm64" 含义如下：

> **moffs8, moffs16, moffs32, moffs64** — A simple memory variable (memory offset) of type byte, word, doubleword, quadword used by some variants of the MOV instruction. The actual address is given by a simple offset relative to the segment base. No ModR/M byte is used in the instruction. The number shown with moffs indicates its size, which is determined by the address-size attribute of the instruction.
>
> **imm64** — An immediate quadword value used for instructions whose operand-size attribute is 64 bits. The value allows the use of a number between +9,223,372,036,854,775,807 and –9,223,372,036,854,775,808 inclusive.

上表中 3 类 MOV instruction 对应的 AT&T 语法格式是 `movabs`，见 [9.16.3.1 AT&T Syntax versus Intel Syntax](https://sourceware.org/binutils/docs/as/i386_002dVariations.html#AT_0026T-Syntax-versus-Intel-Syntax)。

下例中三条 `movabs` 指令分别是将 64-bit 内存地址处的内容加载到 RAX 寄存器，将 RAX 寄存器内容存储至一 64-bit 内存地址处，将任意 64-bit 常量保存至寄存器中。

```gas
movabsq 0xdeadbeef, %rax
movabsq %rax, 0xdeadbeef
movabsq $0xdeadbeef, %rax
```

需要注意，使用 `movabs` 指令将 64-bit 地址处的内存内容加载到寄存器或将寄存器内容存储至 64-bit 地址处时，只能是 RAX 寄存器。

```bash
$ echo -e 'movabsq 0xdeadbeef, %rbx
movabsq %rbx, 0xdeadbeef
movabsq $0xdeadbeef, %rbx'| llvm-mc --triple=x86_64 --filetype=obj -o /dev/null

<stdin>:1:21: error: invalid operand for instruction
movabsq 0xdeadbeef, %rbx
                    ^~~~
<stdin>:2:9: error: invalid operand for instruction
movabsq %rbx, 0xdeadbeef
        ^~~~
```

### Summary

在没有任何假设和约束的情况下，函数 `foo` 和 全局变量 `bar` 的地址可能是任意的。

- 对函数 `foo` 的调用。因函数 `foo` 的地址不能保证一定位于 RIP ± 2GB 的范围内，所以不能使用 `CALL rel32`。所以只能生成最保守的汇编代码，先将函数 `foo` 的地址保存在寄存器或内存中，然后 `CALL r/m64`。

- 为全局变量 `bar` 赋值。

  - 对于 PC-relative addressing，因为无法保证全局变量 `bar` 的地址一定位于 RIP ± 2GB 的范围内，所以不能直接通过 `mov $1, OFFSET(%rip)` 这样一条指令实现，其中 OFFSET = 全局变量 `bar` 的地址 - `%rip`。只能通过多条指令，先将全局变量 `bar` 的地址保存在寄存器中，然后再基于该寄存器进行寻址对全局变量 `bar` 赋值。

  - 对于 absolute addressing，尽管 `MOV RAX, moffs64` 支持 64-bit 的 displacement，但是也需要多条指令实现，先将 `$1` 保存至 RAX 寄存器中，然后 `MOV RAX, moffs64` 然后将全局变量 `bar` 赋值赋值为 1。

**至此，引出 code model：code model 是一种程序员与编译器之间的协议或者说约束，告知编译器在什么样的假设下生成代码，以获得更高的性能和更小的代码大小。**

例如编译器默认的 `-mcmodel=small` (small code model) 的假设就是程序的代码和数据都在 `[0x00000000, 0x7effffff]` 地址范围内。

- 对函数 `foo` 的调用可以直接通过`call foo`，因函数 `foo` 的地址一定在 `[0x00000000, 0x7effffff]` 范围内。

- 对全局变量 `bar` 的访问可以直接通过 `mov $1, bar`，因全局变量 `bar` 的地址一定在 `[0x00000000, 0x7effffff]` 范围内。

附 ELF x86-64-ABI psABI 有关 code model 的定义：

> The AMD64 architecture usually does not allow an instruction to encode arbitrary 64-bit constants as immediate operand. Most instructions accept 32-bit immediates that are sign extended to the 64-bit ones. Additionally the 32-bit operations with register destinations implicitly perform zero extension making loads of 64-bit immediates with upper half set to 0 even cheaper.
>
> In order to **improve** **performance** and **reduce code size**, it is desirable to use **different code models** depending on the requirements.
>
> Code models **define constraints** for symbolic values that allow the compiler to generate better code.

## x86-64 Code Models

编译器提供了编译选项来指定 code model：[x86 Options (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html#index-mcmodel_003d-7)

> `-mcmodel=small` Generate code for the small code model: the program and its symbols must be linked in the lower 2 GB of the address space. Pointers are 64 bits. Programs can be statically or dynamically linked. This is the default code model.
>
> `-mcmodel=kernel` Generate code for the kernel code model. The kernel runs in the negative 2 GB of the address space. This model has to be used for Linux kernel code.
>
> `-mcmodel=medium` Generate code for the medium model: the program is linked in the lower 2 GB of the address space. Small symbols are also placed there. Symbols with sizes larger than -mlarge-data-threshold are put into large data or BSS sections and can be located above 2GB. Programs can be statically or dynamically linked.
>
> `-mcmodel=large` Generate code for the large model. This model makes no assumptions about addresses and sizes of sections.

本节首先介绍不同的 code models，然后通过相同代码在不同 code models 下生成的汇编代码帮助理解 code models。

### Enumerating Code Models

具体介绍不同的 code models。

#### Small Code Model

> This is the fastest code model and we expect it to be suitable for the vast majority of programs.
>
> The virtual address of code executed is known at link time. Additionally all symbols are known to be located in the virtual addresses in the range `[0, 2**31 - 2**24 - 1]` (i.e. `[0x00000000, 0x7effffff]`).
>
> This allows the compiler to encode symbolic references with offsets in the range `[-(2**31), 2**24]` (i.e. `[0x80000000, 0x01000000]`) directly in the sign extended immediate operands, with offsets in the range `[0, 2**31 - 2**24]` (i.e. `[0x00000000, 0x7f000000]`) in the zero extended immediate operands and use instruction pointer relative addressing for the symbols with offsets in the range `[-(2**24), 2**24]` (i.e. `[0xff000000, 0x01000000]`).
>
> The number 24 is chosen arbitrarily. It allows for all memory of objects of size up to 2**24 or 16M bytes to be addressed directly because the base address of such objects is constrained to be less than `2**31 - 2**24` or `0x7f000000`. Without such constraint only the base address would be accessible directly, but not any offsetted variant of it.

Small code model 是最快的 code model，适用于绝大多数程序。Small code model 也是默认的 code model。

Small code model 要求所有 symbols 的虚拟地址都要在 `[0, 2**31 - 2**24)` 范围内。因此 text sections, data sections, GOT, etc 的大小显然不能超过 2GB。

使用 PC-relative addressing 或者 absolute addressing。经测试，small code model 下，x86-64 gcc 15.1 使用 PC-relative addressing，而 x86-64 clang 20.1.0 则使用 absolute addressing。

#### Medium Code Model

> The data sections are split into two parts — the regular data sections still limited in the same way as in the small code model and the large data sections having no limits except for available addressing space. The program layout must be set in a way so that large data sections (`.ldata`, `.lrodata`, `.lbss`) are not between regular text and data sections.
>
> This model requires the compiler to use `movabs` instructions to access large static data and to load addresses into registers, but keeps the advantages of the small code model for manipulation of addresses in the small data and text sections (specially needed for branches).
>
> By default only data larger than 65535 bytes will be placed in the large data section.

Medium code model 与 small code model 的区别是：medium code model 将 data sections (`.data`, `.rodata`, `.bss`) 分为 regular data sections 和 large data sections 两部分。不考虑 large data sections，medium code model 和 small code model 的限制是一样的。Medium code model 下，要求 large data sections 的地址布局不能被放置在 text sections 和 regular data sections 之间，large data sections 的大小是可以超过 2GB 的。

默认情况下，medium code model 会将 > 65535 bytes 的全局变量放在 large data sections 中。可以通过编译选项 `-mlarge-data-threshold=` 对这个阈值进行设置。对于 large data sections 中的全局变量，编译器需要使用 `movabs` 指令将其地址加载至寄存器进行访问。

#### Large Code Model

> It is possible to avoid the limitation on the text section in the small and medium models by breaking up the program into multiple shared libraries, so this model is strictly only required if the text of a single function becomes larger than what the medium model allows.
>
> The large code model makes no assumptions about addresses and sizes of sections. Although not strictly necessary, the data sections can be split into normal and large parts like in the medium model, to improve interoperability.
>
> The compiler is required to use the `movabs` instruction, as in the medium code model, even for dealing with addresses inside the text section. Additionally, indirect branches are needed when branching to addresses whose offset from the current instruction pointer is unknown.

根据 ABI 的描述，几乎不存在需要 large code model 的场景，可以通过将程序拆分为多个 shared libraries 来避免超出 small code model 和 medium code models 的限制。

Large code model 对于 symbols 地址和 sections 大小没有任何假设。在 medium code model 下，只有访问 large data sections 中的全局变量时才需要使用 `movabs` 指令实现。而 large code model 下，因为对 symbols 地址和 sections 大小没有任何假设，所以不仅访问 data sections，访其他 sections 也需要使用 `movabs` 指令实现。例如，large code model 的函数调用是先将函数地址加载至寄存器中，然后通过寄存器间接调用。

#### Small Position Independent Code Model

> Unlike the previous models, the virtual addresses of instructions and data are not known until dynamic link time. So all addresses have to be relative to the instruction pointer.
>
> Additionally the maximum distance between a symbol and the end of an instruction is limited to `2**31 - 2**24 - 1` or `0x7effffff`, allowing the compiler to use instruction pointer relative branches and addressing modes supported by the hardware for every symbol with an offset in the range `[−(2**24), 2**24]`(i.e. `[0xff000000, 0x01000000`).

Small/medium/large position independent code model 都是 position independent code。Position independent code 中指令和数据的虚拟地址只有在被 dynamic loader 加载时才确定，所以 position independent code model 限制的不再是绝对地址而是距离。Position independent code model 使用 PC-relative addressing。

Small position independent code model 限制 symbol 和指令之间的最大距离是 `2**31 - 2**24 - 1`。意味着 text sections, data sections 和 GOT 等互相引用时，被引用的 symbol 与引用该 symbol 的位置之间的距离要在 2GB 以内。

#### Medium Position Independent Code Model

> This model is like the previous model (small position independent code model), but similarly to the medium static model adds large data sections at the end of object files.
>
> In the medium PIC model, the instruction pointer relative addressing can not be used directly for accessing large static data, since the offset can exceed the limitations on the size of the displacement field in the instruction. Instead an unwind sequence consisting of `movabs`, `lea` and `add` needs to be used.

Medium code model 是在 small code model 的基础上将 data sections 划分为 regular data sections 和 large data sections。

Medium position independent code model 也一样，是在 small position independent code model 基础上将 data sections 划分为 regular data sections 和 large data sections。Medium position independent code model 在 large data sections 以外的部分的要求是与 small position independent code model 一样的。对于 large data sections，medium position independent code model 访问 large data sections 数据时，直接通过一条指令 PC-relative addressing 是不能完成的，需要使用 `movabs`, `lea`, `add` 这样的指令序列实现。

#### Large Position Independent Code Model

> This model is like the previous model(Medium position independent code model), but makes no assumptions about the distance of symbols.
>
> The large PIC model implies the same limitation as the medium PIC model regarding addressing of static data. Additionally, references to the global offset table and to the procedure linkage table and branch destinations need to be calculated in a similar way. Further the size of the text segment is allowed to be up to 16EB in size, hence similar restrictions apply to all address references into the text segments, including branches.

在 large code model 中，symbols 之间的距离没有任何假设。

#### Kernel Code Model

> The kernel of an operating system is usually rather small but runs in the negative half of the address space. So we define all symbols to be in the range `[2**64 - 2**31, 2**64 - 2**24]` (i.e. `[0xffffffff8000000, 0xffffffffff00000]`).
>
> This code model has advantages similar to those of the small model, but allows encoding of zero extended symbolic references only for offsets from `2**31` to `2**31 + 2**24` or from `0x80000000` to `0x81000000`. The range offsets for sign extended reference changes from 0 to `2**31 + 2**24` or `0x00000000` to `0x81000000`.

略。

### Coding Examples

通过代码示例帮助理解不同的 code models。

#### Special Assembler Symbols

代码示例中涉及到一些特殊的汇编符号：

- `_GLOBAL_OFFSET_TABLE_`: specifies the offset to the base of the GOT from the current code location.

  `_GLOBAL_OFFSET_TABLE_` 表示 GOT 相对当前 PC 的偏移。

- `name@GOT`: specifies the offset to the GOT entry for the symbol name from the base of the GOT.

  `name@GOT` 表示 symbol `name` 在 GOT 中的条目相对 GOT 起始地址的偏移。

  ```gas
  call    __x86.get_pc_thunk.ax
  addl    $_GLOBAL_OFFSET_TABLE_, %eax
  movl    foo@GOT(%eax), %eax
  ```

- `name@GOTOFF`: specifies the offset to the location of the symbol name from the base of the GOT.

  `name@GOTOFF` 表示 symbol `name` 的地址相对 GOT 起始地址的偏移。

  ```gas
  call    __x86.get_pc_thunk.ax
  addl    $_GLOBAL_OFFSET_TABLE_, %eax
  leal    foo@GOTOFF(%eax), %eax
  ```

- `name@GOTPCREL`: specifies the offset to the GOT entry for the symbol name from the current code location.

  `name@GOTPCREL` 表示 symbol `name` 在 GOT 中的条目相对当前 PC 的偏移，PCREL 就是 PC relative 的意思。

  ```gas
  movq    foo@GOTPCREL(%rip), %rax
  ```

- `name@PLT`: specifies the offset to the PLT entry of symbol name from the current code location.

  `name@PLT` 表示 symbol `name` 在 PLT 中的条目相对当前 PC 的偏移。

  ```gas
  callq fn@PLT
  ```

- `name@PLTOFF`: specifies the offset to the PLT entry of symbol name from the base of the GOT.

  `name@PLTOFF` 表示 symbol `name` 在PLT 中的条目相对 GOT 起始地址的偏移。

  ```gas
  .L3:
          leaq    .L3(%rip), %rax
          movabsq $_GLOBAL_OFFSET_TABLE_-.L3, %r11
          addq    %r11, %rax
          movabsq $_Z3foov@PLTOFF, %rdx
          addq    %rax, %rdx
          call    *%rdx
  ```

#### Data Objects

举例说明不同 code models 是如何访问全局变量的。

```C++
extern int src[65536];
extern int dst[65536];
extern int *ptr;

static int lsrc[65536];
static int ldst[65536];
static int *lptr;

void foo() {
  dst[0] = src[0];
  ptr = dst;
  *ptr = src[0];
  ldst[0] = lsrc[0];
  *lptr = lsrc[0];
}
```

##### Small Models

x86-64 gcc 15.1 https://godbolt.org/z/Ex1qj8vf8

x86-64 clang 20.1.0 https://godbolt.org/z/Y3ExW7PjE

使用 x86-64 clang 20.1.0 编译示例代码，small code model 和 small position independent code model 生成的汇编代码如下：

- x86-64 clang 20.1.0 NO-PIC

  ```gas
  _Z3foov:
      pushq       %rbp
      movq        %rsp, %rbp
  # dst[0] = src[0];
      movl        src, %eax
      # R_X86_64_32S        src
      movl        %eax, dst
      # R_X86_64_32S        dst
  # ptr = dst;
      movabsq     $dst, %rax
      # R_X86_64_64        dst
      movq        %rax, ptr
      # R_X86_64_32S        ptr
  # *ptr = src[0];
      movl        src, %ecx
      # R_X86_64_32S        src
      movq        ptr, %rax
      # R_X86_64_32S        ptr
      movl        %ecx, (%rax)
  # ldst[0] = lsrc[0];
      movl        _ZL4lsrc, %eax
      # R_X86_64_32S        .bss
      movl        %eax, _ZL4ldst
      # R_X86_64_32S        .bss+0x40000
  # *lptr = lsrc[0];
      movl        _ZL4lsrc, %ecx
      # R_X86_64_32S        .bss
      movq        _ZL4lptr, %rax
      # R_X86_64_32S        .bss+0x80000
      movl        %ecx, (%rax)
      popq        %rbp
      retq
  ```

- x86-64 clang 20.1.0 PIC

  ```gas
  _Z3foov:
      pushq       %rbp
      movq        %rsp, %rbp
  # dst[0] = src[0];
      movq        src@GOTPCREL(%rip), %rax
      # R_X86_64_GOTPCREL        src-0x4
      movl        (%rax), %ecx
      movq        dst@GOTPCREL(%rip), %rax
      # R_X86_64_GOTPCREL        dst-0x4
      movl        %ecx, (%rax)
  # ptr = dst;
      movq        ptr@GOTPCREL(%rip), %rax
      # R_X86_64_GOTPCREL        ptr-0x4
      movq        dst@GOTPCREL(%rip), %rcx
      # R_X86_64_GOTPCREL        dst-0x4
      movq        %rcx, (%rax)
  # *ptr = src[0];
      movq        src@GOTPCREL(%rip), %rax
      # R_X86_64_GOTPCREL        src-0x4
      movl        (%rax), %ecx
      movq        ptr@GOTPCREL(%rip), %rax
      # R_X86_64_GOTPCREL        ptr-0x4
      movq        (%rax), %rax
      movl        %ecx, (%rax)
  # ldst[0] = lsrc[0];
      movl        _ZL4lsrc(%rip), %eax
      # R_X86_64_PC32        .bss-0x4
      movl        %eax, _ZL4ldst(%rip)
      # R_X86_64_PC32        .bss+0x3fffc
  # *lptr = lsrc[0];
      movl        _ZL4lsrc(%rip), %ecx
      # R_X86_64_PC32        .bss-0x4
      movq        _ZL4lptr(%rip), %rax
      # R_X86_64_PC32        .bss+0x7fffc
      movl        %ecx, (%rax)
      popq        %rbp
      retq
  ```

注：

1. 本节汇编代码都是使用 AT&T 语法，`$` 前缀用于立即数，`%` 前缀用于寄存器。指令 `movabsq $dst, %rax` 是将 `dst` 的地址保存至寄存器 `%rax` 中，指令 `movl src, %eax` 是将 `src` 地址处的 4-bytes 内存内容保存至寄存器 `%eax` 中。

2. 源代码上的一条语句通常对应多条汇编指令，本节代码示例中将汇编指令对应的源码语句通过注释的形式标注在汇编指令之前。如果汇编指令存在对应的 relocation 信息，relocation 信息以注释形式标注在汇编指令之后。

##### Medium Models

x86-64 gcc 15.1 https://godbolt.org/z/vfPadohYx

x86-64 clang 20.1.0 https://godbolt.org/z/3fKdc5ohn

使用 x86-64 clang 20.1.0 编译示例代码，medium code model 和 medium position independent code model 生成的汇编代码如下：

- x86-64 clang 20.1.0 NO-PIC

  ```gas
  _Z3foov:
      pushq   %rbp
      movq    %rsp, %rbp
  # dst[0] = src[0];
      movabsq $src, %rax
      # R_X86_64_64 src
      movl    (%rax), %edx
      movabsq $dst, %rcx
      # R_X86_64_64 dst
      movl    %edx, (%rcx)
  # ptr = dst;
      movq    %rcx, ptr(%rip)
      # R_X86_64_PC32 ptr-0x4
  # *ptr = src[0];
      movl    (%rax), %ecx
      movq    ptr(%rip), %rax
      # R_X86_64_PC32 ptr-0x4
      movl    %ecx, (%rax)
  # ldst[0] = lsrc[0];
      movabsq $_ZL4lsrc, %rax
      # R_X86_64_64 .lbss
      movl    (%rax), %edx
      movabsq $_ZL4ldst, %rcx
      # R_X86_64_64 .lbss+0x40000
      movl    %edx, (%rcx)
  # *lptr = lsrc[0];
      movl    (%rax), %ecx
      movq    _ZL4lptr, %rax
      # R_X86_64_32S .bss
      movl    %ecx, (%rax)
      popq    %rbp
      retq
  ```

- x86-64 clang 20.1.0 PIC

  ```gas
  _Z3foov:
      pushq   %rbp
      movq    %rsp, %rbp
      leaq    _GLOBAL_OFFSET_TABLE_(%rip), %rax
      # R_X86_64_GOTPC32 _GLOBAL_OFFSET_TABLE_-0x4
  # dst[0] = src[0];
      movq    src@GOTPCREL(%rip), %rdx
      # R_X86_64_GOTPCREL src-0x4
      movl    (%rdx), %ecx
      movq    dst@GOTPCREL(%rip), %rsi
      # R_X86_64_GOTPCREL dst-0x4
      movl    %ecx, (%rsi)
  # ptr = dst;
      movq    ptr@GOTPCREL(%rip), %rcx
      # R_X86_64_GOTPCREL ptr-0x4
      movq    %rsi, (%rcx)
  # *ptr = src[0];
      movl    (%rdx), %edx
      movq    (%rcx), %rcx
      movl    %edx, (%rcx)
  # ldst[0] = lsrc[0];
      movabsq $_ZL4lsrc@GOTOFF, %rcx
      # R_X86_64_GOTOFF64 .lbss
      movl    (%rax,%rcx), %esi
      movabsq $_ZL4ldst@GOTOFF, %rdx
      # R_X86_64_GOTOFF64 .lbss+0x40000
      movl    %esi, (%rax,%rdx)
  # *lptr = lsrc[0];
      movl    (%rax,%rcx), %ecx
      movq    _ZL4lptr(%rip), %rax
      # R_X86_64_PC32 .bss-0x4
      movl    %ecx, (%rax)
      popq    %rbp
      retq
  ```

通过几个问题加深对 code models 的理解：

1. 为什么 medium PIC model 下，读取 `src[0]` 和读取 `lsrc[0]` 所生成的汇编指令是不一样的？

   读取 `src[0]`：

   ```gas
   movq    src@GOTPCREL(%rip), %rdx
   movl    (%rdx), %ecx
   ```

   读取 `lsrc[0]`：

   ```gas
   leaq    _GLOBAL_OFFSET_TABLE_(%rip), %rax
   movabsq $_ZL4lsrc@GOTOFF, %rcx
   movl    (%rax,%rcx), %esi
   ```

   这是因为 `src` 是 `extern` 的，而 `lsrc` 是 `static` 的。`lsrc` 是 non-preemptible，不会被 interposed 的，所以无需通过 GOT 来间接访问，可以直接访问：先获取 `_GLOBAL_OFFSET_TABLE_` 的地址保存至 `%rax`，然后获取 `_ZL4lsrc` 的地址相对 `_GLOBAL_OFFSET_TABLE_` 的偏移保存至 `%rcx`，相加得到 `_ZL4lsrc` 的地址，`(%rax,%rcx)` 就是 `lsrc[0]` 的值。

2. 为什么 medium PIC model 下读取 `lsrc[0]` 不能像 medium code model 一样通过 `mov _ZL4lsrc, %eax`？

   不同于 medium code model，medium PIC model 是 position independent code，可以被加载至任意内存地址执行。所以 medium PIC model 不能通过 absolute addressing 寻址方式访问 `lsrc`，必须通过 PC-relative addressing 访问。

   因为 `static int lsrc[65536]` 的大小 > 65535 bytes，`lsrc` 会放在 `.lbss` section 中，`.text` section 与 `.lbss` section 之间的距离不保证在 2GB 内，所以需要多条指令实现读取 `lsrc[0]`。

3. 为什么 medium PIC model 下，读取 `src` 在 GOT 中条目的内容，可以直接 `movq src@GOTPCREL(%rip), %rdx`？

   因为 GOT 不属于 large date sections，medium PIC model 下 `.got` section 与 `.text` section 之间的距离在 2GB 内。

##### Large Models

x86-64 gcc 15.1 https://godbolt.org/z/nh7vcheWc

x86-64 clang 20.1.0 https://godbolt.org/z/6MchKjPva

使用 x86-64 clang 20.1.0 编译示例代码，large code model 和 large position independent code model 生成的汇编代码如下：

- x86-64 clang 20.1.0 NO-PIC

  ```gas
  _Z3foov:
      pushq   %rbp
      movq    %rsp, %rbp
  # dst[0] = src[0];
      movabsq $src, %rcx
      # R_X86_64_64 src
      movl    (%rcx), %eax
      movabsq $dst, %rdx
      # R_X86_64_64 dst
      movl    %eax, (%rdx)
  # ptr = dst;
      movabsq $ptr, %rax
      # R_X86_64_64 ptr
      movq    %rdx, (%rax)
  # *ptr = src[0];
      movl    (%rcx), %ecx
      movq    (%rax), %rax
      movl    %ecx, (%rax)
  # ldst[0] = lsrc[0];
      movabsq $_ZL4lsrc, %rax
      # R_X86_64_64 .lbss
      movl    (%rax), %edx
      movabsq $_ZL4ldst, %rcx
      # R_X86_64_64 .lbss+0x40000
      movl    %edx, (%rcx)
  # *lptr = lsrc[0];
      movl    (%rax), %ecx
      movabsq $_ZL4lptr, %rax
      # R_X86_64_64 .lbss+0x80000
      movq    (%rax), %rax
      movl    %ecx, (%rax)
      popq    %rbp
      retq
  ```

- x86-64 clang 20.1.0 PIC

  ```gas
  _Z3foov:
      pushq   %rbp
      movq    %rsp, %rbp
  .L0$pb:
      leaq    .L0$pb(%rip), %rax
      movabsq $_GLOBAL_OFFSET_TABLE_-.L0$pb, %rcx
      # R_X86_64_GOTPC64 _GLOBAL_OFFSET_TABLE_+0x9
      addq    %rcx, %rax
  # dst[0] = src[0];
      movabsq $src@GOT, %rcx
      # R_X86_64_GOT64 src
      movq    (%rax,%rcx), %rdx
      movl    (%rdx), %ecx
      movabsq $dst@GOT, %rsi
      # R_X86_64_GOT64 dst
      movq    (%rax,%rsi), %rsi
      movl    %ecx, (%rsi)
  # ptr = dst;
      movabsq $ptr@GOT, %rcx
      # R_X86_64_GOT64 ptr
      movq    (%rax,%rcx), %rcx
      movq    %rsi, (%rcx)
  # *ptr = src[0];
      movl    (%rdx), %edx
      movq    (%rcx), %rcx
      movl    %edx, (%rcx)
  # ldst[0] = lsrc[0];
      movabsq $_ZL4lsrc@GOTOFF, %rcx
      # R_X86_64_GOTOFF64 .lbss
      movl    (%rax,%rcx), %esi
      movabsq $_ZL4ldst@GOTOFF, %rdx
      # R_X86_64_GOTOFF64 .lbss+0x40000
      movl    %esi, (%rax,%rdx)
  # *lptr = lsrc[0];
      movl    (%rax,%rcx), %ecx
      movabsq $_ZL4lptr@GOTOFF, %rdx
      # R_X86_64_GOTOFF64 .lbss+0x80000
      movq    (%rax,%rdx), %rax
      movl    %ecx, (%rax)
      popq    %rbp
      retq
  ```

注意到：

1. 访问 `static int *lptr` 的代码，在 large code model 和 medium code model 下的不同：

   large code model: `movabsq $_ZL4lptr, %rax`, `movq (%rax), %rax`

   medium code model: `movq _ZL4lptr, %rax`

2. 获取变量 `extern int src[65536]` 的地址，在 large PIC model 和 medium PIC model 下的不同：

   medium PIC model:

   ```gas
   movq    src@GOTPCREL(%rip), %rdx
   ```

   large PIC model:

   ```gas
   .L0$pb:
           leaq    .L0$pb(%rip), %rax
           movabsq $_GLOBAL_OFFSET_TABLE_-.L0$pb, %rcx
           addq    %rcx, %rax
           movabsq $src@GOT, %rcx
           movq    (%rax,%rcx), %rdx
   ```

   在 large PIC model 下，不能假设 RIP 与 GOT 之间的距离在 2GB 之内。

#### Function Calls

举例说明不同 code models 是如何进行函数调用的。

```C++
static void (*ptr) (void);
extern void foo (void);
static void bar (void);

void test() {
  foo();
  bar();

  ptr = foo;
  ptr = bar;

  (*ptr)();
}
```

##### Small and Medium Models

small x86-64 gcc 15.1 https://godbolt.org/z/avEzK3sKf

small x86-64 clang 20.1.0 https://godbolt.org/z/qTWGYKfh1

示例代码在 small/medium models 下生成的代码是一样的。
使用 x86-64 clang 20.1.0 编译示例代码，small code model 和 small position independent code model 生成的汇编代码如下：

- x86-64 clang 20.1.0 NO-PIC

  ```gas
  _Z4testv:
      pushq   %rbp
      movq    %rsp, %rbp
  # foo();
      callq   _Z3foov
      # R_X86_64_PLT32 _Z3foov-0x4
  # bar();
      callq   _ZL3barv
      # R_X86_64_PLT32 _ZL3barv-0x4
  # ptr = foo;
      movabsq $_Z3foov, %rax
      # R_X86_64_64 _Z3foov
      movq    %rax, _ZL3ptr
      # R_X86_64_32S .bss
  # ptr = bar;
      movabsq $_ZL3barv, %rax
      # R_X86_64_64 _ZL3barv
      movq    %rax, _ZL3ptr
      # R_X86_64_32S .bss
  # (*ptr)();
      callq   *_ZL3ptr
      # R_X86_64_32S .bss
      popq    %rbp
      retq
  ```

- x86-64 clang 20.1.0 PIC

  ```gas
  _Z4testv:
      pushq   %rbp
      movq    %rsp, %rbp
  # foo();
      callq   _Z3foov@PLT
      # R_X86_64_PLT32 _Z3foov-0x4
  # bar();
      callq   _ZL3barv@PLT
      # R_X86_64_PLT32 _ZL3barv-0x4
  # ptr = foo;
      movq    _Z3foov@GOTPCREL(%rip), %rax
      # R_X86_64_GOTPCREL _Z3foov-0x4
      movq    %rax, _ZL3ptr(%rip)
      # R_X86_64_PC32 .bss-0x4
  # ptr = bar;
      movq    _ZL3barv@GOTPCREL(%rip), %rax
      # R_X86_64_GOTPCREL _ZL3barv-0x4
      movq    %rax, _ZL3ptr(%rip)
      # R_X86_64_PC32 .bss-0x4
  # (*ptr)();
      callq   *_ZL3ptr(%rip)
      # R_X86_64_PC32 .bss-0x4
      popq    %rbp
      retq
  ```

##### Large Models

x86-64 gcc 15.1 https://godbolt.org/z/s9enxc7Ws

x86-64 clang 20.1.0 https://godbolt.org/z/csKKMxWjG

使用 x86-64 clang 20.1.0 编译示例代码，large code model 和 large position independent code model 生成的汇编代码如下：

- x86-64 clang 20.1.0 NO-PIC

  ```gas
  _Z4testv:
    pushq   %rbp
    movq    %rsp, %rbp
    subq    $16, %rsp
  # foo();
    movabsq $_Z3foov, %rax
    # R_X86_64_64 _Z3foov
    movq    %rax, -16(%rbp)  # 8-byte Spill
    callq   *%rax
  # bar();
    movabsq $_ZL3barv, %rax
    # R_X86_64_64 _ZL3barv
    movq    %rax, -8(%rbp)  # 8-byte Spill
    callq   *%rax
    movq    -16(%rbp), %rdx # 8-byte Reload
    movq    -8(%rbp), %rcx  # 8-byte Reload
  # ptr = foo;
    movabsq $_ZL3ptr, %rax
    # R_X86_64_64 .lbss
    movq    %rdx, (%rax)
  # ptr = bar;
    movq    %rcx, (%rax)
  # (*ptr)();
    movq    (%rax), %rax
    callq   *%rax
    addq    $16, %rsp
    popq    %rbp
    retq
  ```

- x86-64 clang 20.1.0 PIC

  ```gas
  _Z4testv:
    pushq %rbp
    movq  %rsp, %rbp
    subq  $32, %rsp
  .L0$pb:
    leaq  .L0$pb(%rip), %rax
    movabsq $_GLOBAL_OFFSET_TABLE_-.L0$pb, %rcx
    # R_X86_64_GOTPC64 _GLOBAL_OFFSET_TABLE_+0x9
    addq  %rcx, %rax           # %rax = &_GLOBAL_OFFSET_TABLE_
    movq  %rax, -8(%rbp)       # 8-byte Spill
  # foo();
    movabsq $_Z3foov@GOT, %rcx
    # R_X86_64_GOT64 _Z3foov
    movq  (%rax,%rcx), %rax
    movq  %rax, -24(%rbp)      # 8-byte Spill
    callq *%rax
    movq  -8(%rbp), %rax       # 8-byte Reload
  # bar();
    movabsq $_ZL3barv@GOT, %rcx
    # R_X86_64_GOT64 _ZL3barv
    movq  (%rax,%rcx), %rax
    movq  %rax, -16(%rbp)      # 8-byte Spill
    callq *%rax
    movq  -24(%rbp), %rsi      # 8-byte Reload
    movq  -16(%rbp), %rdx      # 8-byte Reload
    movq  -8(%rbp), %rax       # 8-byte Reload
  # ptr = foo;
    movabsq $_ZL3ptr@GOTOFF, %rcx
    # R_X86_64_GOTOFF64 .lbss
    movq  %rsi, (%rax,%rcx)
  # ptr = bar;
    movq  %rdx, (%rax,%rcx)
  # (*ptr)();
    movq  (%rax,%rcx), %rax
    callq *%rax
    addq  $32, %rsp
    popq  %rbp
    retq
  ```

### Section Layout of LLD

ld.lld 17 (https://reviews.llvm.org/D150510) 开始，使用的 section layout 如下所示：

```
.lrodata
.rodata
.text
.plt
.data.rel.ro
.got
.data
.bss
.ldata
.lbss
```

可见，通常 `.text` section 与 `.got` section 之间的距离是小于 `.text` section 与 `.data`, `.bss` sections 之间的距离的。

另外，llvm 18.1.0 (https://github.com/llvm/llvm-project/pull/73037) 开始，对于使用 large models 的编译单元，将函数放进 `.ltext` section，ld.lld 21 (https://github.com/llvm/llvm-project/pull/70358) 开始，`.ltext` section 会放置在 `.rodata` section 之前。

## Relocation

回顾下前文使用的例子：

```bash
$ echo 'call 0xdeadbeef' | llvm-mc --triple=x86_64 --filetype=obj -o call-deadbeef.o

$ ld.lld call-deadbeef.o --noinhibit-exec --emit-relocs
ld.lld: error: call-deadbeef.o:(.text+0x1):
relocation R_X86_64_PC32 out of range: 3733827018 is not in [-2147483648, 2147483647];
references ''
>>> defined in call-deadbeef.o

$ llvm-objdump -dr a.out
0000000000201120 <.text>:
  201120: e8 ca ad 8d de  callq        0xffffffffdeadbeef <.text+0xffffffffde8dadcf>
       0000000000201121:  R_X86_64_PC32        *ABS*+0xdeadbeeb

# 0xdeadbeef - (0x201120 + 5) = 3733827018 = 0xde8dadca
```

在 a.out 中 `call 0xdeadbeef` 指令位于 0x201120 地址处，相对偏移 `0xde8dadca` 被编码进指令。如果通过 `objdump` 查看 call-deadbeef.o 的话，会发现 call-deadbeef.o 中 `call 0xdeadbeef` 指令并没有将相对偏移编码进指令中，而是占位符 `0x00000000`。

```bash
$ llvm-objdump -dr call-deadbeef.o
0000000000000000 <.text>:
       0: e8 00 00 00 00                       callq        0x5 <.text+0x5>
                0000000000000001:  R_X86_64_PC32        *ABS*+0xdeadbeeb
```

这是因为在编译生成 call-deadbeef.o 时并不知道 `call 0xdeadbeef` 指令在最终可执行文件 a.out 中的地址，也就不知道相对偏移是多少，所以没有办法将相对偏移编码进指令中。而在 call-deadbeef.o 链接时，链接器知道这条指令在可执行文件中的地址，所以可以将准确的相对偏移填充进指令编码中。

**简单来说，relocation 就是把代码或数据里中的符号引用（比如变量、函数）占位符替换成它们的真实地址的过程。**

再举一个更常见的例子：

```Bash
$ cat tmp.cpp
void (*ptr) (void);
void test() { ptr = test; }

$ g++ -fno-PIC -fno-PIE -c tmp.cpp -o tmp.o && llvm-objdump -dr tmp.o

Disassembly of section .text:
0000000000000000 <_Z4testv>:
       0: 55                                   pushq       %rbp
       1: 48 89 e5                             movq        %rsp, %rbp
       4: 48 c7 05 00 00 00 00 00 00 00 00     movq        $0x0, (%rip) # movq $_Z4testv, ptr(%rip)
             0000000000000007:  R_X86_64_PC32        ptr-0x8
             000000000000000b:  R_X86_64_32S         _Z4testv
       f: 90                                   nop
      10: 5d                                   popq        %rbp
      11: c3                                   retq
```

将 tmp.cpp 编译为 tmp.o，此时并不知道变量 `ptr` 和函数 `test`的地址，所以 tmp.o 中引用 `ptr`/`test` 处写入的并不是 `ptr`/`test` 的地址而是占位符。将此处对`var`/`test` 的引用由占位符替换为实际地址是由链接器完成的。

那么链接器是怎么知道哪些位置需要 relocation、relocation 处需要替换为什么值的呢？答案是依赖于保存在 .o 文件中的 relocation sections。可以通过 `readelf -rW` 查看：

```bash
$ readelf -rW tmp.o

Relocation section '.rela.text' at offset 0x140 contains 2 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000000007  0000000300000002 R_X86_64_PC32          0000000000000000 ptr - 8
000000000000000b  000000040000000b R_X86_64_32S           0000000000000000 _Z4testv + 0
```

- tmp.o 中相对 `.text` section 偏移 0x7 处引用了 `ptr`，relocation type 是 R_X86_64_PC32。

- tmp.o 中相对 `.text` section 偏移 0xb 处引用了 `_Z4testv`，relocation type 是 R_X86_64_32S。

因为 `.text` section 中只有 `_Z4testv`，所以相对 `.text` section 的偏移就是相对 `_Z4testv` 的偏移。Relocation type 描述了如何对占位符进行替换。

### Relocation Types

ELF x86-64-ABI psABI 中列出了所有的 relocation 类型。

计算 relocation 涉及到的符号及其含义如下：

- `A` Represents the addend used to compute the value of the relocatable field.

- `B` Represents the base address at which a shared object has been loaded into memory during execution. Generally, a shared object is built with a 0 base virtual address, but the execution address will be different.

- `G` Represents the offset into the global offset table at which the relocation entry’s symbol will reside during execution.

- `GOT` Represents the address of the global offset table.

- `L` Represents the place (section offset or address) of the Procedure Linkage Table entry for a symbol.

- `P` Represents the place (section offset or address) of the storage unit being relocated (computed using r_offset).

- `S` Represents the value of the symbol whose index resides in the relocation entry.

- `Z` Represents the size of the symbol whose index resides in the relocation entry.

完整的 relocation types 如下：

| Name                                                                                                                   | Value | Field     | Calculation      |
| ---------------------------------------------------------------------------------------------------------------------- | ----- | --------- | ---------------- |
| R_X86_64_NONE                                                                                                          | 0     | none      | none             |
| R_X86_64_64                                                                                                            | 1     | word64    | S + A            |
| R_X86_64_PC32                                                                                                          | 2     | word32    | S + A - P        |
| R_X86_64_GOT32                                                                                                         | 3     | word32    | G + A            |
| R_X86_64_PLT32                                                                                                         | 4     | word32    | L + A - P        |
| R_X86_64_COPY                                                                                                          | 5     | none      | none             |
| R_X86_64_GLOB_DAT                                                                                                      | 6     | wordclass | S                |
| R_X86_64_JUMP_SLOT                                                                                                     | 7     | wordclass | S                |
| R_X86_64_RELATIVE                                                                                                      | 8     | wordclass | B + A            |
| R_X86_64_GOTPCREL                                                                                                      | 9     | word32    | G + GOT + A - P  |
| R_X86_64_32                                                                                                            | 10    | word32    | S + A            |
| R_X86_64_32S                                                                                                           | 11    | word32    | S + A            |
| R_X86_64_16                                                                                                            | 12    | word16    | S + A            |
| R_X86_64_PC16                                                                                                          | 13    | word16    | S + A - P        |
| R_X86_64_8                                                                                                             | 14    | word8     | S + A            |
| R_X86_64_PC8                                                                                                           | 15    | word8     | S + A - P        |
| R_X86_64_DTPMOD64                                                                                                      | 16    | word64    |                  |
| R_X86_64_DTPOFF64                                                                                                      | 17    | word64    |                  |
| R_X86_64_TPOFF64                                                                                                       | 18    | word64    |                  |
| R_X86_64_TLSGD                                                                                                         | 19    | word32    |                  |
| R_X86_64_TLSLD                                                                                                         | 20    | word32    |                  |
| R_X86_64_DTPOFF32                                                                                                      | 21    | word32    |                  |
| R_X86_64_GOTTPOFF                                                                                                      | 22    | word32    |                  |
| R_X86_64_TPOFF32                                                                                                       | 23    | word32    |                  |
| R_X86_64_PC64 †                                                                                                        | 24    | word64    | S + A - P        |
| R_X86_64_GOTOFF64 †                                                                                                    | 25    | word64    | S + A - GOT      |
| R_X86_64_GOTPC32                                                                                                       | 26    | word32    | GOT + A - P      |
| R_X86_64_GOT64                                                                                                         | 27    | word64    | G + A            |
| R_X86_64_GOTPCREL64                                                                                                    | 28    | word64    | G + GOT + A - P  |
| R_X86_64_GOTPC64                                                                                                       | 29    | word64    | GOT + A - P      |
| Deprecated                                                                                                             | 30    |           |                  |
| R_X86_64_PLTOFF64                                                                                                      | 31    | word64    | L + A - GOT      |
| R_X86_64_SIZE32                                                                                                        | 32    | word32    | Z + A            |
| R_X86_64_SIZE64 †                                                                                                      | 33    | word64    | Z + A            |
| R_X86_64_GOTPC32_TLSDESC                                                                                               | 34    | word32    |                  |
| R_X86_64_TLSDESC_CALL                                                                                                  | 35    | none      |                  |
| R_X86_64_TLSDESC                                                                                                       | 36    | word64×2  |                  |
| R_X86_64_IRELATIVE                                                                                                     | 37    | wordclass | indirect (B + A) |
| R_X86_64_RELATIVE64 ††                                                                                                 | 38    | word64    | B + A            |
| Deprecated                                                                                                             | 39    |           |                  |
| Deprecated                                                                                                             | 40    |           |                  |
| R_X86_64_GOTPCRELX                                                                                                     | 41    | word32    | G + GOT + A - P  |
| R_X86_64_REX_GOTPCRELX                                                                                                 | 42    | word32    | G + GOT + A - P  |
| R_X86_64_CODE_4_GOTPCRELX                                                                                              | 43    | word32    | G + GOT + A - P  |
| R_X86_64_CODE_4_GOTTPOFF                                                                                               | 44    | word32    |                  |
| R_X86_64_CODE_4_GOTPC32_TLSDESC                                                                                        | 45    | word32    |                  |
| R_X86_64_CODE_5_GOTPCRELX                                                                                              | 46    | word32    | G + GOT + A - P  |
| R_X86_64_CODE_5_GOTTPOFF                                                                                               | 47    | word32    |                  |
| R_X86_64_CODE_5_GOTPC32_TLSDESC                                                                                        | 48    | word32    |                  |
| R_X86_64_CODE_6_GOTPCRELX                                                                                              | 49    | word32    | G + GOT + A - P  |
| R_X86_64_CODE_6_GOTTPOFF                                                                                               | 50    | word32    |                  |
| R_X86_64_CODE_6_GOTPC32_TLSDESC                                                                                        | 51    | word32    |                  |
| †This relocation is used only for LP64.<br>††This relocation only appears in ILP32 executable files or shared objects. |       |           |                  |

Field 字段就是 relocation 占位符的大小，relocation type 如果以数字作为 name 后缀，那么该数字后缀通常与 field 字段对应。例如 R_X86_64_64, R_X86_64_32, R_X86_64_16, R_X86_64_8 分别对应的 field 为 word64, word32, word16, word8。

Calculation 字段描述填充占位符的值是如何计算出来的。根据 relocation calculation 公式计算出来的值必须在 field 可表示的范围内。例如，R_X86_64_32 类型的 relocation 要求计算出来的值要在 `[0, 4294967295]` 区间内，R_X86_64_32 类型的 relocation 要求计算出来的值要在 `[-2147483648, 2147483647]` 区间内。

通过不同 relocation types 的计算公式，可以理解不同的 relocation types 的使用场景：

- R_X86_64_64, R_X86_64_32, R_X86_64_32S, R_X86_64_16, R_X86_64_8 的计算公式都是 `S + A`，用于 absolute addressing 引用一个 symbol。

- R_X86_64_PC64, R_X86_64_PC32, R_X86_64_PC16, R_X86_64_PC8 的计算公式是 `S + A - P`，用于 PC-relative addressing 引用一个 symbol。

- R_X86_64_GOT64, R_X86_64_GOT32 的计算公式是 `G + A`，用于获取 symbol 在 GOT 中条目相对 GOT 起始地址的偏移。

- R_X86_64_GOTPCREL, R_X86_64_GOTPCREL64, R_X86_64_{REX_}GOTPCRELX 的计算公式是 `G + GOT + A - P`，通过 PC-relative addressing 获取 symbol 在 GOT 中条目的地址。

- R_X86_64_PLT32 计算公式是 `L + A - P`, 用于获取 symbol 在 PLT 中条目相对 PC 的偏移。

- R_X86_64_GOTPC64, R_X86_64_GOTPC32 计算公式是 `GOT + A - P`，用于获取 GOT 相对 PC 的偏移。

- R_X86_64_PLTOFF64 计算公式是 `L + A - GOT`，用于获取 symbol 在 PLT 中条目相对 GOT 的偏移。

- R_X86_64_GOTOFF64 计算公式是 `S + A - GOT`，用于获取 symbol 的地址相对 GOT 的偏移。

可以归纳出 relocation type 的选择都与哪些因素相关：

- “占位符”的大小。如占位符占据 8-bit, 16-bit, 32-bit 还是 64-bit。

- 寻址模式。absolute addressing 还是 PC-relative addressing。

- 是否涉及 GOT, PLT, Thread-Local Storage。

回到本节一开始 tmp.cpp 的例子：

```bash
$ cat tmp.cpp
void (*ptr) (void);
void test() { ptr = test; }

$ g++ -fno-PIC -fno-PIE -c tmp.cpp -o tmp.o && llvm-objdump -dr tmp.o

Disassembly of section .text:
0000000000000000 <_Z4testv>:
       0: 55                                   pushq       %rbp
       1: 48 89 e5                             movq        %rsp, %rbp
       4: 48 c7 05 00 00 00 00 00 00 00 00     movq        $0x0, (%rip) # movq $_Z4testv, ptr(%rip)
             0000000000000007:  R_X86_64_PC32        ptr-0x8
             000000000000000b:  R_X86_64_32S         _Z4testv
       f: 90                                   nop
      10: 5d                                   popq        %rbp
      11: c3                                   retq
```

`movq $_Z4testv, ptr(%rip)` 将函数 `_Z4testv` 的地址保存至指针 `ptr` 中，该指令位于相对 `.text` section 偏移 0x4 处。该指令对应的 intel 汇编语法格式的是 `MOV r/m64, imm32` 即 move imm32 sign extended to 64-bits to r/m64。

- 引用函数 `_Z4testv` 的地址使用 absolute addressing，用 imm32 (sign extended to 64-bits) 表示 `_Z4testv` 的地址，所以这里的 relocation 类型是 R_X86_64_32S。R_X86_64_32S 的计算公式是 `S + A`，这里直接 `A = 0`。

- 引用指针 `ptr` 的地址使用 PC-relative addressing，这里要求 `ptr` 的地址在当前 RIP ± 2GB 范围内，使用 R_X86_64_PC32。R_X86_64_PC32 的计算公式是 `S + A - P`，其中 `S` 是 `ptr` 的地址，`P` 是引用 `ptr` 处的地址 ，那么 `A` 是怎么确定的，为什么是 -8？

  记函数 `_Z4testv` 的地址为 `Address(_Z4testv)`，那么 `movq $_Z4testv, ptr(%rip)` 的地址就是 `Address(_Z4testv) + 4`。当 CPU 执行到这条 `mov` 指令时，它需要计算 `ptr` 的内存地址 `Address(ptr)`，根据 PC-relative addressing 中 effective address 计算方式：`Address(ptr) = RIP + displacement`。链接时链接器根据 R_X86_64_PC32 类型在相对 `_Z4testv` 偏移 0x7 处填入的值就是 `displacement`。根据 R_X86_64_PC32 类型的计算公式有 `displacement = Address(ptr) + A - P`，所以 `A = P - RIP` 即需要 relocation 的位置相对于下一条指令的偏移。所以 `A = P - RIP = Address(_Z4testv) + 0x7 - (Address(_Z4testv) + 0xf) = 0x7 - 0xf = -8`。

### Large Model Relocation Types

有些 relocation types 只有在 large models 下才会使用。再次用 "Data Object" 一节的代码示例来说明，large PIC model 下获取变量 `extern int src[65536]` 的地址所需要的指令序列如下：

```
Disassembly of section .ltext:
0000000000000000 <_Z3foov>:
       0: 55                                   pushq        %rbp
       1: 48 89 e5                             movq        %rsp, %rbp
       4: 48 8d 05 f9 ff ff ff                 leaq        -0x7(%rip), %rax
       b: 48 b9 00 00 00 00 00 00 00 00        movabsq        $0x0, %rcx
                000000000000000d:  R_X86_64_GOTPC64        _GLOBAL_OFFSET_TABLE_+0x9
      15: 48 01 c8                             addq        %rcx, %rax
      18: 48 b9 00 00 00 00 00 00 00 00        movabsq        $0x0, %rcx
                000000000000001a:  R_X86_64_GOT64        src
      22: 48 8b 14 08                          movq        (%rax,%rcx), %rdx
```

- 相对 `_Z3foov` 偏移为 0xd 处的 relocation type 是 R_X86_64_GOTPC64，引用的是 `_GLOBAL_OFFSET_TABLE_`，addend 是 0x9。使用 R_X86_64_GOTPC64 而不是 R_X86_64_GOTPC32 是因为 large PIC model 中 GOT 与 PC 的距离可能超过 2GB，超出了 32-bit 能表示的范围。R_X86_64_GOTPC64 计算公式为 `GOT + A - P`，`GOT + A - P = GOT + 0x9 - (Address(_Z3foov) + 0xd) = GOT - (Address(_Z3foov) + 0x4)`，就是 GOT 相对 `leaq -0x7(%rip), %rax` 这条指令的距离。

- 相对 `_Z3foov` 偏移为 0x1a 处的 relocation type 是 R_X86_64_GOT64，引用的是 `src`，addend 是 0x0。计算公式为 `G + A`，`G + A = G + 0x0 = G` 就是 `src` 在 GOT 中条目相对 GOT 起始地址的偏移。

- `leaq -0x7(%rip), %rax` 这条指令的地址保存在 `%rax` 中，GOT 相对 `leaq -0x7(%rip), %rax` 这条指令的距离保存在 `%rcx` 中，两者相加就是 GOT 的起始地址，再加上 `src` 在 GOT 中条目相对 GOT 起始地址的偏移就是 `src` 在 GOT 中条目的地址。

### Relax Relocations

什么是 relax relocations？简而言之就是可以被链接器 linker relaxation (`--relax`) 优化的 relocation types。

LLVM12 ([[CMake] Default ENABLE_X86_RELAX_RELOCATIONS to ON](https://github.com/llvm/llvm-project/commit/c41a18cf61790fc898dcda1055c3efbf442c14c0)) 开始，CMAKE 配置项 `ENABLE_X86_RELAX_RELOCATIONS` 默认为 `true`，默认生成 relax relocations。可以通过手动添加编译选项 `-Wa,-mrelax-relocations=no` 来控制该行为。

ELF x86-64-ABI psABI 在 4.4.1 Relocation Types 一节有如下内容：

> For the occurrence of name@GOTPCREL in the following assembler instructions:
>
> - call *name@GOTPCREL(%rip)
>
> - jmp *name@GOTPCREL(%rip)
>
> - mov name@GOTPCREL(%rip), %reg
>
> - test %reg, name@GOTPCREL(%rip)
>
> - binop name@GOTPCREL(%rip), %reg
>
> where binop is one of `adc`, `add`, `and`, `cmp`, or, `sbb`, `sub`, `xor` instructions, the following **GOTPCRELX** relocations should be generated (see also section B.2):
>
> - R_X86_64_GOTPCRELX The instruction starts at 2 bytes before the relocation offset.
>
> - R_X86_64_REX_GOTPCRELX The instruction starts at 3 bytes before the relocation offset with the REX prefix.
>
> - R_X86_64_CODE_4_GOTPCRELX The instruction starts at 4 bytes before the relocation offset.
>
> - R_X86_64_CODE_5_GOTPCRELX The instruction starts at 5 bytes before the relocation offset.
>
> - R_X86_64_CODE_6_GOTPCRELX The instruction starts at 6 bytes before the relocation offset.

即 ABI 规定：上述汇编指令**应该**生成 **GOTPCRELX** 类型（注意与 GOTPCREL 做区分）的 relocation types。

ELF x86-64-ABI psABI 在 "B.2 Optimize GOTPCRELX Relocations" 一节，描述了链接器可能如何优化 GOTPCRELX 类型的 relocations。

对 GOTPCRELX Relocations 进行优化需要满足以下几个条件：

1. symbol `foo` is defined locally 即 `foo` 是 non-preemptable

2. relocation addend 是 -4

3. relocation type 是

   1. R_X86_64_GOTPCRELX, R_X86_64_REX_GOTPCRELX

   2. R_X86_64_CODE_4_GOTPCRELX 且 first byte of the instruction at the relocation offset - 4 is 0xd5

对 relax relocations 的优化根据汇编指令的不同可以分为以下 3 类：

- Convert call and jmp

  > Convert memory operand of `call` and `jmp` into immediate operand.

  | Memory Operand           | Immediate Operand |
  | ------------------------ | ----------------- |
  | call *foo@GOTPCREL(%rip) | nop call foo      |
  | call *foo@GOTPCREL(%rip) | call foo nop      |
  | jmp *foo@GOTPCREL(%rip)  | jmp foo nop       |

- Convert mov

  > Convert memory operand of `mov` into immediate operand. When position-independent code is disabled and foo is defined locally in the lower 32-bit address space, memory operand in mov can be converted into immediate operand. Otherwise, `mov` must be changed to `lea`.

  | Memory Operand               | Immediate Operand   |
  | ---------------------------- | ------------------- |
  | mov foo@GOTPCREL(%rip), %reg | mov $foo, %reg      |
  | mov foo@GOTPCREL(%rip), %reg | lea foo(%rip), %reg |

  将 `mov foo@GOTPCREL(%rip), %reg` 转换为 `mov $foo, %reg` 要求非 PIC 且 symbol `foo` 的虚拟地址必须在低 2GB，否则只能将其转换为 `lea foo(%rip), %reg`。

  为什么有这两个要求？因为如果 PIC 那么 `foo` 的虚拟地址只有在 loader 加载时才能确定，不能通过 absolute addressing `$foo` 引用 `foo` 的地址。如果 `foo` 的地址超过 2GB，那么就需要使用 `movabs` 指令。

- Convert Test and Binop

  > Convert memory operand of `test` and `binop` into immediate operand, where `binop` is one of `adc`, `add`, `and`, `cmp`, `or`, `sbb`, `sub`, `xor` instructions, when position-independent code is disabled.

  | Memory Operand                 | Immediate Operand |
  | ------------------------------ | ----------------- |
  | test %reg, foo@GOTPCREL(%rip)  | test $foo, %reg   |
  | binop foo@GOTPCREL(%rip), %reg | binop $foo, %reg  |

  只有在非 PIC 时才会 relax 使用 GOTPCRELX 类型 relocation 的 test 和 binop 指令。

**可能是 ABI 中的缺陷**

ELF x86-64-ABI psABI 中只在 "Convert mov" 中提到将 `mov foo@GOTPCREL(%rip), %reg` 转换为 `mov $foo, %reg` 时 `foo` 的虚拟地址必须在低 2GB。我认为只对 "Convert mov" 有这样的限制是不完全的。

"Convert Test and Binop" 也应该有同样的的要求。例如，对于 test 和 add 指令：

- https://www.felixcloutier.com/x86/test，test 指令最多支持 imm32，不支持 imm64：`TEST r/m64, imm32` AND imm32 sign-extended to 64-bits with r/m64; set SF, ZF, PF according to result.

- https://www.felixcloutier.com/x86/add，add 指令最多支持 imm32，不支持 imm64：`ADD r/m64, imm32` Add imm32 sign-extended to 64-bits to r/m64.

如果 `foo` 的虚拟地址超过 2GB 那么这样的 relocation relaxation 就是错误的。如前 "Section Layout of LLD" 所述，通常 `.text` section 与 `.got` section 之间的距离是小于 `.text` section 与 `.data`/`.bss` section 之间的距离的。所以存在可能：relocation relaxation 之前 `.text` section 与 `.got` section 之间的距离没有超过 relocation overflow 的限制，但是 relocation relaxation 之后由 `.text` section 引用 `.got` section 变为 `.text` section 引用 `.data`/`.bss` section，此时 `.text` section 与 `.data`/`.bss` section 之间的距离就超过 relocation overflow 的限制了。

类似地 "Convert call and jmp" 也应该有同样的限制。

**ld.lld 相关实现**

在 lld 的实现中，只实现了第二种 "convert mov"，即只实现了将 `mov foo@GOTPCREL(%rip), %reg` 转换为 `lea foo(%rip), %reg`。

lld 的 relocation relaxation 实现是 one-pass relaxation，是一个 yes or no 的实现，做不到当发现 relocation relaxation 后会导致 relocation overflow 再回退为不做 relocation relaxation。

- Arthur Eubanks 在 中为 lld 添加了回退 relocation relaxation 决定的能力。

  [[lld/ELF] Don't relax R_X86_64_(REX_)GOTPCRELX when offset is too far](https://github.com/llvm/llvm-project/commit/9d6ec280fc537b55ef8641ea85d3522426bc5766)

  [[lld/ELF][X86] Respect outSecOff when checking if GOTPCREL can be relaxed](https://github.com/llvm/llvm-project/commit/48048051323d5dd74057dc5f32df8c3c323afcd5)

- MaskRay 在 [[ELF] -no-pie: don't optimize addq R_X86_64_REX_GOTPCRELX when !isInt<32>(va)](https://github.com/llvm/llvm-project/commit/f3c4dae6f2c44f1a7f130c4cf4b2861b62402b48) 中扩展支持了非 pie 的情况 。

## Notes of Clang/LLVM Implementation

Clang/LLVM 确定选择哪种 relocation type 的实现细节。

1. 源代码

   ```C++
   // clang -fPIC -c example.cpp -o example.o

   extern int g_var;
   long long get_g_var_addr() {
     return (long long)&g_var;
   }
   ```

2. LLVM IR

   ```Plain
   @g_var = external global i32, align 4

   define noundef i64 @get_g_var_addr()() {
   entry:
     ret i64 ptrtoint (ptr @g_var to i64)
   }
   ```

3. LLVM Machine IR

   ```Plain
   # Machine code for function get_g_var_addr(): IsSSA, TracksLiveness

   bb.0.entry:
     %0:gr64 = MOV64rm $rip, 1, $noreg, target-flags(x86-gotpcrel) @g_var, $noreg; example.cpp:5:3
     $rax = COPY %0:gr64; example.cpp:5:3
     RET64 implicit $rax; example.cpp:5:3

   # End machine code for function get_g_var_addr().
   ```

   这里为 MachineOperand `@g_var` 设置了 target-flags(x86-gotpcrel)。这个 target-flag 会影响最终重定位类型的选择。它告诉后续阶段，我们想要的不是 `g_var` 的直接地址，而是 `g_var` 在 GOT 中的条目相对于当前 PC 的偏移。

   函数 `X86TargetLowering::LowerGlobalOrExternal()` llvm/lib/Target/X86/X86ISelLowering.cpp 调用 `X86Subtarget::classifyGlobalFunctionReference()`, `X86Subtarget::classifyGlobalReference()` 来设置 target-flag。对于本例，是函数 `X86Subtarget::classifyGlobalReference` 设置对变量 `g_var` 引用的 target-flag，受 code model, position-independent-code/position-dependent-code 和 `g_var` 是否 dso_local 等因素的影响。

   Target Operand Flag 是枚举类型，定义在 llvm/lib/Target/X86/MCTargetDesc/X86BaseInfo.h 中，常见的与重定位相关的 flag 有：

   - `MO_NO_FLAG`: 默认，无特殊处理。

   - `MO_GOTPCREL`: 表示 symbol 在 GOT 中的条目相对当前 PC 的偏移。

   - `MO_GOT`: 表示 symbol 在 GOT 中的条目相对 GOT 起始地址的偏移。

   - `MO_PLT`: 用于函数调用，表示 symbol `name` 在 PLT 中的条目相对当前 PC 的偏移。

   - ...

4. AsmPrinter / MC

   1. `X86MCInstLower` 将 `MachineInstr` 转换成 `MCInst`。函数 `X86MCInstLower::LowerSymbolOperand()` llvm/lib/Target/X86/X86MCInstLower.cpp 会根据 MachineOperand 的 target-flags 返回设置了 relocation specifier 的 `MCExpr`。

      本例中的 `MO_GOTPCREL` target-flag 被转换成 `MCExpr` 的 relocation specifier `X86::S_GOTPCREL`。

   2. `X86MCCodeEmitter` 将 `MCInst` 翻译成二进制字节。当它遇到一个表达式操作数 `MCOperand::isExpr()` 时，它无法立即计算出值，于是就创建了一个 `MCFixup`。

      `MCFixup` 的 `Kind` 即 `MCFixupKind` (llvm/include/llvm/MC/MCFixup.h, llvm/lib/Target/X86/MCTargetDesc/X86FixupKinds.h) 是根据指令编码格式设置的。本例是 `movq g_var@GOTPCREL(%rip), %rax` 指令，将 `MCFixupKind` 设置为 `X86::reloc_riprel_4byte_movq_load`。详见函数 `X86MCCodeEmitter::emitMemModRMByte()`。

      `MCFixupKind` 描述的是“占位符”的大小和类型，而 `MCExpr` 描述的是要往“占位符”里填什么。

   3. `X86ELFObjectWriter` 调用函数 `X86ELFObjectWriter::getRelocType()` llvm/lib/Target/X86/MCTargetDesc/X86ELFObjectWriter.cpp 确定最终的 relocation type。是根据 relocation `Specifier`, `MCFixupKind` 和 `IsPCRel` 查表确定的。

      对于本例，根据 relocation specifier 是 `X86::S_GOTPCREL`，`MCFixupKind` 为 `X86::reloc_riprel_4byte_movq_load`，确定最终的 relocation type 是 `ELF::R_X86_64_REX_GOTPCRELX`。

      如果将本例中的 `-fPIC` 修改为 `-fno-PIC`，最终生成的指令就是 `movabsq $g_var, %rax`。`MCFixupKind` 是 `FK_Data_8`，此时 relocation `Specifier` 是 `X86::S_None`，`IsPCRel` 为 false，，最终确定的 relocation type 是 `R_X86_64_64`。

      ```C++
      // clang -fno-PIC -c example.cpp -o example.o
      extern int g_var;
      long long get_g_var_addr() {
        return (long long)&g_var;
      }
      ```

## See Also

1. MarkRay 的博客 [Relocation overflow and code models | MaskRay](https://maskray.me/blog/2023-05-14-relocation-overflow-and-code-models)

2. Eli Bendersky 博客

   1. https://eli.thegreenplace.net/2012/01/03/understanding-the-x64-code-models

   2. https://eli.thegreenplace.net/2011/11/03/position-independent-code-pic-in-shared-libraries/

   3. https://eli.thegreenplace.net/2011/11/11/position-independent-code-pic-in-shared-libraries-on-x64/

## P.S.

1. Clang 默认启用 fPIE,-pie 和 x86 relax relocations

   - [[CMake] Default ENABLE_X86_RELAX_RELOCATIONS to ON](https://github.com/llvm/llvm-project/commit/c41a18cf61790fc898dcda1055c3efbf442c14c0)

   - [D120305 [Driver] Default CLANG_DEFAULT_PIE_ON_LINUX to ON](https://reviews.llvm.org/D120305)

2. 对于 x86-64，汇编上 `call/jmp foo` 和 `call/jmp foo@PLT` 都会生成 R_X86_64_PLT32 类型的 relocation https://github.com/llvm/llvm-project/commit/37f0c8df47d84ba311fc9a2c1884935ba8961e84

3. `main()` 函数中对 `foo()` 的调用原本是 经 PLT 调用，relocation type 也是 R_X86_64_PLT32。实际上在链接后最终的二进制中，变为了直接调用，即 PLT -> direct call。这个实现在 `RelocationScanner::processAux()` lld/ELF/Relocations.cpp

   ```C++
   int foo() { return 0; }
   int main() { return foo(); }
   ```

   > If non-ifunc non-preemptible, change PLT to direct call and optimize GOT indirection.
   >
   > preemptible: if a symbol can be replaced at load-time by a symbol with the same name defined in other ELF executable or DSO.

   R_X86_64_PLT32 -> R_X86_64_PC32 (R_PLT_PC -> R_PC)
