---
title: "Atomics Implementation in Clang/LLVM"
date: 2024-10-06
categories:
  - "Programming"
tags:
  - "C++"
  - "atomic"
  - "LLVM"
comments: true # Enable Disqus comments for specific page
toc: true
---

本文基于 llvm-project 的 release/19.x 分支版本，分析 atomics 在 Clang/LLVM 中的实现，包括 `__atomic_*` builtins、C11 `_Atomic`、C++11 `std::atomic` 等。

<!--more-->

## 0x1. Prologue

https://enna1.github.io/post/cpp-atomic-ordering-x86-tso-perspective/ 中提到：

> 在 GCC libstdc++ 的 std::atomic 实现中，是直接通过 `alignas` 来指定其对齐的，见：
>
> - [gcc/libstdc++-v3/include/std/atomic#L216](https://github.com/gcc-mirror/gcc/blob/releases/gcc-14.1.0/libstdc%2B%2B-v3/include/std/atomic#L216)
>
> - [gcc/libstdc++-v3/include/bits/atomic_base.h#L348](https://github.com/gcc-mirror/gcc/blob/releases/gcc-14.1.0/libstdc%2B%2B-v3/include/bits/atomic_base.h#L348)
>
> - [gcc/libstdc++-v3/include/bits/atomic_base.h#L1476](https://github.com/gcc-mirror/gcc/blob/releases/gcc-14.1.0/libstdc%2B%2B-v3/include/bits/atomic_base.h#L1476)

翻 commit history 发现这一行为在 [re PR libstdc++/65147 (alignment of std::atomic object is not correct)](https://gcc.gnu.org/git/?p=gcc.git;a=commitdiff;h=4cbaaa459e7f402911bb79fade9ffdac194eae75) 这一 commit 中引入的，是为了解决 [Bug 65147 - alignment of std::atomic object is not correct](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=65147) 这个 bug。

这引发了我两个疑问：

1. 在语言标准中，关于 C11 `_Atomic` 和 C++11 `std::atomic` 的 size 和 alignment 的描述是怎样的？

2. 在 libc++ 的实现中，`std::atomic` 的 size 和 alignment 是怎样的？`std::atomic` 在 libc++ 中是怎么实现的？

---

关于第一个问题，我翻阅了 C 和 C++ 的标准：

- C++ 标准 https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/n4950.pdf，关于 atomics 的 size 和 alignment 有如下描述：

  > 33.5.8 Class template atomic [atomics.types.generic]
  >
  > [Note 2 : The representation of an atomic specialization need not have the same size and alignment requirement as its corresponding argument type. — end note]

  > 33.5.12 C compatibility [stdatomic.h.syn]
  >
  > Recommended practice: Implementations should ensure that C and C++ representations of atomic objects are compatible, so that the same object can be accessed as both an _Atomic(T) from C code and an atomic from C++ code. The representations should be the same, and the mechanisms used to ensure atomicity and memory ordering should be compatible.

- C 标准 https://www.open-std.org/jtc1/sc22/wg14/www/docs/n2310.pdf，关于 atomics 的 size 和 alignment 有如下描述：

  > 6.2.5 Types
  >
  > Further, there is the _Atomic qualifier. The presence of the _Atomic qualifier designates an atomic type. The size, representation, and alignment of an atomic type need not be the same as those of the corresponding unqualified type.

  > 7.17.6 Atomic integer types
  >
  > Recommended practice: The representation of an atomic integer type is not required to have the same size as the corresponding regular type but it should have the same size whenever possible, as it eases effort required to port existing code.

所以在 C 和 C++ 标准中，有关 `_Atomic` 和 `std::atomic` 的 size 和 alignment 的描述是一样的 "need not have the same size and alignment requirement as its corresponding argument/unqualified type"。

---

关于第二个问题 “std::atomic 在 libc++ 中是怎么实现的”。随着深入代码我发现，要想完全了解 libc++ 中 `std::atomic` 的实现，光学习 libc++ 的代码是不够的，还要深入 Clang/LLVM 的代码，分析 `__atomic_*` builtins、C11 `_Atomic` 等在 Clang/LLVM 中的实现。

本文的内容组织如下：首先介绍 `__atomic_*` builtins 的实现，然后介绍 C11 `_Atomic` 的实现，最后介绍 C++11 `std::atomic` 在 libc++ 中的实现。

---

在步入正题之前再插播一句，[James Y Knight](https://github.com/jyknight) 于 2016 年总结了 atomics 在 Clang/LLVM 中的实现现状，指出“当前”的实现有些一团乱麻，提出了 cleanup 的方案，并且做了很多工作。但是似乎这项工作没能完全推进下去，现在是 2024 年，atomics 在 Clang/LLVM 中的实现现状与 2016 年相比有所改善但是并不多（所以在分析 atomics 在 Clang/LLVM 中的实现时，可以少些思考为什么这样实现，多些思考怎么改进当前的实现）。

- https://discourse.llvm.org/t/adding-sanity-to-the-atomics-implementation/39692

- https://lists.llvm.org/pipermail/llvm-dev/2016-March/096870.html

## 0x2. PR libstdc++/65147

本文缘起于 [Bug 65147 - alignment of std::atomic object is not correct](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=65147)，在步入正文之前，先试验下对于该 bug 中的 testcases 最新版本 GCC 与 Clang 的行为分别是什么。

- testcase1, https://godbolt.org/z/ojPbf33Pb

  ```C++
  #include <atomic>
  #include <stdio.h>

  typedef struct {
      char c[8];
  } power_of_two_obj;

  typedef struct {
     char c[1];
     std::atomic<power_of_two_obj> ao;
  } container_struct;

  int main ( void ) {
      std::atomic<power_of_two_obj> obj1;
      container_struct              obj2;

      printf("\n Size and Alignment of std::atomic object ");
      printf(" : sizeof(obj1) %d alignof(obj1) %d ", sizeof(obj1), alignof(obj1) );

      printf("\n Size and Alignment of std::atomic member object ");
      printf(" : sizeof(obj2.ao) %d alignof(obj2.ao) %d \n", sizeof(obj2.ao), alignof(obj2.ao) );

      return 0;
  }
  ```

  GCC(with libstdc++) 14.2 和 Clang(with libc++) 19.1.0 的行为是一样的：

  ```Plain
   Size and Alignment of std::atomic object  : sizeof(obj1) 8 alignof(obj1) 8
   Size and Alignment of std::atomic member object  : sizeof(obj2.ao) 8 alignof(obj2.ao) 8
  ```

- testcase2, https://godbolt.org/z/qE6je6a1o

  ```C++
  #include <atomic>
  #include <stdio.h>

  typedef struct {
     char c[16];
  } S16;

  int main ( void ) {
     std::atomic<char>      ac;
     std::atomic<short>     as;
     std::atomic<long>      al;
     std::atomic<long long> all;
     std::atomic<S16>       a16;

     printf("\n sizeof(ac) %d alignof(ac) %d",  sizeof(ac), alignof(ac) );
     printf("\n sizeof(as) %d alignof(as) %d",  sizeof(as), alignof(as) );
     printf("\n sizeof(al) %d alignof(al) %d",  sizeof(al), alignof(al) );
     printf("\n sizeof(all) %d alignof(all) %d",  sizeof(all), alignof(all) );
     printf("\n sizeof(a16) %d alignof(a16) %d",  sizeof(a16), alignof(a16) );
     printf("\n");
  }
  ```

  GCC(with libstdc++) 14.2 和 Clang(with libc++) 19.1.0 的行为是一样的：

  ```Plain
   sizeof(ac) 1 alignof(ac) 1
   sizeof(as) 2 alignof(as) 2
   sizeof(al) 8 alignof(al) 8
   sizeof(all) 8 alignof(all) 8
   sizeof(a16) 16 alignof(a16) 16
  ```

- testcase3, https://godbolt.org/z/8KT5Tvrbx

  注意：testcase3 与 testcase1, testcase2 不一样，testcase3 是 C 代码，不是 C++ 代码。

  ```C
  #include <stdatomic.h>
  #include <stdio.h>

  typedef struct {
     char c[16];
  } S16;

  int main ( void ) {
     _Atomic char      ac;
     _Atomic short     as;
     _Atomic long      al;
     _Atomic long long all;
     _Atomic S16       a16;

     printf("\n sizeof(ac) %d alignof(ac) %d",  sizeof(ac), __alignof__(ac) );
     printf("\n sizeof(as) %d alignof(as) %d",  sizeof(as), __alignof__(as) );
     printf("\n sizeof(al) %d alignof(al) %d",  sizeof(al), __alignof__(al) );
     printf("\n sizeof(all) %d alignof(all) %d",  sizeof(all), __alignof__(all) );
     printf("\n sizeof(a16) %d alignof(a16) %d",  sizeof(a16), __alignof__(a16) );

     printf("\n");
  }
  ```

  ```Plain
   sizeof(ac) 1 alignof(ac) 1
   sizeof(as) 2 alignof(as) 2
   sizeof(al) 8 alignof(al) 8
   sizeof(all) 8 alignof(all) 8
   sizeof(a16) 16 alignof(a16) 16
  ```

对于上述 3 个 testcases，GCC 14.2 和 Clang 19.1.0 的行为一样的。并且不管是 GCC 还是 Clang，它们对于 `_Atomic T` 和 `std::atomic<T>` 的实现都是 layout-compatible 的，满足 C++ 标准中的 recommended practice: "Implementations should ensure that C and C++ representations of atomic objects are compatible"。

那么考虑这样一个问题：GCC 和 Clang 是否所有的 atomics implementation 都是 compatible ？

## 0x3. \_\_atomic_* builtins, \_\_atomic_* libcalls

在介绍 `__atomic_*` builtins 前，需要先了解 builtins 和 libcalls 指什么：

- builtins 即 builtin functions，中文常翻译为内置函数/内建函数，指的是由编译器内部实现的特殊函数，常见 builtins 有 `__builtin_return_address`, `__builtin_expect` 等。

  通用的 builtins 定义在 [clang/include/clang/Basic/Builtins.td](https://github.com/llvm/llvm-project/blob/release/19.x/clang/include/clang/Basic/Builtins.td)，target 相关的 builtins 定义在额外的文件中，如 x86-64 相关的 builtins 定义在 [clang/include/clang/Basic/BuiltinsX86_64.def](https://github.com/llvm/llvm-project/blob/release/19.x/clang/include/clang/Basic/BuiltinsX86_64.def)。

- libcalls 即 library calls，指的对定义在 runtime library 中的函数的调用。

  [llvm/include/llvm/IR/RuntimeLibcalls.def](https://github.com/llvm/llvm-project/blob/release/19.x/llvm/include/llvm/IR/RuntimeLibcalls.def) 中描述了所有 LLVM 后端可能生成的 libcalls。

注意：有很多 `__atomic_*` builtins 和 `__atomic_*` libcalls 的命名是一样的，例如既有名为 `__atomic_load` 的 builtin，也有名为 `__atomic_load` 的 libcall，但是 `__atomic_*` builtins 并不一定被编译器 lower 为 `__atomic_*` libcalls。

### \_\_atomic_* builtins

GCC 支持的 atomic 相关的 builtins 可以在 GCC 的 online docs [6.58 Legacy __sync Built-in Functions for Atomic Memory Access](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fsync-Builtins.html) 和 [6.59 Built-in Functions for Memory Model Aware Atomic Operations](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html) 中找到。这里只列出 load, store, exchange 和 compare_exchange 相关的 `__atomic_*` builtins：

```C
// GCC load, store, exchange, and compare_exchange __atomic_* builtins

// This built-in function implements an atomic load operation. It returns the contents of *ptr.
type __atomic_load_n (type *ptr, int memorder)
// This is the generic version of an atomic load. It returns the contents of *ptr in *ret.
void __atomic_load (type *ptr, type *ret, int memorder)

// This built-in function implements an atomic store operation. It writes val into *ptr.
void __atomic_store_n (type *ptr, type val, int memorder)
// This is the generic version of an atomic store. It stores the value of *val into *ptr.
void __atomic_store (type *ptr, type *val, int memorder)

// This built-in function implements an atomic exchange operation. It writes val into *ptr, and returns the previous contents of *ptr.
type __atomic_exchange_n (type *ptr, type val, int memorder)
// This is the generic version of an atomic exchange. It stores the contents of *val into *ptr. The original value of *ptr is copied into *ret.
void __atomic_exchange (type *ptr, type *val, type *ret, int memorder)

// This built-in function implements an atomic compare and exchange operation.
bool __atomic_compare_exchange_n (type *ptr, type *expected, type desired, bool weak, int success_memorder, int failure_memorder)
// This built-in function implements the generic version of __atomic_compare_exchange.
bool __atomic_compare_exchange (type *ptr, type *expected, type *desired, bool weak, int success_memorder, int failure_memorder)
```

Clang 支持的 atomic 相关 builtins 可以在 [clang/include/clang/Basic/Builtins.td](https://github.com/llvm/llvm-project/blob/release/19.x/clang/include/clang/Basic/Builtins.td) 中找到。Clang 除了支持 GCC 支持的 `__atomic_*` builtins，还实现了很多 `__c11_atomic_*` builtins，这些 `__c11_atomic_*` builtins 由 [Richard Smith](https://github.com/zygoloid) 在 [b1e36c662bcba0ece7893c575dd7b17b2c5ac985](https://github.com/llvm/llvm-project/commit/b1e36c662bcba0ece7893c575dd7b17b2c5ac985) 这一 commit 中引入，是为了实现 C11 的 <stdatomic.h>。

注意：

1. `__c11_atomic_*` builtins 的第一个参数 address argument `ptr` 必须是指向 `_Atomic` 类型的指针，否则会报错 "address argument to atomic operation must be a pointer to _Atomic"。

2. builtins `__atomic_{load, store, exchange, compare_exchange}_n` 的 address argument `ptr` 必须是指向整型或指针类型的指针。
   builtins `__atomic_{load, store, exchange, compare_exchange}` 的 address argument `ptr` 则可以是指向任意类型的指针。

3. 对于 builtins `__atomic_{load, store, exchange, compare_exchange}` 和 `__atomic_{load, store, exchange, compare_exchange}_n`，如果使用 GCC 编译器，使用者必须保证参数 `ptr` 指向的内存是自然对齐的(naturally-aligned)，而如果使用 Clang 编译器则不需要（本文后面的内容会介绍 Clang 中 `__atomic_*` builtins 的实现）。

   相关讨论见 https://gcc.gnu.org/bugzilla/show_bug.cgi?id=96159，测试代码见 https://godbolt.org/z/ETo3hjhTW。

### \_\_atomic_* libcalls

在 [llvm/include/llvm/IR/RuntimeLibcalls.def](https://github.com/llvm/llvm-project/blob/release/19.x/llvm/include/llvm/IR/RuntimeLibcalls.def) 中可以找到 `__atomic_*` libcalls，这些 `__atomic_*` libcalls 是实现在 libatomic 中的，最常见的 libatomic 实现就是 GCC libatomic。关于 GCC libatomic 提供了哪些接口，见 [gcc/libatomic /libatomic.map](https://github.com/gcc-mirror/gcc/blob/releases/gcc-14/libatomic/libatomic.map)。

根据 https://gcc.gnu.org/wiki/Atomic/GCCMM/LIbrary（该页面自 2011-11-02 后就没更新过了，注意时效性），GCC libatomic 中 load, store, exchange 和 compare_exchange 相关的 `__atomic_*` libcalls 如下：

```C
void __atomic_load (size_t size, void *mem, void *return, int model)
I1  __atomic_load_1  (I1 *mem, int model)
I2  __atomic_load_2  (I2 *mem, int model)
I4  __atomic_load_4  (I4 *mem, int model)
I8  __atomic_load_8  (I8 *mem, int model)
I16 __atomic_load_16 (I16 *mem, int model)

void __atomic_store (size_t size, void *mem, void *val, int model)
void __atomic_store_1  (I1 *mem, I1 val, int model)
void __atomic_store_2  (I2 *mem, I2 val, int model)
void __atomic_store_4  (I4 *mem, I4 val, int model)
void __atomic_store_8  (I8 *mem, I8 val, int model)
void __atomic_store_16 (I16 *mem, I16 val, int model)

void __atomic_exchange (size_t size, void *mem, void *val, void *return, int model)
I1  __atomic_exchange_1  (I1 *mem, I1 val, int model)
I2  __atomic_exchange_2  (I2 *mem, I2 val, int model)
I4  __atomic_exchange_4  (I4 *mem, I4 val, int model)
I8  __atomic_exchange_8  (I8 *mem, I8 val, int model)
I16 __atomic_exchange_16 (I16 *mem, I16 val, int model)

bool __atomic_compare_exchange (size_t size, void *obj, void *expected, void *desired, int success, int failure)
bool  __atomic_compare_exchange_1  (I1 *mem, I1 *expected, I1 desired, int success, int failure)
bool  __atomic_compare_exchange_2  (I2 *mem, I2 *expected, I2 desired, int success, int failure)
bool  __atomic_compare_exchange_4  (I4 *mem, I4 *expected, I4 desired, int success, int failure)
bool  __atomic_compare_exchange_8  (I8 *mem, I8 *expected, I8 desired, int success, int failure)
bool  __atomic_compare_exchange_16 (I16 *mem, I16 *expected, I16 desired, int success, int failure)
```

这里简单介绍下 GCC libatomic 中 __atomic_load* 的实现，但几句话难以体现 GCC libatomic 实现之精妙，所以强烈建议阅读 GCC libatomic 的代码。

- `__atomic_load_{1, 2, 4, 8, 16}` 的代码实现在 [libatomic/load_n.c](https://github.com/gcc-mirror/gcc/blob/releases/gcc-14/libatomic/load_n.c)

  1. 如果有可用 `__atomic_load_n` builtin，那么就用该 builtin 实现 `__atomic_load_{1, 2, 4, 6, 16}`。

  2. 否则，尝试使用 compare-and-swap 实现。

  3. 如果上述方式都不可用，那么 fallback 至 barrier + lock 的方式实现（这里 barrier + lock 的实现也很有意思，建议阅读代码）。

- `__atomic_load` 的代码实现在 [libatomic/gload.c#L68](https://github.com/gcc-mirror/gcc/blob/releases/gcc-14/libatomic/gload.c#L68)

  1. 如果 `size` 是 1, 2, 4, 8, 16 且指针 `mem` 指向的地址 naturally-aligned 于 `size`，那么直接调用相应的 `__atomic_load_{1, 2, 4, 8, 16}` 来实现。

  2. 对于

     - `size` 是 1, 2, 4, 8, 16 但指针 `mem` 指向的地址不是 naturally-aligned 于 `size`，或者

     - `size` 不是 1, 2, 4, 8, 16 但小于 16

     这两种情况，依次尝试 `[mem, mem+size)` 是否位于某个 naturally-aligned 的大小为 `[1<<exp for exp in range(ceil(log2(size)), 5)]` bytes 的内存区域中。如果是，那么通过 `__atomic_load_{4, 8, 16}` 将这块 naturally-aligned 的内存 load 至临时 buffer 中，再将 buffer 中 `[mem, mem+size)` 对应的那部分内容 memcpy 至 `return` 指向的内存。

    以 `__atomic_load(3, (void *)0x993, return, model)` 为例进行说明。首先尝试 [0x993, 0x993+3) 是否位于 naturally-aligned 的 4-bytes 区域 [0x990, 0x994) 中，答案是否定的。然后尝试 [0x993, 0x993+3) 是否位于 naturally-aligned 的 8-bytes 区域 [0x990, 0x998) 中，答案是肯定的。所以先通过 `buffer = __atomic_load_8((void *)0x990, model)` 将整个 8-bytes 内存内容 load 至 buffer 中，再通过 `memcpy (return, buffer + 3, 3)` 将 [0x993, 0x993+3) 的内容保存至 `return` 指向的内存。

- 如果上述情况都不满足，那么通过 barrier + lock + memcpy 的方式实现。

  由此可见，`__atomic_load` 支持任意 `size` 并且 `__atomic_load`对参数 `mem` 指向的内存的对齐没有任何要求。

**注意**：

- `__atomic_{load, store, exchange, compare_exchange}` 这 4 个 generic libcalls 支持任意 `size` 并且对参数 `mem` 指向的内存的对齐没有任何要求。

- `__atomic_{load, store, exchange, compare_exchange}_{1, 2, 4, 8, 16}` 这些 libcalls，使用时必须保证参数 `mem` 指向的内存 naturally-aligned 于 {1, 2, 4, 8, 16}。

## 0x4. \_\_atomic_{always, is}_lock_free

### Introduction

builtin `__atomic_{always, is}_lock_free` 用于判断对于 target，特定大小（以字节为单位）的对象是否总是生成 lock-free 的原子指令。区别在于：

- builtin `__atomic_always_lock_free` 的参数 `size` **必须**是编译时常量，其返回值也是编译时常量。即 builtin `__atomic_always_lock_free` 仅能根据编译时信息做判断，当返回 false 时也只能说明仅根据编译时信息无法确定是否总是生成 lock-free 的原子指令。

- builtin `__atomic_is_lock_free` 的参数 `size` **不一定**是编译时常量，其返回值也**不一定**是编译时常量。builtin `__atomic_is_lock_free` 可以看作是 `__atomic_always_lock_free` 的加强版，如果能够在编译时确定生成 lock-free 的原子指令，那么返回值就是编译时常量 true，否则 builtin `__atomic_is_lock_free` 会编译为对 `__atomic_is_lock_free` 的 libcall（GCC libatomic 中 `__atomic_is_lock_free` 的实现见 [libatomic/glfree.c#L63](https://github.com/gcc-mirror/gcc/blob/releases/gcc-14/libatomic/glfree.c#L63)）。

builtin `__atomic_always_lock_free` 和 builtin `__atomic_is_lock_free` 的描述也在 GCC 的 online doc [6.59 Built-in Functions for Memory Model Aware Atomic Operations](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html) 中，摘抄如下：

- builtin `bool __atomic_always_lock_free (size_t size, void *ptr)`

  > This built-in function returns `true` if objects of `size` bytes always generate lock-free atomic instructions for the target architecture. `size` must resolve to a compile-time constant and the result also resolves to a compile-time constant.
  >
  > `ptr` is an optional pointer to the object that may be used to determine alignment. A value of 0 indicates typical alignment should be used. The compiler may also ignore this parameter.

  常见用法 `if (__atomic_always_lock_free (sizeof (long long), 0))`

- builtin `bool __atomic_is_lock_free (size_t size, void *ptr)`

  > This built-in function returns `true` if objects of `size` bytes always generate lock-free atomic instructions for the target architecture. If the built-in function is not known to be lock-free, a call is made to a runtime routine named `__atomic_is_lock_free`.
  >
  > `ptr` is an optional pointer to the object that may be used to determine alignment. A value of 0 indicates typical alignment should be used. The compiler may also ignore this parameter.

Clang 还提供了一个与 builtin `__atomic_is_lock_free` 类似的 builtin `__c11_atomic_is_lock_free`。正如前面所说，Clang 引入 `__c11_atomic_*` builtins 是为了实现 C11 的 <stdatomic.h>，builtin `__c11_atomic_is_lock_free` 的函数原型是 `bool __c11_atomic_is_lock_free (size_t size)`，只接收一个参数 `size`，是 `_Atomic(...)` object 的大小。

### Implementation

本节介绍在 Clang 中 builtin `__atomic_{always, is}_lock_free` 是如何实现的。

在介绍 `__atomic_{always, is}_lock_free` 的实现之前，需要先理解 `clang::TargetInfo` 的两个成员变量 **MaxAtomicPromoteWidth** 和 **MaxAtomicInlineWidth** 的含义。

[Eli Friedman](https://github.com/efriedma-quic) 在 commit [Misc fixes for atomics. Biggest fix is doing alignment correctly for _Atomic types](https://github.com/llvm/llvm-project/commit/4b72fddd9914d76370704c79f5fdedfbc6c6dedc) 中为类 `clang::TargetInfo` 引入了两个成员变量 MaxAtomicPromoteWidth 和 MaxAtomicInlineWidth。关于这两个成员变量的注释是这样写的：

> - MaxAtomicPromoteWidth: the maximum width lock-free atomic operation which will ever be supported for the given target.
>
> - MaxAtomicInlineWidth: the maximum width lock-free atomic operation which can be inlined given the supported features of the given target.

搜索 clang 代码，target 为 x86 时 MaxAtomicPromoteWidth 和 MaxAtomicInlineWidth 的取值如下：

- x86-32: MaxAtomicPromoteWidth = 64, MaxAtomicInlineWidth = 32.
  如果支持 CMPXCHG8B 指令，那么 MaxAtomicInlineWidth = 64。

- x86-64: MaxAtomicPromoteWidth = 128, MaxAtomicInlineWidth = 64.
  如果支持 CMPXCHG16B 指令，那么 MaxAtomicInlineWidth = 128。

基于上述信息还是不好理解这两个变量的作用，结合 [D59566](https://reviews.llvm.org/D59566), [D38046](https://reviews.llvm.org/D38046) 和 [D122377](https://reviews.llvm.org/D122377) 中的讨论能够帮助理解。如果仔细阅读这些讨论的内容你会发现，这些 patch 的作者在提 patch 时也不理解 MaxAtomicPromoteWidth 和 MaxAtomicInlineWidth 和作用与区别，都需要 Eli Friedman 来指正。

- MaxAtomicInlineWidth

  要理解这个变量，就得先明白这里的 "inline" 指的是什么，与 "inline" 对立的就是 "outline"，在这里就是 "libcall"。对于 <= MaxAtomicInlineWidth 的 atomic 操作，可以直接用 target 支持的指令来实现；对于 > MaxAtomicInlineWidth 的 atomic 操作，如果 target 没有直接对应的指令来支持，那么就需要依赖 libatomic，所以编译器生成的是对 libatomic runtime library 中函数的调用即 libcall。

  所以 MaxAtomicInlineWidth 是 target CPU 相关的。对于 x86-32，根据 https://www.felixcloutier.com/x86/cmpxchg8b:cmpxchg16b，Pentium(i.e. i586) processors 之前的 CPU 是不支持 CMPXCHG8B 指令的，所以只有对 Pentium 及之后的 CPU，MaxAtomicInlineWidth 才会被设置为 64。

- MaxAtomicPromoteWidth

  MaxAtomicPromoteWidth 与 MaxAtomicInlineWidth 不同，对于 x86-32 不管 target CPU 是否支持 CMPXCHG8B 指令，MaxAtomicPromoteWidth 都是 64，对于 x86-64 不管 target CPU 是否支持 CMPXCHG16B 指令，MaxAtomicPromoteWidth 都是 128。

  根据 Eli Friedman 在 [D38046](https://reviews.llvm.org/D38046) 和 [D122377](https://reviews.llvm.org/D122377) 的评论：

  > MaxAtomicPromoteWidth affects the ABI, so it can't vary based on the target CPU.
  >
  > MaxAtomicPromoteWidth should not depend on whether quadword-atomics is present, only the target OS. It determines the layout of _Atomic(__int128_t).
  >
  > (MaxAtomicInlineWidth is allowed to adjust as necessary.)

  再结合代码中使用 MaxAtomicPromoteWidth 的地方 [clang/lib/AST/ASTContext.cpp#L2395-L2418](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/AST/ASTContext.cpp#L2395-L2418)，就容易理解 MaxAtomicPromoteWidth 的含义了。这段代码决定 `_Atomic T` 的 width 和 alignment，如果 `T` 的 width 小于等于 MaxAtomicPromoteWidth，那么就将 `_Atomic T` 的 width 向上取整至 2 的幂，并且将 alignment 设置为与 width 相等。考虑 `typedef struct { char c[16]; } S16`，width 为 128-bits，alignment 为 8-bits。如果 MaxAtomicPromoteWidth 与 MaxAtomicInlineWidth 一样也是 target 相关，那么会出现：在支持 CMPXCHG16B 的 target 上，`_Atomic S16` 的 width 为 128-bits，alignment 为 128-bits；而在不支持 CMPXCHG16B 的 target 上，`_Atomic S16` 的 width 为 128-bits，alignment 为 8-bits。这会导致什么问题呢？考虑如下场景：某个基础库为了支持不同的 target，在编译时通过编译选项指定不支持 CMPXCHG16B 指令，而依赖了该基础库的代码只会运行在支持 CMPXCHG16B 的 target 上，于是在编译选项中指定支持 CMPXCHG16B 指令，两者对于 `_Atomic S16` 的 alignment 不一样，代码运行时就很有可能出现非预期行为！

理解 MaxAtomicPromoteWidth 和 MaxAtomicInlineWidth 的含义后，回到 `__atomic_{always, is}_lock_free` 的实现，代码位于 [clang/lib/AST/ExprConstant.cpp#L12936-L12995](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/AST/ExprConstant.cpp#L12936-L12995) 和 [clang/lib/CodeGen/CGBuiltin.cpp#L4755-L4776](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/CodeGen/CGBuiltin.cpp#L4755-L4776)。

如果参数 `size` 不是编译时常量，builtin `__atomic_always_lock_free` 会直接编译报错，builtin `__atomic_is_lock_free` 则会被编译为对 `__atomic_is_lock_free` 的 libcall 调用。

如果参数 `size` 是编译时常量，有以下几种情况：

- 对于 `size` 是 2 的幂且 `size <= MaxAtomicInlineWidth` 的情况。如果 `ptr` 是编译时常量且 `ptr` 对齐于 `size`，那么 `__atomic_{always, is}_lock_free` 返回编译时常量 true。如果 `ptr` 不是编译时常量，但是 `ptr` 指向的对象类型 PointeeType 已知，并且 PointeeType 的 alignment >= `size`，那么 `__atomic_{always, is}_lock_free` 返回编译时常量 true。

- `size` 是编译时常量 1 是一个 short path 的情况，不需要考虑 `ptr`，builtin `__atomic_{always, is}_lock_free` 总是返回编译时常量 true。

- 其他情况，对于 builtin `__atomic_always_lock_free` 会返回编译时常量 false，对于 builtin `__atomic_is_lock_free` 会生成对 `__atomic_is_lock_free` 的 libcall 调用。

P.S. James Y Knight 在 [Handle constant "pointers" for __atomic_always_lock_free/__atomic_is_lock_free](https://github.com/llvm/llvm-project/pull/99340) 中让 Clang 支持这样的用法 `__atomic_is_lock_free(sizeof(T), (void*)4)`。

## 0x5. Life of \_\_atomic_* builtin in Clang/LLVM

本节以 builtin `__c11_atomic_store` 为例，介绍 Clang 是如何编译 `__atomic_*` builtins 的。

### AST

`__atomic_*` builtins 在 AST 上以 AtomicExpr 表示。函数 `foo()` 及其对应的 AST 如下：

```C
void foo(_Atomic(int) *i) {
  __c11_atomic_store(i,  9981, __ATOMIC_SEQ_CST);
}
```

```Plain
|-FunctionDecl <line:1:1, line:3:1> line:1:6 foo 'void (_Atomic(int) *)'
| |-ParmVarDecl <col:10, col:24> col:24 used i '_Atomic(int) *'
| `-CompoundStmt <col:27, line:3:1>
|   `-AtomicExpr <line:2:3, col:47> 'void'
|     |-ImplicitCastExpr <col:22> '_Atomic(int) *' <LValueToRValue>
|     | `-DeclRefExpr <col:22> '_Atomic(int) *' lvalue ParmVar 0xc0ca848 'i' '_Atomic(int) *'
|     |-IntegerLiteral <<built-in>:18:26> 'int' 5
|     `-IntegerLiteral <line:2:26> 'int' 9981
```

AtomicExpr 的第一个子节点对应 `__c11_atomic_store` 的第一个参数 `_Atomic(int) *i`；第二个子节点对应 `__c11_atomic_store` 的第三个参数 `__ATOMIC_SEQ_CST`，`__ATOMIC_SEQ_CST` 是一个预定义宏，定义在 [clang/lib/Frontend/InitPreprocessor.cpp](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/Frontend/InitPreprocessor.cpp#L893)，展开后是 int 5；第三个子节点对应 `__c11_atomic_store` 的第二个参数 int 9981。

P.S. 上述 AST 中看不出来该 AtomicExpr 是 `__c11_atomic_store`，有点不方便。于是我提了一个 patch [[ASTDump] TextNodeDumper learned to dump builtin name for AtomicExpr](https://github.com/llvm/llvm-project/pull/102748)，在 AST 上显示出 AtomicExpr 表示的 builtin name。

函数 `Sema::BuildAtomicExpr()` 用于构建 AtomicExpr，代码在 [clang/lib/Sema/SemaChecking.cpp#L3530](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/Sema/SemaChecking.cpp#L3530)。前文提到，`__c11_atomic_*` builtins 的第一个参数 address argument `ptr` 必须是指向 `_Atomic` 类型的指针，否则会报错 "address argument to atomic operation must be a pointer to _Atomic"，相关代码实现就位于 [clang/lib/Sema/SemaChecking.cpp#L3767-L3771](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/Sema/SemaChecking.cpp#L3767-L3771)。

### LLVM IR

函数 `foo` 对应的 LLVM IR 如下：

```C
void foo(_Atomic(int) *i) {
  __c11_atomic_store(i,  9981, __ATOMIC_SEQ_CST);
}
```

```Plain
define dso_local void @foo(int _Atomic*)(ptr noundef %i) {
entry:
  %i.addr = alloca ptr, align 8
  %.atomictmp = alloca i32, align 4
  store ptr %i, ptr %i.addr, align 8
  %0 = load ptr, ptr %i.addr, align 8
  store i32 993, ptr %.atomictmp, align 4
  %1 = load i32, ptr %.atomictmp, align 4
  store atomic i32 %1, ptr %0 seq_cst, align 4
  ret void
}
```

为 AtomicExpr 类型的 AST 节点生成对应的 LLVM IR 的相关代码在 [clang/lib/CodeGen/CGAtomic.cpp#L817](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/CodeGen/CGAtomic.cpp#L817)，由函数 `CodeGenFunction::EmitAtomicExpr()` 负责。

- 通常情况下，AtomicExpr 类型的 AST 节点会生成对应的 LLVM IR atomic instructions([cmpxchg](https://llvm.org/docs/LangRef.html#i-cmpxchg), [atomicrmw](https://llvm.org/docs/LangRef.html#i-atomicrmw), [fence](https://llvm.org/docs/LangRef.html#i-fence), [atomic load](https://llvm.org/docs/LangRef.html#i-load) 和 [atomic store](https://llvm.org/docs/LangRef.html#i-store))。

  如果 AtomicExpr 的 address argument `ptr` 指向的类型（记作 pointeeType）的大小不是 2 的幂或者大小大于 16-bytes，那么就不再生成 LLVM IR atomic instructions，而是生成 `__atomic_*` libcalls（这一行为由 [[clang][CodeGen] Emit atomic IR in place of optimized libcalls](https://github.com/llvm/llvm-project/pull/73176) 引入，在该改动之前，当 pointeeType 的大小大于 MaxAtomicInlineWidth 或者 AtomicExpr 的 address argument `ptr` 的 alignment 不是 pointeeType 的大小的倍数时，才会生成 `__atomic_*` libcalls）。

  下例中 `sizeof(TestLarge)==32`，`alignof(TestLarge)==8`，函数 `bar()` 对应的 LLVM IR 中就是对 `__atomic_store` 的 libcall，而不是 LLVM IR atomic instructions：

  ```C++
  struct TestLarge {
    uint8_t a;
    uint64_t b, c, d;
  };

  static_assert(sizeof(TestLarge)==32, "");
  static_assert(alignof(TestLarge)==8, "");

  void bar(_Atomic(TestLarge) *ptr) {
    auto val = TestLarge{9, 9, 8, 1};
    __c11_atomic_store(ptr, val, __ATOMIC_SEQ_CST);
  }
  ```

  ```Plain
  %struct.TestLarge = type { i8, i64, i64, i64 }
  @__const.bar(TestLarge _Atomic*).val = private unnamed_addr constant %struct.TestLarge { i8 9, i64 9, i64 8, i64 1 }, align 8

  define dso_local void @bar(TestLarge _Atomic*)(ptr noundef %ptr) {
  entry:
    %ptr.addr = alloca ptr, align 8
    %val = alloca %struct.TestLarge, align 8
    %.atomictmp = alloca %struct.TestLarge, align 8
    store ptr %ptr, ptr %ptr.addr, align 8
    call void @llvm.memcpy.p0.p0.i64(ptr align 8 %val, ptr align 8 @__const.bar(TestLarge _Atomic*).val, i64 32, i1 false)
    %0 = load ptr, ptr %ptr.addr, align 8
    call void @llvm.memcpy.p0.p0.i64(ptr align 8 %.atomictmp, ptr align 8 %val, i64 32, i1 false)
    call void @__atomic_store(i64 noundef 32, ptr noundef %0, ptr noundef %.atomictmp, i32 noundef 5)
    ret void
  }
  ```

- 如果 AtomicExpr 的 address argument `ptr` 的 alignment 不是其 pointeeType 的大小的倍数，那么在编译时会给出一个 warn_atomic_op_misaligned 的 warning。

  还是以上述函数 `bar()` 为例，`sizeof(TestLarge)==32`，`alignof(TestLarge)==8`，编译时会有如下 warning：

  > warning: misaligned atomic operation may incur significant performance penalty; the expected alignment (32 bytes) exceeds the actual alignment (8 bytes) [-Watomic-alignment]

  如果我们通过 `__attribute__((aligned(x)))` 或 `alignas` 将 `struct TestLarge` 的 alignment 设置为 32-bytes，与其 size 相等，那么 `ptr` 的 alignment 也会是 32-bytes，编译时就不会再有上述 warning。

  ```C++
  struct alignas(32) TestLarge {
    uint8_t a;
    uint64_t b, c, d;
  };

  void bar(_Atomic(TestLarge) *ptr) {
    auto val = TestLarge{9, 9, 8, 1};
    __c11_atomic_store(ptr, val, __ATOMIC_SEQ_CST);
  }
  ```

- 如果 AtomicExpr 的 address argument `ptr` 指向类型 pointeeType 的大小超过 MaxAtomicInlineWidth，那么在编译时会给出一个 warn_atomic_op_oversized 类型的 warning。

  还是以上述函数 `bar()` 为例，`sizeof(TestLarge)==32`，在编译时会有如下 warning：

  > warning: large atomic operation may incur significant performance penalty; the access size (32 bytes) exceeds the max lock-free size (8 bytes) [-Watomic-alignment]

  如前所述，对于 x86-64 如果 target CPU 支持 CMPXCHG16B，那么 MaxAtomicInlineWidth = 128。所以如果在编译选项中添加 `-mcx16` 或者其他表示支持 CMPXCHG16B 的编译选项，那么这里的 warning 中的 max lock-free size 就由 8 bytes 变为 16 bytes。

### AtomicExpandPass

AtomicExpandPass 代码位于 [llvm/lib/CodeGen/AtomicExpandPass.cpp](https://github.com/llvm/llvm-project/blob/release/19.x/llvm/lib/CodeGen/AtomicExpandPass.cpp)，注释是这样写的 "a pass (at IR level) to replace atomic instructions with `__atomic_*` library calls, or target specific instruction which implement the same semantics in a way which better fits the target backend."

AtomicExpandPass 虽然内容较多，但是代码结构清晰、易于阅读，所以本节只浅尝辄止地介绍很少部分内容，建议阅读代码了解 AtomicExpandPass 的所有细节。

#### replace with \_\_atomic_* libcalls

对于 LLVM IR atomic instructions [atomic load](https://llvm.org/docs/LangRef.html#i-load), [atomic store](https://llvm.org/docs/LangRef.html#i-store), [atomicrmw](https://llvm.org/docs/LangRef.html#i-atomicrmw) 和 [cmpxchg](https://llvm.org/docs/LangRef.html#i-cmpxchg) 即 LoadInst, StoreInst, AtomicRMWInst, AtomicCmpXchgInst，当函数 `atomicSizeSupported()` 返回 false 时，AtomicExpandPass 会将这些 LLVM IR atomic instructions 替换为 `__atomic_*` libcalls。

```C++
// Determine if a particular atomic operation has a supported size,
// and is of appropriate alignment, to be passed through for target
// lowering. (Versus turning into a __atomic libcall)
template <typename Inst>
static bool atomicSizeSupported(const TargetLowering *TLI, Inst *I) {
  unsigned Size = getAtomicOpSize(I);
  Align Alignment = I->getAlign();
  return Alignment >= Size &&
         Size <= TLI->getMaxAtomicSizeInBitsSupported() / 8;
}
```

函数 `atomicSizeSupported()` 的实现很简单，不用赘述。这里详细介绍下 **MaxAtomicSizeInBitsSupported**。MaxAtomicSizeInBitsSupported 是类 `llvm::TargetLoweringBase` 的成员变量，定义位于 [llvm/include/llvm/CodeGen/TargetLowering.h](https://github.com/llvm/llvm-project/blob/release/19.x/llvm/include/llvm/CodeGen/TargetLowering.h#L3532)，由 James Y Knight 在 [D18200](https://reviews.llvm.org/D18200) 中引入。

MaxAtomicSizeInBitsSupported 的代码注释是这样写的：

> Size in bits of the maximum atomics size the backend supports. Accesses larger than this will be expanded by AtomicExpandPass。

https://llvm.org/docs/Atomics.html 中与 MaxAtomicSizeInBitsSupported 的相关描述：

> AtomicExpandPass can help with that: it will expand all atomic operations to the proper `__atomic_*` libcalls for any size above the maximum set by setMaxAtomicSizeInBitsSupported (which defaults to 0).
>
> LLVM's AtomicExpandPass will translate atomic operations on data sizes above MaxAtomicSizeInBitsSupported into calls to these functions.

MaxAtomicSizeInBitsSupported 与 target CPU 相关，只会被 AtomicExpandPass 使用，在 AtomicExpandPass 中会将 size 大于 setMaxAtomicSizeInBitsSupported 的 atomic operations 转为对应的 `__atomic_*` libcalls。

回顾下前文提到的 Clang 前端的 MaxAtomicInlineWidth，会发现 MaxAtomicSizeInBitsSupported 与 MaxAtomicInlineWidth 的含义非常类似，只不过一个在 LLVM 后端，一个在 Clang 前端。MaxAtomicSizeInBitsSupported 的取值逻辑与 MaxAtomicPromoteWidth 也是一样的。对于 x86，如果 target CPU 支持 CMPXCHG16B 指令，MaxAtomicSizeInBitsSupported 就是 128；如果 target CPU 不支持 CMPXCHG16B 指令但支持 CMPXCHG8B 指令，MaxAtomicSizeInBitsSupported 就是 64；如果 target CPU 既不支持 CMPXCHG16B 指令也不支持 CMPXCHG8B 指令，MaxAtomicSizeInBitsSupported 就是 32。相关代码见 [llvm/lib/Target/X86/X86ISelLowering.cpp#L184-L189](https://github.com/llvm/llvm-project/blob/release/19.x/llvm/lib/Target/X86/X86ISelLowering.cpp)。

举个例子，AtomicExpandPass 不会修改函数 `foo()` LLVM IR 中的 atomic StoreInst，但是对于函数 `baz()` 因 i 的 alignment=2 小于其 size=4 所以 AtomicExpandPass 会将函数 `baz()` LLVM IR 上的 atomic StoreInst 替换为 __atomic_store 的 libcall。在线示例 https://godbolt.org/z/T6W7TG9sz。

问题：为什么这里 AtomicExpandPass 会将 `baz()` 的 LLVM IR atomic StoreInst 替换为 `__atomic_store` 的 libcall 而不是 `__atomic_store_4` 的 libcall？如果不知道答案，可以回顾下 \_\_atomic_* libcalls 这一节。

```C++
void foo(_Atomic(int) *i) {
  __c11_atomic_store(i, 9981, __ATOMIC_SEQ_CST);
}
```

```Plain
define dso_local void @foo(int _Atomic*)(ptr noundef %i) {
entry:
  %i.addr = alloca ptr, align 8
  %.atomictmp = alloca i32, align 4
  store ptr %i, ptr %i.addr, align 8
  %0 = load ptr, ptr %i.addr, align 8
  store i32 9981, ptr %.atomictmp, align 4
  %1 = load i32, ptr %.atomictmp, align 4
  store atomic i32 %1, ptr %0 seq_cst, align 4
  ret void
}
```

```C++
typedef _Atomic(int) Aligned2_Atomic_int __attribute__((aligned(2)));
void baz(Aligned2_Atomic_int *i) {
  __c11_atomic_store(i, 9981, __ATOMIC_SEQ_CST);
}
```

```Plain
define dso_local void @baz(int _Atomic*)(ptr noundef %i) {
entry:
  %0 = alloca i32, align 4
  %i.addr = alloca ptr, align 8
  %.atomictmp = alloca i32, align 4
  store ptr %i, ptr %i.addr, align 8
  %1 = load ptr, ptr %i.addr, align 8
  store i32 9981, ptr %.atomictmp, align 4
  %2 = load i32, ptr %.atomictmp, align 4
  call void @llvm.lifetime.start.p0(i64 4, ptr %0)
  store i32 %2, ptr %0, align 4
  call void @__atomic_store(i64 4, ptr %1, ptr %0, i32 5)
  call void @llvm.lifetime.end.p0(i64 4, ptr %0)
  ret void
}
```

#### replace with better target specific instruction

对于不支持 SSE 和 X87 指令的 x86-32 target，如果支持 CMPXCHG8B 指令，那么使用 CMPXCHG8B 指令实现 64-bits atomic load/store。
对于不支持 AVX 指令的 x86-64 target，如果支持 CMPXCHG16B 指令，那么使用 CMPXCHG8B 指令实现 128-bits atomic load/store。
对于上述情况，AtomicExpandPass 在 LLVM IR 上将 atomic LoadInst, StoreInst 替换为 AtomicCmpXchgInst，对 AtomicCmpXchgInst 指令选择为 CMPXCHG8B/CMPXCHG16B 是 SelectionDAG Instruction Selection 负责的。

举个例子，使用 x86_64-unknown-linux-gnu Clang 编译如下代码时添加 `-mcx16` 编译选项，AtomicExpandPass 会使用 AtomicCmpXchgInst 替换 atomic LoadInst/StoreInst。在线示例 https://godbolt.org/z/frPjTvq4d。

```C++
// compile with -mcx16

void qux(_Atomic(__int128) *ptr) {
  __c11_atomic_store(ptr, 9981, __ATOMIC_RELEASE);
}

__int128 quux(_Atomic(__int128) *ptr) {
  return __c11_atomic_load(ptr, __ATOMIC_ACQUIRE);
}
```

```Diff
define dso_local void @qux(__int128 _Atomic*)(ptr noundef %ptr) {
entry:
  %ptr.addr = alloca ptr, align 8
  %.atomictmp = alloca i128, align 16
  store ptr %ptr, ptr %ptr.addr, align 8
  %0 = load ptr, ptr %ptr.addr, align 8
  store i128 9981, ptr %.atomictmp, align 16
  %1 = load i128, ptr %.atomictmp, align 16
-  store atomic i128 %1, ptr %0 release, align 16
+  %2 = load i128, ptr %0, align 16
+  br label %atomicrmw.start
+
+atomicrmw.start:                                  ; preds = %atomicrmw.start, %entry
+  %loaded = phi i128 [ %2, %entry ], [ %newloaded, %atomicrmw.start ]
+  %3 = cmpxchg ptr %0, i128 %loaded, i128 %1 release monotonic, align 16
+  %success = extractvalue { i128, i1 } %3, 1
+  %newloaded = extractvalue { i128, i1 } %3, 0
+  br i1 %success, label %atomicrmw.end, label %atomicrmw.start
+
+atomicrmw.end:                                    ; preds = %atomicrmw.start
  ret void
}
```

```Diff
define dso_local noundef { i64, i64 } @quux(__int128 _Atomic*)(ptr noundef %ptr) {
entry:
  %retval = alloca i128, align 16
  %ptr.addr = alloca ptr, align 8
  %atomic-temp = alloca i128, align 16
  store ptr %ptr, ptr %ptr.addr, align 8
  %0 = load ptr, ptr %ptr.addr, align 8
-  %1 = load atomic i128, ptr %0 acquire, align 16
-  store i128 %1, ptr %atomic-temp, align 16
+  %1 = cmpxchg ptr %0, i128 0, i128 0 acquire acquire, align 16
+  %loaded = extractvalue { i128, i1 } %1, 0
+  store i128 %loaded, ptr %atomic-temp, align 16
  %2 = load i128, ptr %atomic-temp, align 16
  store i128 %2, ptr %retval, align 16
  %3 = load { i64, i64 }, ptr %retval, align 16
  ret { i64, i64 } %3
}
```

除了本节提到的 AtomicExpandPass 在某些情况下会将 atomic LoadInst, StoreInst 替换为 AtomicCmpXchgInst，AtomicExpandPass 还有很多其他的 "expand"，这里列举一些：

- [[X86] Use movq for i64 atomic load on 32-bit targets when sse2 is enable](https://reviews.llvm.org/D59679)

- [[X86] Enable the use of movlps for i64 atomic load on 32-bit targets with sse1](https://github.com/llvm/llvm-project/commit/15b6aa744881b6e77a3d6773afa3016fc2f9f123)

- [[X86] Use plain load/store instead of cmpxchg16b for atomics with AVX](https://github.com/llvm/llvm-project/pull/74275)

最后看下 AtomicExpandPass 是如何被添加到 pass pipeline 中的。对于 x86 target，`X86PassConfig` 继承自 `TargetPassConfig`，override 了函数 `addIRPasses()`，在函数最开始调用 `addPass(createAtomicExpandLegacyPass())`，将 AtomicExpandPass 添加到 PassManager。相关代码见 [llvm/lib/Target/X86/X86TargetMachine.cpp#L464](https://github.com/llvm/llvm-project/blob/release/19.x/llvm/lib/Target/X86/X86TargetMachine.cpp#L464)。

### SelectionDAG

Eli Friedman 在 commit [Basic x86 code generation for atomic load and store instructions](https://github.com/llvm/llvm-project/commit/342e8df0e0cd5af639a53b26686550132a0ee17e) 中为 x86 实现了 atomic load/store 的 SelectionDAG Instruction Selection 基本支持。

以 `foo()` 函数为例简单说明 SelectionDAG Instruction Selection 过程。在线示例见 https://godbolt.org/z/9P468bssG。

```C++
void foo(_Atomic(int) *i) {
  __c11_atomic_store(i,  9981, __ATOMIC_SEQ_CST);
}
```

使用 Clang 编译上述代码时可以添加编译选项 `-mllvm -debug-only=isel-dump` 输出 SelectionDAG 指令选择的相关信息：

1. Initial selection DAG:

   ```Plain
   Initial selection DAG: %bb.0 '_Z3fooPU7_Atomici:entry'
   SelectionDAG has 13 nodes:
     t0: ch,glue = EntryToken
     t4: i64 = Constant<0>
         t2: i64,ch = CopyFromReg t0, Register:i64 %1
       t6: ch = store<(store (s64) into %ir.i.addr)> t0, t2, FrameIndex:i64<0>, undef:i64
     t7: i64,ch = load<(dereferenceable load (s64) from %ir.i.addr)> t6, FrameIndex:i64<0>, undef:i64, example.cpp:2:22
       t10: ch = store<(store (s32) into %ir..atomictmp)> t7:1, Constant:i32<9981>, FrameIndex:i64<1>, undef:i64, example.cpp:2:3
     t11: i32,ch = load<(dereferenceable load (s32) from %ir..atomictmp)> t10, FrameIndex:i64<1>, undef:i64, example.cpp:2:3
     t12: ch = AtomicStore<(store seq_cst (s32) into %ir.0)> t11:1, t11, t7, example.cpp:2:3
   ```

2. Optimized lowered selection DAG, Type-legalized selection DAG

   Type-legalized selection DAG 前后无变化，故省略。

   ```Plain
   Optimized lowered selection DAG: %bb.0 '_Z3fooPU7_Atomici:entry'
   SelectionDAG has 12 nodes:
     t0: ch,glue = EntryToken
         t2: i64,ch = CopyFromReg t0, Register:i64 %1
       t6: ch = store<(store (s64) into %ir.i.addr)> t0, t2, FrameIndex:i64<0>, undef:i64
     t7: i64,ch = load<(dereferenceable load (s64) from %ir.i.addr)> t6, FrameIndex:i64<0>, undef:i64, example.cpp:2:22
       t10: ch = store<(store (s32) into %ir..atomictmp)> t7:1, Constant:i32<9981>, FrameIndex:i64<1>, undef:i64, example.cpp:2:3
     t11: i32,ch = load<(dereferenceable load (s32) from %ir..atomictmp)> t10, FrameIndex:i64<1>, undef:i64, example.cpp:2:3
     t12: ch = AtomicStore<(store seq_cst (s32) into %ir.0)> t11:1, t11, t7, example.cpp:2:3

   Type-legalized selection DAG: %bb.0 '_Z3fooPU7_Atomici:entry'
   ...
   ```

3. Legalized selection DAG, Optimized legalized selection DAG

   Optimized legalized selection DAG 前后无变化，故省略。

   ```Plain
   Legalized selection DAG: %bb.0 '_Z3fooPU7_Atomici:entry'
   SelectionDAG has 12 nodes:
     t0: ch,glue = EntryToken
         t2: i64,ch = CopyFromReg t0, Register:i64 %1
       t6: ch = store<(store (s64) into %ir.i.addr)> t0, t2, FrameIndex:i64<0>, undef:i64
     t7: i64,ch = load<(dereferenceable load (s64) from %ir.i.addr)> t6, FrameIndex:i64<0>, undef:i64, example.cpp:2:22
       t10: ch = store<(store (s32) into %ir..atomictmp)> t7:1, Constant:i32<9981>, FrameIndex:i64<1>, undef:i64, example.cpp:2:3
     t11: i32,ch = load<(dereferenceable load (s32) from %ir..atomictmp)> t10, FrameIndex:i64<1>, undef:i64, example.cpp:2:3
     t13: i32,ch = AtomicSwap<(store seq_cst (s32) into %ir.0)> t11:1, t7, t11, example.cpp:2:3

   Optimized legalized selection DAG: %bb.0 '_Z3fooPU7_Atomici:entry'
   ...
   ```

4. Selected selection DAG:

   ```Plain
   Selected selection DAG: %bb.0 '_Z3fooPU7_Atomici:entry'
   SelectionDAG has 15 nodes:
     t0: ch,glue = EntryToken
         t2: i64,ch = CopyFromReg t0, Register:i64 %1
       t6: ch = MOV64mr<Mem:(store (s64) into %ir.i.addr)> TargetFrameIndex:i64<0>, TargetConstant:i8<1>, Register:i64 $noreg, TargetConstant:i32<0>, Register:i16 $noreg, t2, t0
     t7: i64,ch = MOV64rm<Mem:(dereferenceable load (s64) from %ir.i.addr)> TargetFrameIndex:i64<0>, TargetConstant:i8<1>, Register:i64 $noreg, TargetConstant:i32<0>, Register:i16 $noreg, t6, example.cpp:2:22
       t10: ch = MOV32mi<Mem:(store (s32) into %ir..atomictmp)> TargetFrameIndex:i64<1>, TargetConstant:i8<1>, Register:i64 $noreg, TargetConstant:i32<0>, Register:i16 $noreg, TargetConstant:i32<9981>, t7:1, example.cpp:2:3
     t11: i32,ch = MOV32rm<Mem:(dereferenceable load (s32) from %ir..atomictmp)> TargetFrameIndex:i64<1>, TargetConstant:i8<1>, Register:i64 $noreg, TargetConstant:i32<0>, Register:i16 $noreg, t10, example.cpp:2:3
     t13: i32,ch = XCHG32rm<Mem:(store seq_cst (s32) into %ir.0)> t11, t7, TargetConstant:i8<1>, Register:i64 $noreg, TargetConstant:i32<0>, Register:i16 $noreg, t11:1, example.cpp:2:3
   ```

SelectionDAG Instruction Selection 过程中：

- Legalized selection DAG 将 t12 AtomicStore(store seq_cst) 节点 legalize 为 t13 AtomicSwap 节点

  相关代码见 [llvm/lib/CodeGen/SelectionDAG/LegalizeDAG.cpp#L1034](https://github.com/llvm/llvm-project/blob/release/19.x/llvm/lib/CodeGen/SelectionDAG/LegalizeDAG.cpp#L1034), [llvm/lib/Target/X86/X86ISelLowering.cpp#31805](https://github.com/llvm/llvm-project/blob/release/19.x/llvm/lib/Target/X86/X86ISelLowering.cpp#31805)。

- AtomicSwap 节点指令选择为 XCHG32rm

  相关代码见 [llvm/include/llvm/Target/TargetSelectionDAG.td#L715](https://github.com/llvm/llvm-project/blob/release/19.x/llvm/include/llvm/Target/TargetSelectionDAG.td#L715), [llvm/lib/Target/X86/X86InstrMisc.td#L817-L850](https://github.com/llvm/llvm-project/blob/release/19.x/llvm/lib/Target/X86/X86InstrMisc.td#L817-L850)。

“普通”的 atomic_load 和 atomic_store 指令选择，见 [llvm/lib/Target/X86/X86InstrCompiler.td#L1177-L1189](https://github.com/llvm/llvm-project/blob/release/19.x/llvm/lib/Target/X86/X86InstrCompiler.td#L1177-L1189)

## 0x6. Implementation of C11 _Atomic

Standard header for atomic types and operations: [clang/lib/Headers/stdatomic.h](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/Headers/stdatomic.h)

`_Atomic` 关键字是 C11 引入的，用法是 `_Atomic (base-type-name)` 或 `_Atomic base-type-name`，详见 https://en.cppreference.com/w/c/language/atomic。

类型被 `_Atomic` 关键字修饰会改变什么？主要是以下两个方面

1. width(size) 和 alignment

   > the atomic version of *type-name* may have a different size, alignment, and object representation.

   对于 `_Atomic (base-type-name)`，如果 base type 的 width 小于等于 目标 target 的 MaxAtomicPromoteWidth，那么就将 width 和 alignment 设置更有利于原子操作的值，将 `_Atomic (base-type-name)` 类型的 width 向上取整至 2 的幂，并且将 alignment 的值设置为等于 width 的值。相关代码见 [clang/lib/AST/ASTContext.cpp#L2395-L2418](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/AST/ASTContext.cpp#L2395-L2418)。

2. operations

   > Built-in [increment and decrement operators](https://en.cppreference.com/w/c/language/operator_incdec) and [compound assignment](https://en.cppreference.com/w/c/language/operator_assignment) are read-modify-write atomic operations with total sequentially consistent ordering (as if using [memory_order_seq_cst](https://en.cppreference.com/w/c/atomic/memory_order)).

考虑如下例子 https://godbolt.org/z/EfT8b1vza：

```C
// clang/test/CodeGen/pr45476.cpp

struct s3 {
  char a, b, c;
};

_Atomic struct s3 a;
struct s3 b;

void foo() { a = s3{1, 2, 3}; }
void bar() { b = s3{1, 2, 3}; }
```

1. width 和 alignment：`sizeof(struct s3)=3`, `alignof(struct s3)=1`，而 `sizeof(_Atomic struct s3)=4`, `alignof(_Atomic struct s3)=4`

2. operations：函数 `foo()` 中 `a = s3{1, 2, 3}` 对应的 LLVM IR 是一条 seq_cst atomic store 指令，而函数 `foo()` 中 `a = s3{1, 2, 3}` 对应的 LLVM IR 是三条普通的非 atomic 的 store 指令。

## 0x7. Implementation of C++11 std::atomic

[Louis Dionne](https://github.com/ldionne) 在 [[libc++] Enable <atomic> when threads are disabled](https://github.com/llvm/llvm-project/commit/92832e4889ae6038cc8b3e6e449af2a9b9374ab4) commit message 中有这样一句话：

> std::atomic is, for the most part, just a thin veneer on top of compiler builtins.

看过 libc++ 的 `std::atomic` 代码实现后会发现确实如此。

`std::atomic` 是一个结构体。在 libc++ 中，结构体 `atomic` 继承自结构体 `__atomic_base`，绝大多数 `std::atomic` 的成员函数都是继承自结构体 `__atomic_base` 的实现。见 [libcxx/include/__atomic/atomic.h#L36](https://github.com/llvm/llvm-project/blob/release/19.x/libcxx/include/__atomic/atomic.h#L36), [libcxx/include/__atomic/atomic_base.h#L31](https://github.com/llvm/llvm-project/blob/release/19.x/libcxx/include/__atomic/atomic_base.h#L31)。

截取结构体 `__atomic_base` 的部分代码实现：

```C++
template <class _Tp, bool = is_integral<_Tp>::value && !is_same<_Tp, bool>::value>
struct __atomic_base // false
{
  mutable __cxx_atomic_impl<_Tp> __a_;

  _LIBCPP_HIDE_FROM_ABI void store(_Tp __d, memory_order __m = memory_order_seq_cst) _NOEXCEPT
      _LIBCPP_CHECK_STORE_MEMORY_ORDER(__m) {
    std::__cxx_atomic_store(std::addressof(__a_), __d, __m);
  }

  _LIBCPP_HIDE_FROM_ABI _Tp load(memory_order __m = memory_order_seq_cst) const _NOEXCEPT
      _LIBCPP_CHECK_LOAD_MEMORY_ORDER(__m) {
    return std::__cxx_atomic_load(std::addressof(__a_), __m);
  }

  _LIBCPP_HIDE_FROM_ABI _Tp exchange(_Tp __d, memory_order __m = memory_order_seq_cst) _NOEXCEPT {
    return std::__cxx_atomic_exchange(std::addressof(__a_), __d, __m);
  }

  _LIBCPP_HIDE_FROM_ABI bool compare_exchange_weak(_Tp& __e, _Tp __d, memory_order __s, memory_order __f) _NOEXCEPT
      _LIBCPP_CHECK_EXCHANGE_MEMORY_ORDER(__s, __f) {
    return std::__cxx_atomic_compare_exchange_weak(std::addressof(__a_), std::addressof(__e), __d, __s, __f);
  }

  _LIBCPP_HIDE_FROM_ABI bool compare_exchange_strong(_Tp& __e, _Tp __d, memory_order __s, memory_order __f) _NOEXCEPT
      _LIBCPP_CHECK_EXCHANGE_MEMORY_ORDER(__s, __f) {
    return std::__cxx_atomic_compare_exchange_strong(std::addressof(__a_), std::addressof(__e), __d, __s, __f);
  }

  // ...
}
```

像 `std::atomic::{store, load, exchange, compare_exchange_weak, compare_exchange_strong}` 这样的 `std::atomic<T>` 成员函数都是继承自结构体 `__atomic_base` 对应的成员函数的实现。结构体 `__atomic_base` 有一个 `mutable` 的 `__cxx_atomic_impl<_Tp>` 类型的成员变量 `__a_`，而 `__atomic_base::{store, load, exchange, compare_exchange_weak, compare_exchange_strong}` 的实现其实就是以 `std::addressof(__a_)` 作为第一个参数调用对应的函数 `std::__cxx_atomic_{store, load, exchange, compare_exchange_weak, compare_exchange_strong}`。

P.S. 这里为什么用 `std::addressof(__a_)` 而不是直接用 `&__a_`？

- 查看 [C++ reference](https://en.cppreference.com/w/cpp/memory/addressof) 和 [Contributing to libc++](https://libcxx.llvm.org/Contributing.html#coding-standards) 后找到了答案：避免调用重载(overload)版本的 `operator&`

- 顺便看了下 `std::addressof` 的实现 [libcxx/include/__memory/addressof.h](https://github.com/llvm/llvm-project/blob/release/19.x/libcxx/include/__memory/addressof.h)，简单来说就是 `__builtin_addressof`

结构体 `__cxx_atomic_impl` 和函数 `std::__cxx_atomic_{store, load, exchange, compare_exchange_weak, compare_exchange_strong}` 的实现都位于 [libcxx/include/__atomic/cxx_atomic_impl.h](https://github.com/llvm/llvm-project/blob/release/19.x/libcxx/include/__atomic/cxx_atomic_impl.h)。

### __cxx_atomic_impl

```C++
template <typename _Tp, typename _Base = __cxx_atomic_base_impl<_Tp> >
struct __cxx_atomic_impl : public _Base {
  static_assert(is_trivially_copyable<_Tp>::value, "std::atomic<T> requires that 'T' be a trivially copyable type");

  _LIBCPP_HIDE_FROM_ABI __cxx_atomic_impl() _NOEXCEPT = default;
  _LIBCPP_HIDE_FROM_ABI _LIBCPP_CONSTEXPR explicit __cxx_atomic_impl(_Tp __value) _NOEXCEPT : _Base(__value) {}
};
```

结构体 `__cxx_atomic_impl` 继承自结构体 `__cxx_atomic_base_impl`。

结构体 `__cxx_atomic_base_impl` 的实现分为两种情况：宏 `_LIBCPP_HAS_C_ATOMIC_IMP` 被定义、宏 `_LIBCPP_HAS_GCC_ATOMIC_IMP` 被定义。这两个宏在 [libcxx/include/__config](https://github.com/llvm/llvm-project/blob/release/19.x/libcxx/include/__config#L897) 中定义：

```C++
#  if __has_feature(cxx_atomic) || __has_extension(c_atomic) || __has_keyword(_Atomic)
#    define _LIBCPP_HAS_C_ATOMIC_IMP
#  elif defined(_LIBCPP_COMPILER_GCC)
#    define _LIBCPP_HAS_GCC_ATOMIC_IMP
#  endif
```

Clang:

- __has_feature(cxx_atomic) [clang/include/clang/Basic/Features.def#L167](https://github.com/llvm/llvm-project/blob/release/19.x/clang/include/clang/Basic/Features.def#L167)

- __has_extension(c_atomic) [clang/include/clang/Basic/Features.def#L158](https://github.com/llvm/llvm-project/blob/release/19.x/clang/include/clang/Basic/Features.def#L158)

- __has_keyword(_Atomic) [clang/include/clang/Basic/TokenKinds.def#L333](https://github.com/llvm/llvm-project/blob/release/19.x/clang/include/clang/Basic/TokenKinds.def#L333)

GCC:

- __has_extension(c_atomic) [gcc/blob/releases/gcc-14/gcc/c/c-objc-common.cc#L51](https://github.com/gcc-mirror/gcc/blob/releases/gcc-14/gcc/c/c-objc-common.cc#L51)

  GCC 编译 C++ 代码时 `__has_extension(c_atomic)` 为 false，只有编译 C 代码时 `__has_extension(c_atomic)` 为 true。

也就是说宏 `_LIBCPP_HAS_C_ATOMIC_IMP` 被定义时是 Clang + libc++ 的情况，而宏 `_LIBCPP_HAS_GCC_ATOMIC_IMP` 被定义时是 GCC + libc++ 的情况。

1. 宏 `_LIBCPP_HAS_C_ATOMIC_IMP` 被定义时，结构体 `__cxx_atomic_base_impl` 的实现如下：

   ```C++
   template <typename _Tp>
   struct __cxx_atomic_base_impl {
     _LIBCPP_HIDE_FROM_ABI
     __cxx_atomic_base_impl() _NOEXCEPT = default;
     _LIBCPP_CONSTEXPR explicit __cxx_atomic_base_impl(_Tp __value) _NOEXCEPT : __a_value(__value) {}
     _LIBCPP_DISABLE_EXTENSION_WARNING _Atomic(_Tp) __a_value;
   };
   ```

2. 宏 `_LIBCPP_HAS_GCC_ATOMIC_IMP` 被定义时，结构体 `__cxx_atomic_base_impl` 的实现如下：

   ```C++
   template <typename _Tp>
   struct __cxx_atomic_base_impl {
     _LIBCPP_HIDE_FROM_ABI
     __cxx_atomic_base_impl() _NOEXCEPT = default;
     _LIBCPP_CONSTEXPR explicit __cxx_atomic_base_impl(_Tp value) _NOEXCEPT : __a_value(value) {}
     _Tp __a_value;
   };
   ```

区别在于：`__cxx_atomic_base_impl` 的成员变量 `__a_value` 的类型，在宏 `_LIBCPP_HAS_C_ATOMIC_IMP` 被定义时为 `_Atomic(_Tp)`，而在宏 `_LIBCPP_HAS_GCC_ATOMIC_IMP` 被定义时为 `_Tp`。可以回顾下前文，使用 Clang 编译时类型被 `_Atomic` 关键字修饰会改变什么。

### \_\_cxx_atomic_{store, load, exchange, compare_exchange_weak, compare_exchange_strong}

1. 宏 `_LIBCPP_HAS_C_ATOMIC_IMP` 被定义时，函数`std::__cxx_atomic_store` 的实现如下：

   ```C++
   template <class _Tp>
   _LIBCPP_HIDE_FROM_ABI void
   __cxx_atomic_store(__cxx_atomic_base_impl<_Tp>* __a, _Tp __val, memory_order __order) _NOEXCEPT {
     __c11_atomic_store(std::addressof(__a->__a_value), __val, static_cast<__memory_order_underlying_t>(__order));
   }
   ```

2. 宏 `_LIBCPP_HAS_GCC_ATOMIC_IMP` 被定义时，结构体 `std::__cxx_atomic_store` 的实现如下：

   ```C++
   template <typename _Tp>
   _LIBCPP_HIDE_FROM_ABI void __cxx_atomic_store(__cxx_atomic_base_impl<_Tp>* __a, _Tp __val, memory_order __order) {
     __atomic_store(std::addressof(__a->__a_value), std::addressof(__val), __to_gcc_order(__order));
   }
   ```

这里只列出了 `__cxx_atomic_store` 的实现，`__cxx_atomic_{store, load, exchange, compare_exchange_weak, compare_exchange_strong}` 的实现也是一样的，根据宏 `_LIBCPP_HAS_C_ATOMIC_IMP` 被定义还是宏 `_LIBCPP_HAS_GCC_ATOMIC_IMP` 被定义，分别调用 [Clang __c11_atomic builtins](https://clang.llvm.org/docs/LanguageExtensions.html#c11-atomic-builtins)、[GNU __atomic builtins](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html).。

简单总结：`std::atomic` 在 libc++ 中的实现，如果使用 Clang 编译器配合 libc++，那么 `std::atomic` 使用 C11 `_Atomic` 配合 [Clang __c11_atomic builtins](https://clang.llvm.org/docs/LanguageExtensions.html#c11-atomic-builtins) 实现的，如果使用 GCC 编译器配合 libc++，那么 `std::atomic` 是使用 [GNU __atomic builtins](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html) 的实现的。

前文提到：对于 builtins `__atomic_{load, store, exchange, compare_exchange}` 和 `__atomic_{load, store, exchange, compare_exchange}_n`，如果使用 GCC 编译器，使用者必须保证参数 `ptr` 指向的内存是自然对齐的(naturally-aligned)。而使用 GCC + libc++ 时，`std::atomic` 是直接使用 GNU __atomic builtins 的实现的，libc++ 并没有任何操作来保证对齐，所以 GCC + libc++ 的 `std::atomic` 存在不是 atomic 的情况，相关讨论见：

- https://bugs.llvm.org/show_bug.cgi?id=38871

- https://gcc.gnu.org/bugzilla/show_bug.cgi?id=87237

## 0x8. Epilogue

本文首先介绍了 `__atomic_*` builtins，包括 Clang 是如何编译 `__atomic_*` builtins，以 __atomic_load* 为例管中窥豹地介绍 GCC libatomic 是怎么实现 `__atomic_*` libcalls 的，然后介绍 C11 `_Atomic` 在 Clang 中的实现，最后介绍 `std::atomic` 在 libc++ 中是怎么实现的。

虽然本文内容很多，但是相信在阅读本文后对“std::atomic 在 libc++ 中是怎么实现的”会有比较清晰完整的理解。

最后回到本文在 "0x2. PR libstdc++/65147" 提出的问题：

> GCC 和 Clang 是否所有的 atomics implementation 都是 compatible ？

答案是否定的。有很多关于 GCC and Clang/LLVM incompatible 的讨论：

- https://gcc.gnu.org/bugzilla/show_bug.cgi?id=65146

- https://gcc.gnu.org/bugzilla/show_bug.cgi?id=96159

- https://gcc.gnu.org/bugzilla/show_bug.cgi?id=115954

## P.S.

### target-cpu, target-feature, tune-cpu

在编译时有很多 target 相关的编译选项，如 `-march=`*`cpu-type`*,`-mtune=`*`cpu-type`*, `-msse`, `-mavx` 等。

- https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html

- https://clang.llvm.org/docs/ClangCommandLineReference.html#x86

Clang 会将这些编译选项解析为 `target-cpu`, `target-feature` 和 `tune-cpu` 保存为 `clang::TargetOptions` 的成员变量，相关代码见 [clang/lib/Driver/ToolChains/Clang.cpp#L6032-L6039](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/Driver/ToolChains/Clang.cpp#L6032-L6039)。

对于 default target x86_64-unknown-linux-gnu 的 Clang，如果编译时不指定任何 target 相关的编译选项：

- `target-cpu` 默认是 "x86-64"。如果 target 是 i386-unknown-linux-gnu，那么 `target-cpu` 是 "pentium4"。相关代码见 [clang/lib/Driver/ToolChains/Arch/X86.cpp#L24](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/Driver/ToolChains/Arch/X86.cpp#L24)

- `target-feature` 基于编译选项和 `target-cpu` 计算得到，`target-cpu` 默认为 "x86-64" 时，`target-feature` 为 "+cmov,+cx8,+fxsr,+mmx,+sse,+sse2,+x87"。相关代码见 [clang/lib/Basic/Targets.cpp#L809-L831](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/Basic/Targets.cpp#L809-L831)

- `tune-cpu` 默认是 "generic" [clang/lib/Driver/ToolChains/Clang.cpp#L2333](https://github.com/llvm/llvm-project/blob/release/19.x/clang/lib/Driver/ToolChains/Clang.cpp#L2333)

举个例子 https://godbolt.org/z/zxx6WxnEG



```C++
void foo(_Atomic(__int128) *i) {
  __c11_atomic_store(i, 9981, __ATOMIC_SEQ_CST);
}
```

使用 x86_64-unknown-linux-gnu Clang 19.1.0 编译：

- 编译选项中添加 `-v`，可以看到传给 `-cc1` 的参数中出现 `-target-cpu x86-64 -tune-cpu generic`，在 LLVM IR 的 attributes 中有 `"target-cpu"="x86-64" "target-features"="+cmov,+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic"`

- 编译选项中添加 `-march=core-avx2`，可以看到可以看到传给 `-cc1` 的参数中出现 `-target-cpu core-avx2`，在 LLVM IR 的 attributes 中有 `"target-cpu"="core-avx2" "target-features"="+avx,+avx2,+bmi,+bmi2,+cmov,+crc32,+cx16,+cx8,+f16c,+fma,+fsgsbase,+fxsr,+invpcid,+lzcnt,+mmx,+movbe,+pclmul,+popcnt,+rdrnd,+sahf,+sse,+sse2,+sse3,+sse4.1,+sse4.2,+ssse3,+x87,+xsave,+xsaveopt"`

