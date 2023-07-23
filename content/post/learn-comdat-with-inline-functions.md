---
title: "Learn COMDAT with inline functions"
date: 2023-07-23
categories:
  - "Programming"
tags:
  - "LLVM"
katex: false
comments: true # Enable Disqus comments for specific page
toc: true
---

本文通过一个 inline 函数的例子学习 COMDAT 的作用。

<!--more-->

考虑以下例子：

```cpp
//--- a.cpp
inline const char* hello() { return "Hello,World!"; }
void foo() { printf("%s: %s\n", __FILE__, hello()); }
int main() {
  return 0;
}

//--- b.cpp
inline const char* hello() { return "Hello,World!"; }
void bar() { printf("%s: %s\n", __FILE__, hello()); }

// clang a.cpp b.cpp -o a.out
```

该例子，在 a.cpp 中定义了 inline 函数 `hello()`，在 b.cpp 中同样定义了 inline 函数 `hello()`，最终在编译链接产物 a.out 中只会保留一份 inline 函数 `hello()` 的定义。

那么在 a.out 中只保留一份 inline 函数 `hello()` 的定义是如何实现呢？

注：对于一个 inline 函数，如果其在不同编译单元的定义不一样，则是未定义行为。

## Weak definition

使用 `readelf` 可以看到 a.out 中函数 `hello()` 的符号是 weak 的。

```
$ readelf -sW a.out | grep hello
    50: 0000000000401170    16 FUNC    WEAK   DEFAULT   13 _Z5hellov
```

对于在多个编译单元都有定义的 weak 符号，链接器是不会报错该符号存在重复定义的，并且链接器会将所有对该符号的引用指向该符号的第一处定义。

那么 inline 函数是否就是单纯将函数设置为 weak 符号呢？

可以直接写个测试用例验证下：

```cpp
//--- weak_a.cpp
__attribute__((weak)) const char* hello() { return "Hello,World!"; }
void foo() { printf("%s: %s\n", __FILE__, hello()); }
int main() {
  return 0;
}

//--- weak_b.cpp
__attribute__((weak)) const char* hello() { return "Hello,World!"; }
void bar() { printf("%s: %s\n", __FILE__, hello()); }

// clang weak_a.cpp weak_b.cpp -o a.out.weak
```

虽然使用 `readelf` 可以看到 a.out.weak 中函数 `hello()` 的符号属性是一模一样的，但是仔细看 `objdump` 反汇编 a.out.weak 的结果：

```
0000000000401130 <_Z5hellov>:
  401130:       55                      push   %rbp
  401131:       48 89 e5                mov    %rsp,%rbp
  401134:       48 b8 04 20 40 00 00    movabs $0x402004,%rax
  40113b:       00 00 00
  40113e:       5d                      pop    %rbp
  40113f:       c3                      retq

0000000000401170 <main>:
  401170:       55                      push   %rbp
  401171:       48 89 e5                mov    %rsp,%rbp
  401174:       31 c0                   xor    %eax,%eax
  401176:       c7 45 fc 00 00 00 00    movl   $0x0,-0x4(%rbp)
  40117d:       5d                      pop    %rbp
  40117e:       c3                      retq
  40117f:       90                      nop
  401180:       55                      push   %rbp
  401181:       48 89 e5                mov    %rsp,%rbp
  401184:       48 b8 04 20 40 00 00    movabs $0x402004,%rax
  40118b:       00 00 00
  40118e:       5d                      pop    %rbp
  40118f:       c3                      retq
```

在 a.out.weak 中只有一个函数 `hello` 的符号，但是仔细看 `main` 函数的反汇编 ，`40117f` 处就是 `main` 函数的结束，`401180 - 40118f` 这段汇编指令与 `401130 - 40113f` 这段函数 `hello` 的汇编指令是一模一样的，`401180 - 40118f` 这段汇编指令其实就是定义在 b.cpp 的函数 `hello` 的实现。

这说明单纯将函数 `hello` 设置为 weak 符号，虽然使得最终链接产物只有一个 `hello` 符号，但是 .text section 中实际上还是保留了定义在不同编译单元的 `hello` 的代码/汇编指令。

对比 `objdump` 反汇编 a.out 的结果，将函数 `hello` 设置为 inline 使得 .text section 只保留了一份 `hello` 代码/汇编指令。

