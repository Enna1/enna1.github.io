---
title: "HWASAN Internals"
date: 2023-04-12
categories:
  - "Programming"
tags:
- "Sanitizer"
- "LLVM"
comments: true # Enable Disqus comments for specific page
toc: true
---

本文深入分析了 HWASAN (HardWare-assisted AddressSanitizer) 检测内存错误的原理。

<!--more-->

## 概述

**HWASAN**: **H**ard**W**are-assisted **A**ddress**San**itizer, a tool similar to AddressSanitizer, but based on partial hardware assistance and consumes much less memory.

这里所谓的 "partial hardware assistance" 就是指 AArch64 的 **TBI** (Top Byte Ignore) 特性。

> TBI (Top Byte Ignore) feature of AArch64: bits [63:56] are ignored in address translation and can be used to store a tag.

以如下代码举例，Linux/AArch64 下将指针 x 的 top byte 设置为 0xfe，不影响程序执行：

```cpp
// $ cat tbi.cpp
int main(int argc, char **argv) {
  int * volatile x = (int *)malloc(sizeof(int));
  *x = 666;
  printf("address: %p, value: %d\n", x, *x);
  x = reinterpret_cast<int*>(reinterpret_cast<uintptr_t>(x) | (0xfeULL << 56));
  printf("address: %p, value: %d\n", x, *x);
  free(x);
  return 0;
}
// $ clang++ tbi.cpp && ./a.out
address: 0xaaab1845fe70, value: 666
address: 0xfe00aaab1845fe70, value: 666
```

AArch64 的 TBI 特性使得软件可以在 64-bit 虚拟地址的最高字节中存储任意数据，HWASAN 正是基于 TBI 这一特性设计并实现的内存错误检测工具。

举个例子，以下代码中存在 heap-buffer-overflow bug：

```cpp
// cat test.c
#include <stdlib.h>
int main() {
    int * volatile x = (int *)malloc(sizeof(int)*10);
    x[10] = 0;
    free(x);
}
```

使用 HWASAN 检测上述代码中的 heap-buffer-overflow bug：

```
$ clang -fuse-ld=lld -g -fsanitize=hwaddress ./test.c && ./a.out
==3581920==ERROR: HWAddressSanitizer: tag-mismatch on address 0xec2bfffe0028 at pc 0xaaad830db1a4
WRITE of size 4 at 0xec2bfffe0028 tags: 69/08(69) (ptr/mem) in thread T0
    #0 0xaaad830db1a4 in main ./test.c:4:11
    #1 0xfffd07350da0 in __libc_start_main libc-start.c:308:16
    #2 0xaaad83090820 in _start (./a.out+0x40820)

[0xec2bfffe0000,0xec2bfffe0030) is a small allocated heap chunk; size: 48 offset: 40

Cause: heap-buffer-overflow
0xec2bfffe0028 is located 0 bytes after a 40-byte region [0xec2bfffe0000,0xec2bfffe0028)
allocated by thread T0 here:
    #0 0xaaad83099248 in __sanitizer_malloc.part.13 llvm-project/compiler-rt/lib/hwasan/hwasan_allocation_functions.cpp:151:3
    #1 0xaaad830db17c in main ./test.c:3:31
    #2 0xfffd07350da0 in __libc_start_main libc-start.c:308:16
    #3 0xaaad83090820 in _start (/a.out+0x40820)

Thread: T0 0xeffc00002000 stack: [0xffffc3a10000,0xffffc4210000) sz: 8388608 tls: [0xfffd076a5030,0xfffd076a5e70)
Memory tags around the buggy address (one tag corresponds to 16 bytes):
  0xec2bfffdf800: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffdf900: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffdfa00: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffdfb00: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffdfc00: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffdfd00: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffdfe00: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffdff00: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
=>0xec2bfffe0000: 69  69 [08] 00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffe0100: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffe0200: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffe0300: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffe0400: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffe0500: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffe0600: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffe0700: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
  0xec2bfffe0800: 00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
Tags for short granules around the buggy address (one tag corresponds to 16 bytes):
  0xec2bfffdff00: ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..
=>0xec2bfffe0000: ..  .. [69] ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..
  0xec2bfffe0100: ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..  ..
See https://clang.llvm.org/docs/HardwareAssistedAddressSanitizerDesign.html#short-granules for a description of short granule tags
Registers where the failure occurred (pc 0xaaad830db1a4):
    x0  a100ffffc4201580  x1  6900ec2bfffe0028  x2  0000000000000000  x3  0000000000000000
    x4  0000000000000020  x5  0000000000000000  x6  0000000000100000  x7  fffffffffff00005
    x8  6900ec2bfffe0000  x9  6900ec2bfffe0000  x10 0030f15d14c79f97  x11 00ffffffffffffff
    x12 00001f0d780b69d2  x13 0000000000000001  x14 0000ffffc4200b60  x15 0000000000000696
    x16 0000aaad830a3540  x17 000000000000000b  x18 0000000000000100  x19 0000aaad830db600
    x20 0200effd00000000  x21 0000aaad830907f0  x22 0000000000000000  x23 0000000000000000
    x24 0000000000000000  x25 0000000000000000  x26 0000000000000000  x27 0000000000000000
    x28 0000000000000000  x29 0000ffffc4201590  x30 0000aaad830db1a8   sp 0000ffffc4201550
SUMMARY: HWAddressSanitizer: tag-mismatch ./test.c:4:11 in main
```

