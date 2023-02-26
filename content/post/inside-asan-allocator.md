---
title: "Inside AddressSanitizer Allocator"
date: 2023-02-26
categories:
  - "Programming"
tags:
- "Sanitizer"
- "LLVM"
katex: false
comments: true # Enable Disqus comments for specific page
toc: true
---

本文深入分析了 AddressSanitizer Allocator 的实现。

<!--more-->

AddressSanitizer 通过 intercept malloc/free, new/delete 的调用，使得开启 ASan 后程序的内存分配和释放都由 ASan allocator 负责，本文深入分析了 AddressSanitizer Allocator 的实现。

ASan allocator 与其他 allocator 不同的点：

- 为了检测 heap-buffer-overflow，ASan allocator 在分配内存时会在用户申请的内存左右两侧多申请内存作为 redzone，这样一旦访问到 redzone 就说明发生了溢出，这样 ASan 就检测到了 heap-buffer-overflow。

- 为了检测 heap-use-after-free, double-free，程序调用 free/delete 释放内存时，ASan allocator 并不会真正释放内存，而是先将其放入暂存区 quarantine，当 quarantine 满了以后才会去真正释放以前被放入 quarantine 的内存。这样一旦访问了已释放的内存，只要该已释放的内存还处于 quarantine 中，ASan 就能检测到发生了 heap-use-after-free。

## Memory Chunk

每当调用 malloc 或 new 申请一块内存时，ASan allocator 真正申请的内存大小是比用户请求的内存大小更大的。这是因为：一方面需要额外的内存来保存一些信息，如：用户申请内存的大小、申请内存时的 stacktrace 等；另一方面也需要在用户申请的内存左右两侧多申请内存作为 redzone，用于检测 heap-buffer-overflow bug。

我们将 ASan allocator 实际申请的内存块称为 memory chunk。由 ASan allocator 分配得到的 memory chunk 的布局如下所示：

```
 L L L L L L H H U U U U U U R R
   L -- left redzone words (0 or more bytes)
   H -- ChunkHeader (16 bytes), which is also a part of the left redzone.
   U -- user memory.
   R -- right redzone (0 or more bytes)
```

如上所示，memory chunk 中返回给用户使用的内存称为 user memory。ASan allocator 额外申请的紧邻 user memory 左右两侧的额外内存就是 redzone。

- left redzone 的最后 16 bytes 同时作为 ChunkHeader，存储一些当前 chunk 的状态，如：用户申请的内存大小、申请内存时的 stacktrace 等。因此 left redzone 最小为 16-bytes，即 ChunkHeader 的大小。

- right redzone 最小为 0-bytes，即存在没有 right redzone 的情况。具体什么情况需要 right redzone、什么情况不需要 right redzone，本文靠后内容会详细说明。

当 left redzone 的大小大于 ChunkHeader 的大小（即 16-bytes）时，ASan allocator 会将 magic value: kAllocBegMagic 存储到 memory chunk 的第一个 8-bytes 中，将 ChunkHeader 的地址保存在 memory chunk 的第二个 8-bytes 中：

```
 M B L L L L L L L L L  H H U U U U U U
   |                    ^
   ---------------------|
   M -- magic value kAllocBegMagic
   B -- address of ChunkHeader pointing to the first 'H'
```

ChunkHeader 的代码实现如下：

```cpp
class ChunkHeader {
  atomic_uint8_t chunk_state;
  u8 alloc_type : 2;
  u8 lsan_tag : 2;

  // align < 8 -> 0
  // else      -> log2(min(align, 512)) - 2
  u8 user_requested_alignment_log : 3;

  u16 user_requested_size_hi;
  u32 user_requested_size_lo;
  atomic_uint64_t alloc_context_id;
};

static const uptr kChunkHeaderSize = sizeof(ChunkHeader);
COMPILER_CHECK(kChunkHeaderSize == 16);
```