```
0000000000401160 <main>:
  401160:       55                      push   %rbp
  401161:       48 89 e5                mov    %rsp,%rbp
  401164:       31 c0                   xor    %eax,%eax
  401166:       c7 45 fc 00 00 00 00    movl   $0x0,-0x4(%rbp)
  40116d:       5d                      pop    %rbp
  40116e:       c3                      retq
  40116f:       90                      nop

0000000000401170 <_Z5hellov>:
  401170:       55                      push   %rbp
  401171:       48 89 e5                mov    %rsp,%rbp
  401174:       48 b8 12 20 40 00 00    movabs $0x402012,%rax
  40117b:       00 00 00
  40117e:       5d                      pop    %rbp
  40117f:       c3                      retq
```

## COMDAT/section group

那么在 a.out 中只保留一份 inline 函数 `hello()` 的定义是如何实现呢？

对比 a.cpp 与 weak_a.cpp、b.cpp 与 weak_b.cpp 对应的 LLVM IR：

```
; clang -S -emit-llvm a.cpp -o -
$_Z5hellov = comdat any

@.str.2 = private unnamed_addr constant [13 x i8] c"Hello,World!\00", align 1

; Function Attrs: noinline nounwind optnone uwtable
define linkonce_odr dso_local i8* @_Z5hellov() #2 comdat {
entry:
  ret i8* getelementptr inbounds ([13 x i8], [13 x i8]* @.str.2, i64 0, i64 0)
}

; clang -S -emit-llvm weak_a.cpp -o -
@.str = private unnamed_addr constant [13 x i8] c"Hello,World!\00", align 1

; Function Attrs: noinline optnone uwtable
define weak dso_local i8* @_Z5hellov() #0 {
entry:
  ret i8* getelementptr inbounds ([13 x i8], [13 x i8]* @.str, i64 0, i64 0)
}
```

```
; clang -S -emit-llvm b.cpp -o -
$_Z5hellov = comdat any

@.str.2 = private unnamed_addr constant [13 x i8] c"Hello,World!\00", align 1

; Function Attrs: noinline nounwind optnone uwtable
define linkonce_odr dso_local i8* @_Z5hellov() #2 comdat {
entry:
  ret i8* getelementptr inbounds ([13 x i8], [13 x i8]* @.str.2, i64 0, i64 0)
}

; clang -S -emit-llvm weak_b.cpp -o -
@.str = private unnamed_addr constant [13 x i8] c"Hello,World!\00", align 1

; Function Attrs: noinline optnone uwtable
define weak dso_local i8* @_Z5hellov() #0 {
entry:
  ret i8* getelementptr inbounds ([13 x i8], [13 x i8]* @.str, i64 0, i64 0)
}
```

a.cpp 与 weak_a.cpp、b.cpp 与 weak_b.cpp 对应的 LLVM IR 之间的关键区别在于 `$_Z5hellov = comdat any`！

根据 https://llvm.org/docs/LangRef.html#comdats

> Comdat IR provides access to object file COMDAT/section group functionality which represents interrelated sections.
>
> Comdats have a name which represents the COMDAT key and a selection kind to provide input on how the linker deduplicates comdats with the same key in two different object files. A comdat must be included or omitted as a unit. Discarding the whole comdat is allowed but discarding a subset is not.
>
> A global object may be a member of at most one comdat.
>
> For selection kinds other than `nodeduplicate`, only one of the duplicate comdats may be retained by the linker and the members of the remaining comdats must be discarded.
>
> For selection kind `any`, the linker may choose any COMDAT key, the choice is arbitrary.

a.cpp 和 b.cpp 对应的 LLVM IR 中都创建了一个 key 为 `_Z5hellov` 的 COMDAT，并且 a.cpp 中定义的函数 `hello` 属于 a.cpp 中的 `_Z5hellov` COMDAT，b.cpp 中定义的函数 `hello` 属于 b.cpp 中的 `_Z5hellov` COMDAT。在链接时，对于 a.cpp 和 b.cpp 中的 `_Z5hellov` COMDAT，链接器会任意选择其中一个 `_Z5hellov` COMDAT 保留，并丢弃另外的 `_Z5hellov` COMDAT。假设链接器选择保留 a.cpp 的 `_Z5hellov` COMDAT，那么会丢弃掉 b.cpp 的 `_Z5hellov` COMDAT，b.cpp 的 `_Z5hellov` COMDAT 中的函数 `hello` 也就会被丢弃。

