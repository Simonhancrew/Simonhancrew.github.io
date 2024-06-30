---
title: 现代c++并发编程6-协程
date: 2024-06-30 22:44:00 +0800
categories: [Blogging, gcc, optimize flags]
tags: [writing]
---

简单的记录一下gcc的优化选项，以及一些细节。

正常情况下，能选择开/关的编译器优化，只有有符号的哪些

你可以通过

```bash
gcc -c -Q -O3 --help=optimizers > /tmp/O3-opts
gcc -c -Q -O2 --help=optimizers > /tmp/O2-opts
diff /tmp/O2-opts /tmp/O3-opts | grep enabled
```

看哪些优化被开启了，O2和O3的区别。

## 默认优化-O0

O0是默认的优化选项，理论上是不进行任何优化，但是在查阅资料之后发现也有一些优化

```text
  -faggressive-loop-optimizations       [enabled]
  -fallocation-dce                      [enabled]
  -fasynchronous-unwind-tables          [enabled]
  -fauto-inc-dec                        [enabled]
  -fbit-tests                           [enabled]
  -fdce                                 [enabled]
  -fearly-inlining                      [enabled]
  -ffp-int-builtin-inexact              [enabled]
  -ffunction-cse                        [enabled]
  -fgcse-lm                             [enabled]
  -finline-atomics                      [enabled]
  -fipa-stack-alignment                 [enabled]
  -fipa-strict-aliasing                 [enabled]
  -fira-hoist-pressure                  [enabled]
  -fira-share-save-slots                [enabled]
  -fira-share-spill-slots               [enabled]
  -fivopts                              [enabled]
  -fjump-tables                         [enabled]
  -flifetime-dse                        [enabled]
  -fmath-errno                          [enabled]
  -fpeephole                            [enabled]
  -fplt                                 [enabled]
  -fprintf-return-value                 [enabled]
  -freg-struct-return                   [enabled]
  -fsched-critical-path-heuristic       [enabled]
  -fsched-dep-count-heuristic           [enabled]
  -fsched-group-heuristic               [enabled]
  -fsched-interblock                    [enabled]
  -fsched-last-insn-heuristic           [enabled]
  -fsched-rank-heuristic                [enabled]
  -fsched-spec                          [enabled]
  -fsched-spec-insn-heuristic           [enabled]
  -fsched-stalled-insns-dep             [enabled]
  -fschedule-fusion                     [enabled]
  -fsemantic-interposition              [enabled]
  -fshort-enums                         [enabled]
  -fshrink-wrap-separate                [enabled]
  -fsigned-zeros                        [enabled]
  -fsplit-ivs-in-unroller               [enabled]
  -fssa-backprop                        [enabled]
  -fstdarg-opt                          [enabled]
  -ftrapping-math                       [enabled]
  -ftree-forwprop                       [enabled]
  -ftree-loop-im                        [enabled]
  -ftree-loop-ivcanon                   [enabled]
  -ftree-loop-optimize                  [enabled]
  -ftree-phiprop                        [enabled]
  -ftree-reassoc                        [enabled]
  -ftree-scev-cprop                     [enabled]
  -funreachable-traps                   [enabled]
  -funwind-tables                       [enabled]
```

## -O1优化

简单的看下-O1的描述

```text
Optimize. Optimizing compilation takes somewhat more time, and a lot more memory for a large function. With -O, the compiler tries to reduce code size and execution time, without performing any optimizations that take a great deal of compilation time.
```

实际进行下面的优化