如上所示，HWASAN 与 ASAN 相比不管是用法 (`-fsanitize=hwaddress` v.s. `-fsanitize=address`) 还是检测到错误后的报告都很相似。

下面对比分析 ASAN 与 HWASAN 检测内存错误的技术原理：

ASAN (AddressSanitizer)：

- 使用 shadow memory 技术，每 8-bytes 的 application memory 对应 1-byte 的 shadow memory。

- 使用 redzone 来检测 buffer-overflow。不管是栈内存还是堆内存，在申请内存时都会在原本内存的两侧额外申请一定大小的内存作为 redzone，一旦访问到了 redzone 则说明发生了缓冲区溢出。

- 使用 quarantine 检测 use-after-free。应用程序执行 delete 或 free 释放内存时，并不真正释放，而是放在一个暂存区 (quarantine) 中，一旦访问了位于 quarantine 中的内存则说明访问了已释放的内存。

- 每 1-byte 的 shadow memory 编码表示对应的 8-byte application memory 的信息，每次访问 application memory 之前 ASAN 都会检查对应的 shadow memory 判断本次内存访问是否合法。例如：shadow memory 的值为 0xfd 表示对应的 8-bytes application memory 是 freed memory，所以当访问的 application memory 其 shadow memory byte 为 0xfd 就说明此时访问的是已经释放的内存了即 use-after-free；shadow memory 为 0xfa 表示相应的 application memory 是堆内存的 redzone，所以当访问到的 appllcation memory 其 shadow memory 为 0xfa 就说明此时访问的是堆内存附近的 redzone 发生了堆缓冲区溢出错误即 heap-buffer-overflow

HWASAN (HardWare-assisted AddressSanitizer)

- 同样使用 shadow memory 技术，不过与 ASAN 不同的是：HWASAN 每 16-bytes 的 application memory 对应 1-byte 的 shadow memory。

- 不依赖 redzone 检测 buffer-overflow，不依赖 quarantine 检测 use-after-free，仅基于 TBI 特性就能检测 buffer-overflow 和 use-after-free。