- chunk_state 顾名思义表示 chunk 的状态，有 3 种可能取值：
  
  - CHUNK_INVALID: either just allocated by underlying allocator, but chunk is not yet ready, or almost returned to undelying allocator and chunk is already meaningless.
  
  - CHUNK_ALLOCATED: the chunk is allocated and not yet freed.
  
  - CHUNK_QUARANTINE: the chunk was freed and put into quarantine zone.

- alloc_type 有 3 种可能取值：
  
  - FROM_MALLOC：表示当前 chunk 是通过 `malloc`, `calloc`, `realloc` 申请的
  
  - FROM_NEW：表示当前 chunk 是通过 `operator new` 申请的
  
  - FROM_NEW_BR：表示当前 chunk 是通过 `operator new[]` 申请的

- lsan_tag 是 LeakSanitizer 用的，本文不涉及

- user_requested_alignment_log 用于表示用户申请的内存指定的对齐 alignment

- user_requested_size_hi, user_requested_size_lo 用于表示用户申请的内存大小 size

- alloc_context_id 保存的是申请内存的线程 id 和 stacktrace。关于 sanitizer 是是如何获取和保存 stack trace 的，可以参考 [How Sanitizer Get Stack Trace - Enna1's website](https://enna1.github.io/post/how-sanitizer-get-stacktrace/)

---

当应用程序申请一块内存时，ASan allocator 需要计算为 user memory 添加 redzone 后所需要的内存大小 needed_size，伪代码如下：

```cpp
uptr ComputeNeededSize(uptr size, uptr alignment) {
  const uptr min_alignment = ASAN_SHADOW_GRANULARITY;
  if (alignment < min_alignment)
      alignment = min_alignment;
  uptr rz_log = ComputeRZLog(size);
  uptr rz_size = RZLog2Size(rz_log);
  uptr rounded_size = RoundUpTo(Max(size, kChunkHeader2Size), alignment);
  uptr needed_size = rounded_size + rz_size;
  if (alignment > min_alignment)
      needed_size += alignment;
  if (!PrimaryAllocator::CanAllocate(needed_size, alignment))
      needed_size += rz_size;
  return needed_size;
}
```

1. 首先根据用户申请的内存大小计算所需的 redzone 大小 rz_size。基本逻辑是用户申请的内存越大，则对应的 redzone 就越大。并且如前所述， rz_size 最小就是 ChunkHeader 的大小 16-bytes。

2. 计算 rounded_size 是将 `Max(size, kChunkHeader2Size)` 对齐至 alignment，这是因为：对于已经释放的内存，ASan 需要像用 `atomic_uint64_t alloc_context_id` 保存申请内存的线程 id 和 stacktrace 一样，使用 `atomic_uint64_t free_context_id` 记录释放内存的线程 id 和 stacktrace。通过 `Max(size, kChunkHeader2Size)` 使得 chunk 中 user memory 的大小至少为 kChunkHeader2Size，这样 ASan 就可以通过复用 user memory 来保存 free_context_id 了。
   
   ```cpp
   class ChunkBase : public ChunkHeader {
     atomic_uint64_t free_context_id;
   };
   
   static const uptr kChunkHeader2Size = sizeof(ChunkBase) - kChunkHeaderSize;
   COMPILER_CHECK(kChunkHeader2Size <= 16);
   ```

3. `if (!PrimaryAllocator::CanAllocate()) needed_size += rz_size;` ASan allocator 由 PrimaryAllocator 和 SecondaryAllocator 组成，优先从 PrimaryAllocator 中分配内存。由 PrimaryAllocator 分配的 chunk 只有 left redzone 没有 right redzone，这是因为 PrimaryAllocator 保证相同大小的 chunk 是相邻分布的，所以对于当前 chunk，可以使用相邻的右侧 chunk 的 left redzone 作为当前 chunk 的 right redzone。而 SecondaryAllocator 则没有这个保证，所以需要额外申请 rz_size 大小内存作为 right redzone。

## Size Class

应用程序在申请内存时，ASan allocator 首先会计算添加 redzone 后所需要的内存大小 needed_size，如果小于等于 128KB，那么会由 PrimaryAllocator 进行分配。PrimaryAllocator 将分配的内存大小划分为了 53 个类别，称为 **size class**，每个 size class 都对应一个大小，比如 16-bytes, 32-bytes, 48-bytes。PrimaryAllocator 在真正申请内存时会 needed_size 向上取整到 size class 的大小，例如应用程序申请 8-bytes 的内存，添加 redzone 后所需要的内存大小 needed_size 为 24-bytes，然后会将 24-bytes RoundUp 到最近的 size class 2 对应的大小 32-bytes，最终 PrimaryAllocator 在真正申请内存时申请的大小是 32-bytes。

SizeClassMap 用于将 allocation sizes 映射到对应的 size class，以及根据 size class 得到其对应的 allocation size。SizeClassMap 在代码实现时被实现为一个模版类，包含以下模版参数：

- kMinSizeLog: defines the class 1 as 2^kMinSizeLog.

- kMaxSizeLog: defines the last class as 2^kMaxSizeLog.

- kMidSizeLog: the classes starting from 1 increase with step 2^kMinSizeLog until 2^kMidSizeLog.

- kNumBits: the number of non-zero bits in sizes after 2^kMidSizeLog.
  E.g. with kNumBits==3 all size classes after 2^kMidSizeLog look like 0b1xx0..0, where x is either 0 or 1.

ASan 使用的 SizeClassMap 模版参数为：kNumBits=3, kMinSizeLog=4, kMidSizeLog=8, kMaxSizeLog=17，具体的 size class 和 allocation size 的对应关系如下表：

| size class | allocation size (bytes) |                                                    |
| ---------- | ----------------------- | -------------------------------------------------- |
| 0          | 0                       | Class 0 always corresponds to size 0               |
| 1          | 16                      | KMinSize (2^kMinSizeLog = 2^4 = 16)                |
| 2          | 32                      | diff: +16                                          |
| 3          | 48                      |                                                    |
| 4          | 64                      |                                                    |
| 5          | 80                      |                                                    |
| 6          | 96                      |                                                    |
| 7          | 112                     |                                                    |
| 8          | 128                     |                                                    |
| 9          | 144                     |                                                    |
| 10         | 160                     |                                                    |
| 11         | 176                     |                                                    |
| 12         | 192                     |                                                    |
| 13         | 208                     |                                                    |
| 14         | 224                     |                                                    |
| 15         | 240                     |                                                    |
| 16         | 256                     | KMidSize = 2^KMidSizeLog = 2^8 = 256 = 0b100000000 |
| 17         | 320                     | diff: +64, 320 = 0b101000000                       |
| 18         | 384                     | diff: +64, 384 = 0b110000000                       |
| 19         | 448                     | diff: +64, 448 = 0b111000000                       |
| 20         | 512                     | diff: +64, 512 = 0b1000000000                      |
| 21         | 640                     | diff: +128, 640 = 0b1010000000                     |
| 22         | 768                     | diff: +128, 768 = 0b1100000000                     |
| 23         | 896                     | diff: +128, 896 = 0b1110000000                     |
| 24         | 1024                    | diff: +128, 1024 = 0b10000000000                   |
| 25         | 1280                    | diff: +256, 1280 = 0b10100000000                   |
| 26         | 1536                    | diff: +256, 1536 = 0b11000000000                   |
| 27         | 1792                    | diff: +256, 1792 = 0b11100000000                   |
| ...        |                         |                                                    |
| 52         | 131072                  | kMaxSize = 2^kMaxSizeLog = 2^17 = 131072           |

### AllocationSize To ClassId

```cpp
  static const uptr kMinSize = 1 << kMinSizeLog;
  static const uptr kMidSize = 1 << kMidSizeLog;
  static const uptr kMidClass = kMidSize / kMinSize;
  static const uptr S = kNumBits - 1;
  static const uptr M = (1 << S) - 1;

  static uptr ClassID(uptr size) {
    if (UNLIKELY(size > kMaxSize))
      return 0;
    if (size <= kMidSize)
      return (size + kMinSize - 1) >> kMinSizeLog;
    const uptr l = MostSignificantSetBitIndex(size);
    const uptr hbits = (size >> (l - S)) & M;
    const uptr lbits = size & ((1U << (l - S)) - 1);
    const uptr l1 = l - kMidSizeLog;
    return kMidClass + (l1 << S) + hbits + (lbits > 0);
  }
```

- 当 AllocationSize <= kMidSize 时，SizeClass 对应的 AllocationSize 都是以 kMinSize=2^kMinSizeLog=16 的 step 进行增长的，所以我们可以通过 `(size + kMinSize - 1) >> kMinSizeLog` 得到 AllocationSize 对应的 SizeClass 大小。
  
  例如：AllocationSize 为 160 时，对应的 SizeClass 为 `(160 + 16 - 1) >> 4` 即 10。

- 当 AllocationSize > KMidSize 时，计算 AllocationSize 对应的 SizeClass 稍显复杂。
  
  首先我们需要明确 AllocationSize > KMidSize 时，AllocationSize 的增长方式：因为 kNumBits=3，所以 kMidSize=256 之后的 AllocationSize 的二进制表示都是形如 0b1xx0..0，x 的可能取值是 0 或 1。kMidSize=256，二进制表示为 0b100000000，所以 256 后面紧跟的 4 个 AllocationSize 分别是 0b**101**000000, 0b**110**000000, 0b**111**000000, 0b**1000**000000。
  
  从 KMidSize=256 开始，每隔 2^(kNumBits-1)=4 个 SizeClass，AllocationSize 的最高有效位会左移一次。
  
  - SizeClass: 16, AllocationSize=0b100000000
  
  - SizeClass: 17, AllocationSize=0b101000000
  
  - SizeClass: 18, AllocationSize=0b110000000
  
  - SizeClass: 19, AllocationSize=0b111000000
  
  - SizeClass: 20, AllocationSize=0b1000000000
  
  所以对于一个给定的 AllocationSize=0b1xx0..0 来说，我们可以将它拆分成两部分：`AllocationSize = 0b1xx0..0 = 0b1000..0 + 0b0xx0..0`，我们将 0b**1**000..0 记作 AllocationSizeBase，将 0b0**xx0.00** 记作 AllocationSizeOffset。
  
  1. 首先计算其 AllocationSize 最高有效位的下标索引，其最高有效位的位置相比 KMidSize 的最高有效位的位置左移了 x 位，那么 AllocationSizeBase 所对应的 SizeClass 就是 kMidClass + x * 2^(kNumBits-1)
  
  2. 然后再看 AllocationSize 最高有效位后的 kNumBits - 1 位，该 kNumBits - 1 位构成的数与 (2^(kNumBits-1) - 1) 做**与运算**得到的值就是给定的 AllocationSize 相比 AllocationSizeBase 又增加了几个 SizeClass
  
  我们以 AllocationSize 为 640 为例进行说明：
  
  1. 640=0b1010000000，最高有效位是第 9 位；KMidSize = 256 = 0b100000000，其最高有效位是第 8 位，那么 0b1000000000 = 512 对应的 ClassId 就是 kMidClass + (9 - 8) * 2^(kNumBits-1) = 16 + 1 * 2^(3-1) = 20
  
  2. 640=0b1010000000，最高有效位后的 kNumBit-1=2 位就是 0b01，0b01 & (2^(kNumBits-1) - 1) = 0b01 & 3 = 1
  
  3. 所以 AllocationSize = 640 对应的 ClassId 就是 20 + 1 = 21

### ClassId To AllocationSize

```cpp
  static const uptr kMinSize = 1 << kMinSizeLog;
  static const uptr kMidSize = 1 << kMidSizeLog;
  static const uptr kMidClass = kMidSize / kMinSize;
  static const uptr S = kNumBits - 1;
  static const uptr M = (1 << S) - 1;

  static uptr Size(uptr class_id) {
    // Estimate the result for kBatchClassID because this class does not know
    // the exact size of TransferBatch. It's OK since we are using the actual
    // sizeof(TransferBatch) where it matters.
    if (UNLIKELY(class_id == kBatchClassID))
      return kMaxNumCachedHint * sizeof(uptr);
    if (class_id <= kMidClass)
      return kMinSize * class_id;
    class_id -= kMidClass;
    uptr t = kMidSize << (class_id >> S);
    return t + (t >> S) * (class_id & M);
  }
```

- 当 class_id <= kMidClass 时，class_id 对应的 AllocationSize 都是以 kMinSize=2^kMinSizeLog=16 的 step 进行增长的，所以我们可以通过 kMinSize*class_id 得到 class_id 对应的 AllocationSize。

- 当 class_id > kMidClass 时，因为从 kMidClass=16 开始，每隔 2^(kNumBits-1)=4 个 SizeClass，AllocationSize 就会增大一倍。
  
  对于一个给定的 AllocationSize=0b1xx0..0 来说，我们可以将它拆分成两部分：`AllocationSize = 0b1xx0..0 = 0b1000..0 + 0b0xx0..0`，我们将 0b**1**000..0 记作 AllocationSizeBase，将 0b0**xx0.00** 记作 AllocationSizeOffset。
  
  `t = kMidSize << (class_id >> S)` 就是计算 AllocationSizeBase 的值，`(t >> S) * (class_id & M)` 就是计算 AllocationSizeOffset 的值

## AsanThreadLocalMallocStorage

对于每个线程，ASan 都为其开辟了一块特殊的空间，称为 **AsanThreadLocalMallocStorage**。AsanThreadLocalMallocStorage 的结构如下图所示：

![](/blog/inside-asan-allocator/3c751d85af7ef5665daa62ed784feed6ac4c9ace.png)

AsanThreadLocalMallocStorage 包含两个成员变量：quarantine_cache 和 allocator_cache。

- quarantine_cache
  
  当应用程序释放 memory chunk 时，并不会直接将其释放，而是会将其（实际上是指向该 memory chunk 的指针）保存在当前线程的 quarantine_cache 中。如果当前线程的 quarantine_cache 中保存的所有 memory chunks 的总大小超过了阈值 thread_local_quarantine_size_kb（默认为 1024KB，可以通过环境变量 ASAN_OPTIONS 进行设置），则会将当前线程 quarantine_cache 中保存的 chunks 转移到全局 central quarantine 中。而当全局 central quarantine 中保存的所有 chunks 的总大小超过了阈值 quarantine_size_mb（默认为 256MB，可以通过环境变量 ASAN_OPTIONS 进行设置），才会真正释放这些 memory chunks。
  
  quarantine_cache 有两个成员变量：`atomic_uintptr_t size_` 和 `IntrusiveList<QuarantineBatch> list_`。
  
  - `atomic_uintptr_t size_`：用于记录 quarantine_cache 中保存的所有 chunks 的大小之和。
  
  - `IntrusiveList<QuarantineBatch> list_`：当应用程序释放 memory chunk 时，会将指向该 chunk 的指针保存在 list_ 的最后一个 QuarantineBatch 中。每个 QuarantineBatch 中最多保存 1021 个指向已释放的 chunk 的指针，一旦 QuarantineBatch 占满后，会新创建一个 QuarantineBatch 添加至 list_ 末尾。

- allocator_cache：
  
  对于小于等于 128KB 的内存分配申请，会优先从 allocator_cache 中选择提前从 PrimaryAllocator 分配好的保存在 allocator_cache 中的空闲 chunk 返回。由于每个线程都有一个 allocator_cache，因此从 allocator_cache 中取一个空闲 chunk 返回给应用程序是不需要加锁的，开销很小。
  
  allocator_cache 的关键成员变量是 `PerClass per_class_[kNumClasses]`，每个 PerClass 对应一个 SizeClass。以 per_class_[1] 为例，per_class_[1].size = 16 表明 per_class_[1].chunks 中保存的是提前由 PrimaryAllocator 分配好的大小为 16-bytes 的 chunks；per_class_[1].count 表示当前 per_class_[1].chunks 中空闲的 chunk 的数量；当 per_class_[1].count = 0 即 per_class_[1].chunks 中没有空闲的 memory chunks 时，会从 PrimaryAllocator 中分配 per_class_[1].max_count / 2 = 256 / 2 = 128 个大小为 16-bytes 的 memory chunks 保存至 per_class_[1].chunks 中，供后续使用。不同的 per_class_[i]，其 max_count 是不同的，例如 per_class_[1].max_count = 256, per_class_[24].max_count = 128, per_class_[52].max_count = 2。

## PrimaryAllocator(SizeClassAllocator64)

ASan allocator 的 PrimaryAllocator 就是代码中的 SizeClassAllocator64。PrimaryAllocator 用于处理 128KB 以内的内存分配。PrimaryAllocator 维护了一个 address space，结构如下图所示：

![](/blog/inside-asan-allocator/e276938890af4aa53aa0ce9a27a2c6e63804943f.png)

Address space 由两部分组成：一块大小为 kSpaceSize 的连续内存地址空间，本文中称之为 RegionSpace；一块大小为 AdditionalSize 的连续内存地址空间，本文中称之为 RegionInfoSpace。

- RegionSpace
  
  RegionSpace 的起始地址是 kSpaceBeg = 0x600000000000，结束地址是 SpaceEnd = kSpaceBeg + kSpaceSize = 0x600000000000 + 0x40000000000 = 0x640000000000。RegionSpace 被平均分为 kNumClassesRounded = 64 个 Region，每个 Region 的大小都是 0x1000000000。不同的 Region 用于不同的 SizeClass 的内存分配，例如 Region of ClassId1 用于 16-bytes memory chunk 的分配，Region of ClassId2 用于 32-bytes memory chunk 的分配。
  
  每个 Region 又是由 UserChunk, MetaChunk 和 FreeArray 组成的，而实际上 ASan 没有 MetaChunk，所以 Region 其实是由 UserChunk 和 FreeArray 组成的。UserChunk 就是后续返回给用户的 chunk，同一 Region 的 UserChunk 大小是固定的，例如 Region of ClassId1 中的 UserChunk 的大小就是 16-bytes。每个 Region 的最后 1/8 大小用作 FreeArray，FreeArray 中保存了空闲的可用于分配的 UserChunk（在代码实现上， FreeArray 中存储的是空闲的 UserChunk 的地址相对于其所在 Region 起始地址的相对偏移，相对偏移用 4-bytes 整型来存储，这样可以节省空间）。

- RegionInfoSpace
  
  RegionInfoSpace 起始地址就是 RegionSpace 的结束地址。每一个 Region 都会对应一个 RegionInfo，RegionInfo 中保存了对应的 Region 的相关信息，如 Region 中已经分配给用户的内存大小，详见结构体 RegionInfo 的定义。

## SecondaryAllocator(LargeMmapAllocator)

ASan allocator 的 SecondaryAllocator 就是代码中的 LargeMmapAllocator。LargeMmapAllocator 用于处理大于 128KB 的内存分配，顾名思义 LargeMmapAllocator 是通过 mmap 来分配内存的。

由 SecondaryAllocator 负责分配的内存会额外申请 page_size_ 大小的内存用作额外的 Header：

- map_beg 就是 SecondaryAllocator 通过 mmap 申请内存的起始地址

- map_size 即 SecondaryAllocator 实际分配的内存大小

- size 是为用户申请的内存 user memory 添加 left redzone, right redzone 后所需要的内存大小 needed_size

- chunk_idx 就是这块 SecondaryAllocator 申请的内存的索引

![](/blog/inside-asan-allocator/17f5e7268583a90ca1879c5d39d88d209ea1ad9e.png)

SecondaryAllocator 会维护一个数组 ptr_array_，该数组中保存的是指向SecondaryAllocator 已分配内存的 Header 的指针：

![](/blog/inside-asan-allocator/00d2ede99be2fc0eb375ea439dda0294a3839963.png)

给定一个指针 ptr，SecondaryAllocator 在判断该指针 ptr 是否指向一块内存由自己分配的chunk 时，就是通过遍历 ptr_array_ 查看 `ptr_array_[i]->map_beg <= ptr < ptr_array_[i]->map_beg + ptr_array_[i]->map_size` 是否满足。

## The flow of asan_allocate()

我们将 ASan allocator 分配内存的流程抽象为 `asan_allocate()`，`asan_allocate()` 的流程图如下所示：

![](/blog/inside-asan-allocator/dfc21568b22771cc9718eae247460e64a7ba4811.png)

1. 用户申请的内存大小是 user_requested_size，首先计算添加 redzone 后所需要的内存大小 needed_size。

2. 如果 needed_size 超过了 ASan 能够申请的最大内存大小 kMaxAllowedMallocSize = 0x10000000000，那么根据在环境变量 ASAN_OPTIONS 中是否设置了 may_return_null=1 而选择报错 "allocation-size-too-big" 并终止程序还是返回 nullptr。

3. 如果 needed_size <= 128KB，那么由 thread local allocator cache 负责分配，执行步骤 4。否则由 SecondaryAllocator 负责分配，执行步骤 5。

4. 对于 <= 128KB 的内存分配由 thread local allocator cache 负责。如果当前线程的 allocator cache 有 SizeClassMap::ClassId(needed_size) 的空闲 chunk，那么直接将空闲 chunk 返回给用户。如果没有的话，则先向 PrimaryAllocator 申请一些 SizeClassMap::ClassId(needed_size) 的 chunks，补充至当前线程的 allocator cache 中，然后再将空闲可用的 chunk 返回给用户。

5. 对于 > 128KB 的内存分配由 SecondaryAllocator 负责，SecondaryAllocator 通过 mmap 申请内存返回给用户。

6. 如果在步骤 4 或步骤 5 中没有足够的内存可供分配，则报错 "out-of-memory" 终止程序。

7. 成功申请到了内存后，设置 ChunkHeader：alloc_type, user_requested_alignment_log, user_requested_size, alloc_context_id。

8. 将 memory chunk 中 user memory 对应的 shadow memory 的每个字节的值都设置为表示 addressable 的 magic value，将 memory chunk 中 redzone 对应的 shadow memory 的每个字节的值设置为表示 redzone 的 magic value: kAsanHeapLeftRedzoneMagic。

9. 然后 malloc_fill 就是填充 user memory 的内容，通过 memset 将 user memory 的值设置为 0xbe，默认情况下最多只填充 user memory 的前 4KB。可以通过在环境变量 ASAN_OPTIONS 设置 max_malloc_fill_size 来控制最多填充 user memory 的多少个字节，默认 max_malloc_fill_size=0x1000；可以通过在环境变量 ASAN_OPTIONS 设置 malloc_fill_byte 即用于填充 user memory 每个字节的值，默认malloc_fill_byte=0xbe。

10. 最后将 ChunkHeader 中的 chunk_state 更新为 CHUNK_ALLOCATED，返回给用户一个指向 chunk 中 user memory 的指针。至此，ASan allocator 分配内存的流程结束。

## The flow of asan_deallocate()

我们将 ASan allocator 释放内存的流程抽象为 `asan_deallocate()`，完整的流程如下图所示：

![](/blog/inside-asan-allocator/fb0a999302bb8b253329a5b679c52430fc10f38b.png)

1. 首先查看当前正在释放的 chunk 的 ChunkHeader，如果 chunk_state 不是 CHUNK_ALLOCATED，报错 "double-free" 或 "bad-free" 并终止程序，否则继续执行步骤 2。

2. 首先更新当前正在释放的 chunk 的 ChunkHeader，将 chunk_state 更新为 CHUNK_QUARANTINE。

3. 检查 alloc_type 和 dealloc_type 是否匹配，例如申请内存时使用的 `operator new[]` 而是释放内存时却使用的 `operator delete` 这种情况。如果 alloc_type 和 dealloc_type 不匹配，报错 "alloc-dealloc-mismatch" 并终止程序。

4. 更新 ChunkHeader，设置 dealloc_context_id 即释放内存的 stacktrace 和 thread id。

5. free_fill 填充 user memory 的内容：通过 memset 将 user memory 的值设置为 0x55。默认情况下不填充 user memory，可以通过环境变量 ASAN_OPTIONS 设置 max_free_fill_size 来控制最多填充 user memory 的多少个字节，默认 max_free_fill_size=0，即不填充；可以通过环境变量 ASAN_OPTIONS 设置 free_fill_byte 即用于填充 user memory 每个字节的值，默认 free_fill_byte=0x55。

6. 将正在释放的 chunk 中 user memory 对应的 shadow memory 每个字节的值设置为 kAsanHeapFreeMagic，表示这块内存在语义上已经被释放了，实际上并没有释放而是被放入了 quarantine 中。

7. 将被释放的 chunk 添加至当前线程的 quarantine cache 中。如果当前线程的 quarantine cache 中保存的所有已释放的 chunks 的大小之和不超过 thread_local_quarantine_size_kb（默认为 1024KB，可以通过环境变量 ASAN_OPTIONS 进行设置），则 `asan_deallocate()` 的流程结束，否则继续执行步骤 8。

8. 将当前线程的 quarantine cache 转移 transfer 到全局 central quarantine 中。如果当前线程的 quarantine cache 中保存的所有已释放的 chunks 的大小之和不超过 quarantine_size_mb（默认为 256MB，可以通过环境变量 ASAN_OPTIONS 进行设置），则 `asan_deallocate()` 的流程结束，否则继续执行步骤 9。

9. 不停地从全局 central quarantine 队列中取出 chunk（记作 extracted chunks），直至全局 central quarantine 中剩余的 chunks 的大小之和不超过 quarantine_size_mb 的 90%。

10. 更新 extracted chunks 的 ChunkHeader，将 chunk_state 更新为 CHUNK_INVALID。

11. 将 extracted chunks 中 user memory 对应的 shadow memory 每个字节的值设置为 kAsanHeapLeftRedzoneMagic，表示这些 extraced chunks 已经真正被释放了，不再保存在 quarantine 中，交由 PrimaryAllocator 或 SecondaryAllocator 去释放。

12. 判断每一个 extracted chunk 在申请时是由 PrimaryAllocator 或 SecondaryAllocator 分配的。如果是由 PrimaryAllocator 分配的，那么执行步骤 12 交给 PrimaryAllocator 释放；如果是由 SecondaryAllocator 分配的，那么执行步骤 13 交给 SecondaryAllocator 释放。

13. 由 PrimaryAllocator 释放 extracted chunk，首先查看 extraced chunk 属于哪一个 size class，如果当前线程 allocator cache 中该 size class 的空闲 chunks 已经满了 (per_class_[size_class].count == per_class_[size_class].max_count)，那么将该 size class 的空闲 chunks 的一半归还给 PrimaryAllocator，保存在 PrimaryAllocator 对应的 Region 的 FreeArray 中，PrimaryAllocator 会定期将 FreeArray 中的 chunks 归还给操作系统。最后将 extracted chunk 添加至当前线程 allocator cache 中对应的 per_class_ 中。执行步骤 15。

14. 由 SecondaryAllocator 释放 extracted chunk，因为申请时是通过 mmap 分配的，所以直接通过 unmap 释放。执行步骤 15。

15. 至此，ASan allocator 释放内存的流程结束。

## Summary

本文分析了 ASan allocator 的实现，还是有很多细节没有详细讨论的，如：

- thread local allocator cache(SizeClassAllocator64LocalCache) 是怎么向 PrimaryAllocator(SizeClassAllocator64) 申请一些 chunks 补充至 thread local allocator cache 的

- thread local allocator cache(SizeClassAllocator64LocalCache) 是怎么向 PrimaryAllocator(SizeClassAllocator64) 归还 chunks 的

- PrimaryAllocator(SizeClassAllocator64) 又是怎么将内存归还操作系统的

本文没有将这些细节事无巨细地写下来，可以阅读 ASan 代码来学习这些细节。