`readelf` 提供了一个选项 `-g --section-groups` 查看 section groups，查看 a.cpp 和 b.cpp 对应的目标文件 a.o 和 b.o 的 section groups：

```
$ readelf -g a.o
COMDAT group section [    4] `.group' [_Z5hellov] contains 2 sections:
   [Index]    Name
   [    5]   .text._Z5hellov
   [    6]   .rela.text._Z5hellov

$ readelf -g b.o
COMDAT group section [    4] `.group' [_Z5hellov] contains 2 sections:
   [Index]    Name
   [    5]   .text._Z5hellov
   [    6]   .rela.text._Z5hellov
```

通过 `objdump` 查看 a.o 的 .text._Z5hellov section 的内容，可以看到 .text._Z5hellov section 中就是函数 `hello` 的定义：

```
$ objdump -d a.o
Disassembly of section .text._Z5hellov:
0000000000000000 <_Z5hellov>:
   0:        55                           push   %rbp
   1:        48 89 e5                     mov    %rsp,%rbp
   4:        48 b8 00 00 00 00 00         movabs $0x0,%rax
   b:        00 00 00
   e:        5d                           pop    %rbp
   f:        c3                           retq
```

作为对比，weak_a.cpp 对应的目标文件 weak_a.o 则没有任何 section group：

```
$ readelf -g weak_a.o
There are no section groups in this file.
```

除了 COMDAT 外，a.cpp 与 weak_a.cpp、b.cpp 与 weak_b.cpp 对应的 LLVM IR 还有一点区分：a.cpp/b.cpp 对应的 LLVM IR 中函数 `hello` 的 linkage type 是 linkonce_odr，而 weak_a.cpp/weak_b.cpp 对应的 LLVM IR 中函数 `hello` 的 linkage type 是 weak。关于 linkage type 的详细解释见 https://llvm.org/docs/LangRef.html#linkage-types 。

## Usage of COMDAT

如果我们想通过 LLVM IR 层面插桩的方式实现为链接产物添加一个函数定义 `foo`，那么可以编写一个 <u><a href="https://llvm.org/doxygen/classllvm_1_1ModulePass.html">ModulePass</a></u>，为每一个 Module 都插入一个 `foo` 函数定义。因为 C++ 项目会有很多个编译单元，所以为每个编译单元执行该 ModulePass 会为每个编译单元都插入一个 `foo` 函数定义。尽管我们可以将函数 `foo` 的 linkage type 设置为 linkonce_odr，但是还是会使得最终链接产物中有多份 `foo` 函数的副本，增加 .text section 的大小。这个时候就可以利用 COMDAT，在 ModulePass 中创建函数`foo` 时同时创建一个 key 为 foo 的 COMDAT，将函数`foo`添加至该 COMDAT 中，这样在最终链接产物中就只会保留一份 `foo` 函数了。

```cpp
void insertFunctionFoo(llvm::Module &M) {
  llvm::FunctionType *FTT = llvm::FunctionType::get(Int8PtrTy, false);
  llvm::FunctionCallee FC = M.getOrInsertFunction("foo", FTT);
  llvm::Function *F = dyn_cast<llvm::Function>(FC.getCallee());
  if (F && F->isDeclaration()) {
    F->setComdat(M.getOrInsertComdat("foo"));
    F->setLinkage(llvm::GlobalValue::LinkOnceODRLinkage);
    llvm::BasicBlock *BB =
        llvm::BasicBlock::Create(M.getContext(), "entry", F);
    // Create a constant string "Hello,World"
    // We use private linkage for module-local string, and set the unnamed_addr
    // attribute to make it can be merged with another one.
    llvm::Constant *StrConst = llvm::ConstantDataArray::getString(
        M.getContext(), "Hello,World");
    llvm::GlobalVariable *GV = new llvm::GlobalVariable(
        M, StrConst->getType(), true, llvm::GlobalValue::PrivateLinkage,
        StrConst, "hello_world_string_constant");
    GV->setUnnamedAddr(llvm::GlobalValue::UnnamedAddr::Global);
    // Strings may not be merged w/o setting alignment explicitly.
    GV->setAlignment(llvm::Align(1));
    llvm::ReturnInst::Create(
        M.getContext(),
        llvm::ConstantExpr::getPointerCast(GV, Int8PtrTy), BB);
  }
}
```

## Reference

https://maskray.me/blog/2021-07-25-comdat-and-section-group