- 举例说明 HWASAN 检测内存错误的原理

  因为 HWASAN 会将每 16-bytes 的 application memory 都对应 1-byte 的 shadow tag，所以 HWASAN 会将申请的内存都对齐到 16-bytes，因此下图中 new char[20] 实际申请的内存是 32-bytes。

  HWASAN 会生成一个随机 tag 保存在 operator new 返回的指针 p 的 top byte 中，同时会将 tag 保存在 p 指向的内存对应 shadow memory 中。

  为了方便说明，下图中用不同的颜色表示不同的 tag，绿色表示 tag 0xa，蓝色表示 tag 0xb，紫色表示 tag 0xc。

  - 检测 heap-buffer-overflow

    假设 HWASAN 为 `new char[20]` 生成的 tag 为 0xa 即绿色，所以指针 p 的 top byte 为 0xa。在通过 `p[32]` 访问内存时，HWASAN 会检查保存在指针 p 的 tag 与 `p[32]` 指向的内存所对应的 shadow memory 中保存的 tag 是否一致。显然保存在指针 p 的 tag 是绿色 而`p[32]` 指向的内存所对应的 shadow memory 中保存的 tag 是蓝色，即 tag 是不匹配的，这说明访问 `p[32]` 时存在内存错误。

    ![](/blog/hwasan-internals/2023-04-08-18-31-23-image.PNG)

  - 检测 use-after-free

    假设 HWASAN 为 `new char[20]` 生成的 tag 为 0xa 即绿色，所以指针 p 的 top byte 为 0xa。执行 `delete[]  p` 释放内存时，HWASAN 将这块释放的内存 retag 为紫色，即将这块释放的内存对应的 shadow memory 从绿色修改为紫色。在通过 `p[0]` 访问内存时，HWASAN 会检查保存在指针 p 的 tag 与 `p[0]` 指向的内存所对应的 shadow memory 中保存的 tag 是否一致。显然保存在指针 p 的 tag 是绿色 而`p[0]` 指向的内存所对应的 shadow memory 中保存的 tag 是紫色，即 tag 是不匹配的，这说明访问 `p[0]` 时存在内存错误。

    ![](/blog/hwasan-internals/2023-04-08-18-30-56-image.png)

## 算法

- shadow memory：每 16-bytes 的 application memory 对应 1-byte 的 shadow memory。

- 每个 heap/stack/global 内存对象都被对齐到 16-bytes，这样每个 heap/stack/global 内存对象至少对应的 1-byte shadow memory。

- 为每一个 heap/stack/global 内存对象生成一个 1-byte 的随机 tag，将该随机 tag 保存到指向这些内存对象的指针的 top byte 中，同样将该随机 tag 保存到这些内存对象对应的 shadow memory 中。

- 在每一处内存读写之前插桩：比较保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 是否一致，如果不一致则报错。

## 实现

### shadow mapping

HWASAN 与 ASAN 一样都使用了 shadow memory 技术。ASAN 默认使用 static shadow mapping，只有对 IOS 和 32-bit Android 平台才使用 dynamic shadow mapping。而 HWASAN 则总是使用 dynamic shadow mapping。

- ASAN: static shadow mapping。在 llvm-project/compiler-rt/lib/asan/asan_mapping.h 中预定义了不同平台下 shadow memory 的布局：HighMem, HighShadow, ShadowGap, LowShadow, LowMem 的地址区间。

  Linux/x86_64 下 ASAN 的 shadow mapping 如下所示：

  ```cpp
  // Typical shadow mapping on Linux/x86_64 with SHADOW_OFFSET == 0x00007fff8000:
  || `[0x10007fff8000, 0x7fffffffffff]` || HighMem    ||
  || `[0x02008fff7000, 0x10007fff7fff]` || HighShadow ||
  || `[0x00008fff7000, 0x02008fff6fff]` || ShadowGap  ||
  || `[0x00007fff8000, 0x00008fff6fff]` || LowShadow  ||
  || `[0x000000000000, 0x00007fff7fff]` || LowMem     ||
  ```

  给定 application memory 地址 `addr`，计算其对应的 shadow memory 地址的公式如下：

  ```cpp
  uptr MemToShadow(uptr addr) { return (addr >> 3) + 0x7fff8000; }
  ```