```text
  -faggressive-loop-optimizations       [enabled]
  -fallocation-dce                      [enabled]
  -fasynchronous-unwind-tables          [enabled]
  -fauto-inc-dec                        [enabled]
  -fbit-tests                           [enabled]
  -fbranch-count-reg                    [enabled]
  -fcombine-stack-adjustments           [enabled]
  -fcompare-elim                        [enabled]
  -fcprop-registers                     [enabled]
  -fdce                                 [enabled]
  -fdefer-pop                           [enabled]
  -fdse                                 [enabled]
  -fearly-inlining                      [enabled]
  -fforward-propagate                   [enabled]
  -ffp-int-builtin-inexact              [enabled]
  -ffunction-cse                        [enabled]
  -fgcse-lm                             [enabled]
  -fguess-branch-probability            [enabled]
  -fif-conversion                       [enabled]
  -fif-conversion2                      [enabled]
  -finline                              [enabled]
  -finline-atomics                      [enabled]
  -finline-functions-called-once        [enabled]
  -fipa-modref                          [enabled]
  -fipa-profile                         [enabled]
  -fipa-pure-const                      [enabled]
  -fipa-reference                       [enabled]
  -fipa-reference-addressable           [enabled]
  -fipa-stack-alignment                 [enabled]
  -fipa-strict-aliasing                 [enabled]
  -fira-hoist-pressure                  [enabled]
  -fira-share-save-slots                [enabled]
  -fira-share-spill-slots               [enabled]
  -fivopts                              [enabled]
  -fjump-tables                         [enabled]
  -flifetime-dse                        [enabled]
  -fmath-errno                          [enabled]
  -fmove-loop-invariants                [enabled]
  -fmove-loop-stores                    [enabled]
  -fomit-frame-pointer                  [enabled]
  -fpeephole                            [enabled]
  -fplt                                 [enabled]
  -fprintf-return-value                 [enabled]
  -freg-struct-return                   [enabled]
  -freorder-blocks                      [enabled]
  -fsched-critical-path-heuristic       [enabled]
  -fsched-dep-count-heuristic           [enabled]
  -fsched-group-heuristic               [enabled]
  -fsched-interblock                    [enabled]
  -fsched-last-insn-heuristic           [enabled]
  -fsched-rank-heuristic                [enabled]
  -fsched-spec                          [enabled]
  -fsched-spec-insn-heuristic           [enabled]
  -fsched-stalled-insns-dep             [enabled]
  -fschedule-fusion                     [enabled]
  -fsemantic-interposition              [enabled]
  -fshort-enums                         [enabled]
  -fshrink-wrap                         [enabled]
  -fshrink-wrap-separate                [enabled]
  -fsigned-zeros                        [enabled]
  -fsplit-ivs-in-unroller               [enabled]
  -fsplit-wide-types                    [enabled]
  -fssa-backprop                        [enabled]
  -fssa-phiopt                          [enabled]
  -fstdarg-opt                          [enabled]
  -fthread-jumps                        [enabled]
  -ftoplevel-reorder                    [enabled]
  -ftrapping-math                       [enabled]
  -ftree-bit-ccp                        [enabled]
  -ftree-builtin-call-dce               [enabled]
  -ftree-ccp                            [enabled]
  -ftree-ch                             [enabled]
  -ftree-coalesce-vars                  [enabled]
  -ftree-copy-prop                      [enabled]
  -ftree-dce                            [enabled]
  -ftree-dominator-opts                 [enabled]
  -ftree-dse                            [enabled]
  -ftree-forwprop                       [enabled]
  -ftree-fre                            [enabled]
  -ftree-loop-im                        [enabled]
  -ftree-loop-ivcanon                   [enabled]
  -ftree-loop-optimize                  [enabled]
  -ftree-phiprop                        [enabled]
  -ftree-pta                            [enabled]
  -ftree-reassoc                        [enabled]
  -ftree-scev-cprop                     [enabled]
  -ftree-sink                           [enabled]
  -ftree-slsr                           [enabled]
  -ftree-sra                            [enabled]
  -ftree-ter                            [enabled]
  -funwind-tables                       [enabled]
```

## -O2优化

O2相较于O1，他的描述显得激进了一点

```text
Optimize even more. GCC performs nearly all supported optimizations that do not involve a space-speed tradeoff. As compared to -O, this option increases both compilation time and the performance of the generated code.
```

在O1的基础上， O2还做了下面的这些优化

```text
<   -falign-functions                           [enabled]
<   -falign-jumps                               [enabled]
<   -falign-labels                              [enabled]
<   -falign-loops                               [enabled]
<   -fcaller-saves                              [enabled]
<   -fcode-hoisting                             [enabled]
<   -fcrossjumping                              [enabled]
<   -fcse-follow-jumps                          [enabled]
<   -fdevirtualize                              [enabled]
<   -fdevirtualize-speculatively                [enabled]
<   -fexpensive-optimizations                   [enabled]
<   -fgcse                                      [enabled]
<   -fhoist-adjacent-loads                      [enabled]
<   -findirect-inlining                         [enabled]
<   -finline-functions                          [enabled]
<   -finline-small-functions                    [enabled]
<   -fipa-bit-cp                                [enabled]
<   -fipa-cp                                    [enabled]
<   -fipa-icf                                   [enabled]
<   -fipa-icf-functions                         [enabled]
<   -fipa-icf-variables                         [enabled]
<   -fipa-ra                                    [enabled]
<   -fipa-sra                                   [enabled]
<   -fipa-vrp                                   [enabled]
<   -fisolate-erroneous-paths-dereference       [enabled]
<   -flra-remat                                 [enabled]
<   -foptimize-sibling-calls                    [enabled]
<   -foptimize-strlen                           [enabled]
<   -fpartial-inlining                          [enabled]
<   -fpeephole2                                 [enabled]
<   -free                                       [enabled]
<   -freorder-blocks-and-partition              [enabled]
<   -freorder-functions                         [enabled]
<   -frerun-cse-after-loop                      [enabled]
<   -fschedule-insns2                           [enabled]
<   -fstore-merging                             [enabled]
<   -fstrict-aliasing                           [enabled]
<   -ftree-loop-distribute-patterns             [enabled]
<   -ftree-loop-vectorize                       [enabled]
<   -ftree-pre                                  [enabled]
<   -ftree-slp-vectorize                        [enabled]
<   -ftree-switch-conversion                    [enabled]
<   -ftree-tail-merge                           [enabled]
<   -ftree-vrp                                  [enabled]
<   -funroll-loops                              [enabled]
```

## -O3优化

再额外扩充一下

```text
>   -fgcse-after-reload                         [enabled]
>   -fipa-cp-clone                              [enabled]
>   -floop-interchange                          [enabled]
>   -floop-unroll-and-jam                       [enabled]
>   -fpeel-loops                                [enabled]
>   -fpredictive-commoning                      [enabled]
>   -fsplit-loops                               [enabled]
>   -fsplit-paths                               [enabled]
>   -ftree-loop-distribution                    [enabled]
>   -ftree-partial-pre                          [enabled]
>   -funroll-completely-grow-size               [enabled]
>   -funswitch-loops                            [enabled]
>   -fversion-loops-for-strides                 [enabled]
```

剩下的哪些优化选项就自己后面再看了

## REF

1. [Options That Control Optimization](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)
2. [Options Controlling the Kind of Output](https://gcc.gnu.org/onlinedocs/gcc/Overall-Options.html)
