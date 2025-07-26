---
title: "A whirlwind tour of the LLVM optimizer"
date: 2023-05-15
categories:
  - "Random"
comments: true # Enable Disqus comments for specific page
enableInlineShortcodes: true # hugo-embed-pdf-shortcode
---

todo list of learning llvm optimization passes.

<!--more-->

## Intro

最近在 Nikita Popov 在 EuroLLVM 2023 进行了题为 "A whirlwind tour of the LLVM optimizer" 的分享，目前只开放了 slide，youtube 上还没有更新视频。

我以前也阅读过一些 LLVM passes 的源码，但是感觉只是为了阅读而阅读，后来就慢慢地停下来了。在看到 Nikita Popov 的分享 slide 后，决心把 slide 中提到的所有 optimization passes 研究透，于是开坑本文作为记录。

## Optimization passes todo list :

- SSA Construction

  - Mem2Reg

  - SROA

- Control-Flow Optimization

  - SimplifyCFG

  - JumpThreading

- Instruction Combining(Peephole Optimization)

  - InstCombine

  - CorrelatedValuePropagation

- Redundancy Elimination

  - EarlyCSE

  - GVN

- Memory Optimizations

  - MemCpyOpt

  - DSE

- Loop Optimization

  - LICM

  - IndVarSimplify

  - LoopUnroll

- Vectorization

  - LoopVectorize

  - SLPVectorize

- Inter-Procedural Optimization (IPO)

  - FunctionAttrs

  - IPSCCP

## Attachment

{{< embed-pdf url="/blog/a-whirlwind-tour-of-the-llvm-optimizer/awhirlwindtourofthellvmoptimizer.pdf" >}}