- HWASAN: dynamic shadow mapping。根据 MaxUserVirtualAddress 计算 shadow memory 所需要的总大小 shadow_size，通过 mmap(shadow_size) 得到 shadow memory 区间，再具体划分 HighMem, HighShadow, ShadowGap, LowShadow, LowMem 的地址区间。

  伪算法如下（未考虑对齐）：

  ```cpp
  kHighMemEnd = GetMaxUserVirtualAddress();
  shadow_size = MemToShadowSize(kHighMemEnd);
  __hwasan_shadow_memory_dynamic_address = mmap(shadow_size);
  // Place the low memory first.
  kLowMemEnd = __hwasan_shadow_memory_dynamic_address - 1;
  kLowMemStart = 0;
  // Define the low shadow based on the already placed low memory.
  kLowShadowEnd = MemToShadow(kLowMemEnd);
  kLowShadowStart = __hwasan_shadow_memory_dynamic_address;
  // High shadow takes whatever memory is left up there.
  kHighShadowEnd = MemToShadow(kHighMemEnd);
  kHighShadowStart = Max(kLowMemEnd, MemToShadow(kHighShadowEnd)) + 1;
  // High memory starts where allocated shadow allows.
  kHighMemStart = ShadowToMem(kHighShadowStart);
  ```

  ```cpp
  uptr MemToShadow(uptr untagged_addr) {
    return (untagged_addr >> 4) + __hwasan_shadow_memory_dynamic_address;
  }
  uptr ShadowToMem(uptr shadow_addr) {
    return (shadow_addr - __hwasan_shadow_memory_dynamic_address) << 4;
  }
  ```

  Linux/AArch64 下 HWASAN 的某种 shadow mapping 如下所示：

  ```cpp
  // Typical mapping on Linux/AArch64
  // with dynamic shadow mapped: [0xefff00000000, 0xffff00000000]:
  || [0xffff00000000, 0xffffffffffff] || HighMem ||
  || [0xfffef0000000, 0xfffeffffffff] || HighShadow ||
  || [0xfefef0000000, 0xfffeefffffff] || ShadowGap ||
  || [0xefff00000000, 0xfefeefffffff] || LowShadow ||
  || [0x000000000000, 0xeffeffffffff] || LowMem ||
  ```

### tagging

- stack 内存对象：在编译时插桩阶段，HWASAN 会插入代码来实现对 stack 内存对象的 tagging，将生成的随机 tag 保存在 stack 内存对象指针的 top byte，同时将随机 tag 保存这些 stack 内存对象对应的 shadow memory 中。

  每个 stack 内存对象的 tag 是通过 `stack_base_tag ^ RetagMask(AllocaNo)` 计算得到的。`stack_base_tag` 对于不同的 stack frame 是不同的值，`RetagMask(AllocaNo)` 的实现如下（AllocaNo 可以看作是 stack 内存对象的序号）。

  ```cpp
    static unsigned RetagMask(unsigned AllocaNo) {
    // A list of 8-bit numbers that have at most one run of non-zero bits.
    // x = x ^ (mask << 56) can be encoded as a single armv8 instruction for these
    // masks.
    // The list does not include the value 255, which is used for UAR.
    //
    // Because we are more likely to use earlier elements of this list than later
    // ones, it is sorted in increasing order of probability of collision with a
    // mask allocated (temporally) nearby. The program that generated this list
    // can be found at:
    // https://github.com/google/sanitizers/blob/master/hwaddress-sanitizer/sort_masks.py
    static unsigned FastMasks[] = {0,  128, 64,  192, 32,  96,  224, 112, 240,
                                   48, 16,  120, 248, 56,  24,  8,   124, 252,
                                   60, 28,  12,  4,   126, 254, 62,  30,  14,
                                   6,  2,   127, 63,  31,  15,  7,   3,   1};
    return FastMasks[AllocaNo % std::size(FastMasks)];
  }
  ```

- heap 内存对象：随机 tag 是在运行时申请 heap 内存对象时由 HWASAN 的内存分配器生成的。

  当程序调用 malloc 或者 operator new 申请内存时，HWASAN 的内存分配器会将随机生成的 tag 保存在 malloc 或者 operator new 返回的指针的 top byte 中，同时将随机 tag 保存在这些 heap 内存对象对应的 shadow memory 中。

  当程序调用 free 或者 operator delete 释放内存时，HWASAN 的内存分配器会再次随机生成 tag ，将新生成的随机 tag 保存在正释放的 heap 内存对象对应的 shadow memory 中。

- global 内存对象：对于每个编译单元内的 global 内存对象，在编译时插桩阶段都会生成对应的 tag，编译单元内的第一个 global 内存对象的 tag 是对当前编译单元文件路径计算 MD5 哈希值然后取第一个字节得到的，编译单元内的后续 global 内存对象的 tag 是基于之前 global 内存对象的 tag 递增得到的。

  在编译时插桩阶段 HWASAN 会插入代码来将生成的随机 tag 保存在 global 内存对象指针的 top byte，而将随机 tag 保存到这些 global 内存对象对应的 shadow memory 中则是由 HWASAN runtime 在程序启动阶段做的。对于每一个 global 内存对象，HWASAN 在编译插桩阶段都会创建一个 descriptor 保存在 "hwasan_globals" section 中，descriptor 中保存了 global 内存对象的大小以及 tag 等信息，程序启动时 HWASAN runtime 会遍历 "hwasan_globals" section 中所有的 descriptor 来设置每个 global 内存对象对应的 shadow memory tag。

### short granules

每个 heap/stack/global 内存对象都会被对齐到 16-bytes，heap/stack/global 内存对象原本大小记作 size，如果 `size % 16 != 0`，那么就需要 padding，heap/stack/global 内存对象最后不足 16-bytes 的部分就被称为 short granule。此时会将 tag 存储到 padding 的最后一个字节，而 padding 所在的 16-bytes 内存对应的 1-byte shadow memory 中存储的则是 short granule size 即 `size % 16`。

举例如下：

```cpp
uint8_t buf[20];
```

`uint8_t buf[20]` 开启 HWASAN 后会变为：

```cpp
uint8_t buf[32]; // 20-bytes aligned to 16-bytes -> 32-bytes
uint8_t tag = __hwasan_generate_tag();
buf[31] = tag;
*(char *)MemToShadow(buf) = tag;
*((char *)MemToShadow(buf)+1) = 20 % 16;
uint8_t *tagged_buf = reinterpret_cast<int8_t *>(
                     reinterpret_cast<uintptr_t>(buf) | (tag << 56));
// Replace all uses of `buf` with `tagged_buf`
```

`uint_t buf[20]` 的最后 4-bytes 就是 short granule，short granule size 即 20 % 16 = 4。

因为 short granules 的存在，所以在比较保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 是否一致时，需要考虑如下两种可能：

1. 保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 相同。

2. 保存在 shadow memory 中的 tag 实际上是 short granule size，保存在指针 top byte 的 tag 等于保存在指针指向的内存所在的 16-bytes 内存的最后一个字节的 tag。

![](/blog/hwasan-internals/2023-04-12-10-32-16-image.png)

为什么需要 short granules ？

考虑 `uint8_t buf[20]`，假设代码中存在访问 `buf[22]` 导致的 buffer-overflow。因为 HWASAN 会将 heap/stack/global 内存对象对齐到 16-bytes，所以实际为 `uint8_t buf[20]` 申请的空间是 `uint8_t buf[32]`。

如果没有 short granules，那么保存在 `buf` 指针 top byte 的 tag 为 0xa1，保存在 `buf[22]` 对应的 shadow memory 中的 tag 为 0xa1，尽管访问 `buf[22]` 时发生了 buffer-overflow，此时 HWASAN 也检测不到，因为保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 是否一致的。

有了 short granules，保存在 `buf` 指针 top byte 的 tag 为 0xa1，保存在 `buf[22]` 对应的 shadow memory 中的 tag 则是 short granule size 即 20 % 16 = 4。访问 `buf[22]` 时，HWASAN 发现保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 不一致，保存在 `buf[22]` 对应的 shadow memory 中的 tag 是 short granule size 为 4，这意味着 `buf[22]` 所在的 16-bytes 内存只有前 4-bytes 是可以合法访问的，而 `buf[22]` 访问的却是其所在 16-bytes 内存的第 7 个 byte，说明访问 `buf[22]` 时发生了 buffer-overflow！

### hwasan check memaccess

本节说明 HWASAN 如何在每一处内存读写之前通过插桩来比较保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 是否一致的。

开启 HWASAN 后，HWAddressSanitizer instrumentation pass 会在 LLVM IR 层面进行插桩。默认情况下，HWASAN 在每一处内存读写之前添加对 `llvm.hwasan.check.memaccess.shortgranules` intrinsic 的调用。该 `llvm.hwasan.check.memaccess.shortgranules` intrinsic 会在生成汇编代码时转换为对相应函数的调用。

还是以如下代码为例说明：

```cpp
// cat test.c
#include <stdlib.h>
int main() {
    int * volatile x = (int *)malloc(sizeof(int)*10);
    x[10] = 0;
    free(x);
}
```

上述代码 `clang -O1 -fsanitize=hwaddress test.c -S -emit-llvm` 开启 HWASAN 生成的 LLVM IR 如下：

```llvm
%5 = alloca ptr, align 8
%6 = call noalias ptr @malloc(i64 noundef 40)
store ptr %6, ptr %5, align 8
%7 = load volatile ptr, ptr %5, align 8
%8 = getelementptr inbounds i32, ptr %7, i64 10
call void @llvm.hwasan.check.memaccess.shortgranules(ptr %__hwasan_shadow_memory_dynamic_address, ptr %8, i32 18)
store i8 0, ptr %8, align 4
%9 = load volatile ptr, ptr %5, align 8
call void @free(ptr noundef %9)
```

`llvm.hwasan.check.memaccess.shortgranules` intrinsic 有三个参数：

1. __hwasan_shadow_memory_dynamic_address

2. 本次 memory access 访问的内存地址

3. 常数 AccessInfo，编码了本次 memory access 的相关信息。计算公式如下：

   ```cpp
   int64_t HWAddressSanitizer::getAccessInfo(bool IsWrite,
                                             unsigned AccessSizeIndex) {
     return (CompileKernel << HWASanAccessInfo::CompileKernelShift) |
            (MatchAllTag.has_value() << HWASanAccessInfo::HasMatchAllShift) |
            (MatchAllTag.value_or(0) << HWASanAccessInfo::MatchAllShift) |
            (Recover << HWASanAccessInfo::RecoverShift) |
            (IsWrite << HWASanAccessInfo::IsWriteShift) |
            (AccessSizeIndex << HWASanAccessInfo::AccessSizeShift);
   }

   // Bit field positions for the accessinfo parameter to
   // llvm.hwasan.check.memaccess. Shared between the pass and the backend. Bits
   // 0-15 are also used by the runtime.
   enum {
     AccessSizeShift = 0, // 4 bits
     IsWriteShift = 4,
     RecoverShift = 5,
     MatchAllShift = 16, // 8 bits
     HasMatchAllShift = 24,
     CompileKernelShift = 25,
   };
   ```

   - IsWrite 布尔值，0 表示本次 memory access 是读操作，1 表示本次 memory access 为 写操作。

   - AccessSizeIndex 由 `__builtin_ctz(AccessSize)` 计算得到。AccessSize 为 1-byte, 2-bytes, 4-bytes, 8-bytes, 16-bytes 时，AccessSizeIndex 分别为 0, 1, 2, 3, 4。

   - CompileKernel 布尔值，只有 HWASAN 用于 [KernelHWAddressSanitizer](https://lwn.net/Articles/763684/) 时 CompileKernel 值才为 1。

   - MatchAllTag 类型为 uint8_t，MatchAllTag 的默认值为 -1，表示没有设置 MatchAllTag。可以通过编译时选项 `-mllvm -hwasan-match-all-tag=-1` 进行设置。当保存在指针 top byte 的 tag 值 和保存在 shadow memory 中的 tag 不匹配时说明 HWASAN 检测到的内存错误，如果设置了 MatchAllTag，那么 HWASAN 会忽略所有 pointer tag 为 MatchAllTag 时出现的 tag mismatch。

   - Recover 布尔值，0 表示 HWASAN 检测到错误不再继续执行程序，1 表示 HWASAN 检测到错误后继续执行。Recover 默认为 0，在编译时添加参数 `-fsanitize-recover=hwaddress` 后 Recover 为 1。

上述例子 LLVM IR 中，调用 `llvm.hwasan.check.memaccess.shortgranules`  的第一参数就是 `%__hwasan_shadow_memory_dynamic_address`，第二个参数是 `%8` 对应源码中的 `x[10]` 的地址，第三个参数是常数 `18` 表示本次内存访问是写入 4-byte 的操作。

---

`llvm.hwasan.check.memaccess.shortgranules` intrinsic 在 AsmPrinter 阶段被转换为汇编代码。对于 AArch64 后端，相关的函数为 `LowerHWASAN_CHECK_MEMACCESS()`, `emitHwasanMemaccessSymbols()`，代码位于 llvm/lib/Target/AArch64/AArch64AsmPrinter.cpp。

`llvm.hwasan.check.memaccess.shortgranules` intrinsic 会根据参数不同而生成不同的汇编函数。例如上述例子中 `call void @llvm.hwasan.check.memaccess.shortgranules(ptr %__hwasan_shadow_memory_dynamic_address, ptr %8, i32 18)` 生成的汇编符号/函数名为 `__hwasan_check_x0_18_short_v2`：

- x0 表示本次 memory access 访问的内存地址保存在 X0 寄存器中

- 18 即本次 memory access 的 AccessInfo 的值

另外，AArch64AsmPrinter 在将 `llvm.hwasan.check.memaccess.shortgranules` intrinsic 转换为汇编代码时，总是将 __hwasan_shadow_memory_dynamic_address 即 shadow base 保存在 X20 寄存器中。

`__hwasan_check_x0_18_short_v2` 完整的汇编代码如下：

```
__hwasan_check_x0_18_short_v2:
  sbfx    x16, x0, #4, #52    // shadow offset
  ldrb    w16, [x20, x16]     // load shadow tag
  cmp     x16, x0, lsr #56    // extract address tag, compare with shadow tag
  b.ne    .Ltmp0              // jump to short tag handler on mismatch
.Ltmp1:
  ret
.Ltmp0:
  cmp     w16, #15            // is this a short tag?
  b.hi    .Ltmp2              // if not, error
  and     x17, x0, #0xf       // find the address's position in the short granule
  add     x17, x17, #3        // adjust to the position of the last byte loaded
  cmp     w16, w17            // check that position is in bounds
  b.ls    .Ltmp2              // if not, error
  orr     x16, x0, #0xf       // compute address of last byte of granule
  ldrb    w16, [x16]          // load tag from it
  cmp     x16, x0, lsr #56    // compare with pointer tag
  b.eq    .Ltmp1              // if matches, continue
.Ltmp2:
  // save original x0, x1 on stack (they will be overwritten)
  stp     x0, x1, [sp, #-256]!
  // create frame record
  stp     x29, x30, [sp, #232]
  // set x1 to a constant indicating the type of failure
  mov     x1, #18
  // call runtime function to save remaining registers and report error
  adrp    x16, :got:__hwasan_tag_mismatch_v2
  // (load address from GOT to avoid potential register clobbers in delay load handler)
  ldr     x16, [x16, :got_lo12:__hwasan_tag_mismatch_v2]
  br      x16
```

---

前面内容提到，HWASAN 在编译插桩时，默认情况下是在每一处内存读写之前添加对 `llvm.hwasan.check.memaccess.shortgranules` intrinsic 的调用，那非默认情况呢？

- 如果在编译时添加选项 `-mllvm -hwasan-instrument-with-calls=true`，那么编译插桩时 HWASAN 在每一处内存读写之前添加的则是对 `__hwasan_[load|store][1|2|4|8|16|n]` 的函数调用。

  还是以本文开头的示例代码进行说明，`clang -O1 -fsanitize=hwaddress -mllvm -hwasan-instrument-with-calls=true test.c -S -emit-llvm` 生成的 LLVM IR 如下：

  ```llvm
    %2 = alloca ptr, align 8
    %3 = call noalias ptr @malloc(i64 noundef 40)
    store volatile ptr %3, ptr %2, align 8
    %4 = load volatile ptr, ptr %2, align 8
    %5 = getelementptr inbounds i32, ptr %4, i64 10
    %6 = ptrtoint ptr %5 to i64
    call void @__hwasan_store4(i64 %6)
    store i32 0, ptr %5, align 4
    %7 = load volatile ptr, ptr %2, align 8
    call void @free(ptr noundef %7)
  ```

  `__hwasan_[load|store][1|2|4|8|16|n]` 函数由 HWASAN runtime 中实现，代码位于 compiler-rt/lib/hwasan/hwasan.cpp。根据本次 memory access 为读操作或写操作，调用 `___hwasan_load` 或 `___hwasan_store`；根据本次 memory access 读写的内存大小（字节），调用相应的版本，如本次 memory acesss 是对 4-bytes 内存的写操作，HWASAN 插入的是就对 `___hwasan_store4` 的调用。

  `__hwasan_[load|store][1|2|4|8|16|n]` 函数的实现几乎一致，下面给出 `___hwasan_store4` 的代码实现：

  ```cpp
  void __hwasan_store4(uptr p) {
    CheckAddress<ErrorAction::Abort, AccessType::Store, 2>(p);
  }

  template <ErrorAction EA, AccessType AT, unsigned LogSize>
  __attribute__((always_inline, nodebug)) static void CheckAddress(uptr p) {
    if (!InTaggableRegion(p))
      return;
    uptr ptr_raw = p & ~kAddressTagMask;
    tag_t mem_tag = *(tag_t *)MemToShadow(ptr_raw);
    if (UNLIKELY(!PossiblyShortTagMatches(mem_tag, p, 1 << LogSize))) {
      SigTrap<EA, AT, LogSize>(p);
      if (EA == ErrorAction::Abort)
        __builtin_unreachable();
    }
  }

  __attribute__((always_inline, nodebug)) static inline bool
  PossiblyShortTagMatches(tag_t mem_tag, uptr ptr, uptr sz) {
    DCHECK(IsAligned(ptr, kShadowAlignment));
    tag_t ptr_tag = GetTagFromPointer(ptr);
    if (ptr_tag == mem_tag)
      return true;
    if (mem_tag >= kShadowAlignment)
      return false;
    if ((ptr & (kShadowAlignment - 1)) + sz > mem_tag)
      return false;
    return *(u8 *)(ptr | (kShadowAlignment - 1)) == ptr_tag;
  }
  ```

- 如果在编译时添加选项 `-mllvm -hwasan-inline-all-checks=true`，那么编译插桩时 HWASAN 会直接在 LLVM IR 上实现检查保存在指针 top byte 的 tag 和保存在 shadow memory 中的 tag 是否匹配的逻辑。

  还是以本文开头的示例代码进行说明，`clang -O1 -fsanitize=hwaddress -mllvm -hwasan-inline-all-checks=true test.c -S -emit-llvm` 生成的 LLVM IR 如下：

  ```llvm
    %5 = alloca ptr, align 8
    %6 = call noalias  ptr @malloc(i64 noundef 40)
    store volatile ptr %6, ptr %5, align 8
    %7 = load volatile ptr, ptr %5, align 8
    %8 = getelementptr inbounds i8, ptr %7, i64 30
    %9 = ptrtoint ptr %8 to i64
    %10 = lshr i64 %9, 56
    ; extrac pointer tag
    %11 = trunc i64 %10 to i8
    %12 = and i64 %9, 72057594037927935
    ; shadow offset
    %13 = lshr i64 %12, 4
    ; shadow base + shadow offset
    %14 = getelementptr i8, ptr %4, i64 %13
    ; load shadow tag
    %15 = load i8, ptr %14, align 1
    ; compare pointer tag with shadow tag
    %16 = icmp ne i8 %11, %15
    ; jump to short tag handler on mismatch
    br i1 %16, label %17, label %31

  17:                                               ; preds = %0
    ; is this a short tag?
    %18 = icmp ugt i8 %15, 15
    ; if not, error
    br i1 %18, label %19, label %20

  19:                                               ; preds = %25, %20, %17
    call void asm sideeffect "brk #2320", "{x0}"(i64 %9)
    unreachable

  20:                                               ; preds = %17
    ; find the address's position in the short granule
    %21 = and i64 %9, 15
    %22 = trunc i64 %21 to i8
    ; adjust to the position of the last byte loaded
    %23 = add i8 %22, 3
    ; check that position is in bounds
    %24 = icmp uge i8 %23, %15
    ; if not, error
    br i1 %24, label %19, label %25

  25:                                               ; preds = %20
    ; compute address of last byte of granule
    %26 = or i64 %12, 15
    %27 = inttoptr i64 %26 to ptr
    ; load tag from it
    %28 = load i8, ptr %27, align 1
    ; compare with pointer tag
    %29 = icmp ne i8 %11, %28
    ; if mismatche, error
    br i1 %29, label %19, label %30

  30:                                               ; preds = %25
    br label %31

  31:                                               ; preds = %0, %30
    store i8 0, ptr %8, align 4
    %32 = load volatile ptr, ptr %5, align 8
    call void @free(ptr noundef %32)
  ```

## 参考链接

- [Hardware-assisted AddressSanitizer Design Documentation](https://clang.llvm.org/docs/HardwareAssistedAddressSanitizerDesign.html)
- https://github.com/google/sanitizers/tree/master/hwaddress-sanitizer
