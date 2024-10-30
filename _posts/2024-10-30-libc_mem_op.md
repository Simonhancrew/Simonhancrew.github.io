---
title: libc关于mem*的实现
date: 2024-10-30 9:30:10 +0800
categories: [Blogging, unix, libc, memcpy, memmove, memset]
tags: [writing]
---

自己手写代码应该是拍马都赶不上lib里的了，都用linux下x86的实现来看看.

实际上这些mem相关的操作，在现代cpu上都有`rep ***`这种指令。

### memcpy

实现32和64的分离了，最后还是一套，代码在`arch/x86/lib/memcpy_64.S`和`arch/x86/lib/memcpy_32.S`里

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/string.h>
#include <linux/export.h>

#undef memcpy
#undef memset
#undef memmove

__visible void *memcpy(void *to, const void *from, size_t n)
{
	return __memcpy(to, from, n);
}
EXPORT_SYMBOL(memcpy);

__visible void *memset(void *s, int c, size_t count)
{
	return __memset(s, c, count);
}
EXPORT_SYMBOL(memset);
```

然后这玩意的实现是

```c
/* SPDX-License-Identifier: GPL-2.0-only */
/* Copyright 2002 Andi Kleen */

#include <linux/export.h>
#include <linux/linkage.h>
#include <linux/cfi_types.h>
#include <asm/errno.h>
#include <asm/cpufeatures.h>
#include <asm/alternative.h>

.section .noinstr.text, "ax"

/*
 * memcpy - Copy a memory block.
 *
 * Input:
 *  rdi destination
 *  rsi source
 *  rdx count
 *
 * Output:
 * rax original destination
 *
 * The FSRM alternative should be done inline (avoiding the call and
 * the disgusting return handling), but that would require some help
 * from the compiler for better calling conventions.
 *
 * The 'rep movsb' itself is small enough to replace the call, but the
 * two register moves blow up the code. And one of them is "needed"
 * only for the return value that is the same as the source input,
 * which the compiler could/should do much better anyway.
 */
SYM_TYPED_FUNC_START(__memcpy)
	ALTERNATIVE "jmp memcpy_orig", "", X86_FEATURE_FSRM

	movq %rdi, %rax
	movq %rdx, %rcx
	rep movsb
	RET
SYM_FUNC_END(__memcpy)
EXPORT_SYMBOL(__memcpy)

SYM_FUNC_ALIAS_MEMFUNC(memcpy, __memcpy)
EXPORT_SYMBOL(memcpy)

SYM_FUNC_START_LOCAL(memcpy_orig)
	movq %rdi, %rax

	cmpq $0x20, %rdx
	jb .Lhandle_tail

	/*
	 * We check whether memory false dependence could occur,
	 * then jump to corresponding copy mode.
	 */
	cmp  %dil, %sil
	jl .Lcopy_backward
	subq $0x20, %rdx
.Lcopy_forward_loop:
	subq $0x20,	%rdx

	/*
	 * Move in blocks of 4x8 bytes:
	 */
	movq 0*8(%rsi),	%r8
	movq 1*8(%rsi),	%r9
	movq 2*8(%rsi),	%r10
	movq 3*8(%rsi),	%r11
	leaq 4*8(%rsi),	%rsi

	movq %r8,	0*8(%rdi)
	movq %r9,	1*8(%rdi)
	movq %r10,	2*8(%rdi)
	movq %r11,	3*8(%rdi)
	leaq 4*8(%rdi),	%rdi
	jae  .Lcopy_forward_loop
	addl $0x20,	%edx
	jmp  .Lhandle_tail

.Lcopy_backward:
	/*
	 * Calculate copy position to tail.
	 */
	addq %rdx,	%rsi
	addq %rdx,	%rdi
	subq $0x20,	%rdx
	/*
	 * At most 3 ALU operations in one cycle,
	 * so append NOPS in the same 16 bytes trunk.
	 */
	.p2align 4
.Lcopy_backward_loop:
	subq $0x20,	%rdx
	movq -1*8(%rsi),	%r8
	movq -2*8(%rsi),	%r9
	movq -3*8(%rsi),	%r10
	movq -4*8(%rsi),	%r11
	leaq -4*8(%rsi),	%rsi
	movq %r8,		-1*8(%rdi)
	movq %r9,		-2*8(%rdi)
	movq %r10,		-3*8(%rdi)
	movq %r11,		-4*8(%rdi)
	leaq -4*8(%rdi),	%rdi
	jae  .Lcopy_backward_loop

	/*
	 * Calculate copy position to head.
	 */
	addl $0x20,	%edx
	subq %rdx,	%rsi
	subq %rdx,	%rdi
.Lhandle_tail:
	cmpl $16,	%edx
	jb   .Lless_16bytes

	/*
	 * Move data from 16 bytes to 31 bytes.
	 */
	movq 0*8(%rsi), %r8
	movq 1*8(%rsi),	%r9
	movq -2*8(%rsi, %rdx),	%r10
	movq -1*8(%rsi, %rdx),	%r11
	movq %r8,	0*8(%rdi)
	movq %r9,	1*8(%rdi)
	movq %r10,	-2*8(%rdi, %rdx)
	movq %r11,	-1*8(%rdi, %rdx)
	RET
	.p2align 4
.Lless_16bytes:
	cmpl $8,	%edx
	jb   .Lless_8bytes
	/*
	 * Move data from 8 bytes to 15 bytes.
	 */
	movq 0*8(%rsi),	%r8
	movq -1*8(%rsi, %rdx),	%r9
	movq %r8,	0*8(%rdi)
	movq %r9,	-1*8(%rdi, %rdx)
	RET
	.p2align 4
.Lless_8bytes:
	cmpl $4,	%edx
	jb   .Lless_3bytes

	/*
	 * Move data from 4 bytes to 7 bytes.
	 */
	movl (%rsi), %ecx
	movl -4(%rsi, %rdx), %r8d
	movl %ecx, (%rdi)
	movl %r8d, -4(%rdi, %rdx)
	RET
	.p2align 4
.Lless_3bytes:
	subl $1, %edx
	jb .Lend
	/*
	 * Move data from 1 bytes to 3 bytes.
	 */
	movzbl (%rsi), %ecx
	jz .Lstore_1byte
	movzbq 1(%rsi), %r8
	movzbq (%rsi, %rdx), %r9
	movb %r8b, 1(%rdi)
	movb %r9b, (%rdi, %rdx)
.Lstore_1byte:
	movb %cl, (%rdi)

.Lend:
	RET
SYM_FUNC_END(memcpy_orig)
```

暂时看不懂，我也不看别的arch了

### memmove

```c
/* SPDX-License-Identifier: GPL-2.0 */
/*
 * Normally compiler builtins are used, but sometimes the compiler calls out
 * of line code. Based on asm-i386/string.h.
 *
 * This assembly file is re-written from memmove_64.c file.
 *	- Copyright 2011 Fenghua Yu <fenghua.yu@intel.com>
 */
#include <linux/export.h>
#include <linux/linkage.h>
#include <asm/cpufeatures.h>
#include <asm/alternative.h>

#undef memmove

.section .noinstr.text, "ax"

/*
 * Implement memmove(). This can handle overlap between src and dst.
 *
 * Input:
 * rdi: dest
 * rsi: src
 * rdx: count
 *
 * Output:
 * rax: dest
 */
SYM_FUNC_START(__memmove)

	mov %rdi, %rax

	/* Decide forward/backward copy mode */
	cmp %rdi, %rsi
	jge .Lmemmove_begin_forward
	mov %rsi, %r8
	add %rdx, %r8
	cmp %rdi, %r8
	jg 2f

#define CHECK_LEN	cmp $0x20, %rdx; jb 1f
#define MEMMOVE_BYTES	movq %rdx, %rcx; rep movsb; RET
.Lmemmove_begin_forward:
	ALTERNATIVE_2 __stringify(CHECK_LEN), \
		      __stringify(CHECK_LEN; MEMMOVE_BYTES), X86_FEATURE_ERMS, \
		      __stringify(MEMMOVE_BYTES), X86_FEATURE_FSRM

	/*
	 * movsq instruction have many startup latency
	 * so we handle small size by general register.
	 */
	cmp  $680, %rdx
	jb	3f
	/*
	 * movsq instruction is only good for aligned case.
	 */

	cmpb %dil, %sil
	je 4f
3:
	sub $0x20, %rdx
	/*
	 * We gobble 32 bytes forward in each loop.
	 */
5:
	sub $0x20, %rdx
	movq 0*8(%rsi), %r11
	movq 1*8(%rsi), %r10
	movq 2*8(%rsi), %r9
	movq 3*8(%rsi), %r8
	leaq 4*8(%rsi), %rsi

	movq %r11, 0*8(%rdi)
	movq %r10, 1*8(%rdi)
	movq %r9, 2*8(%rdi)
	movq %r8, 3*8(%rdi)
	leaq 4*8(%rdi), %rdi
	jae 5b
	addq $0x20, %rdx
	jmp 1f
	/*
	 * Handle data forward by movsq.
	 */
	.p2align 4
4:
	movq %rdx, %rcx
	movq -8(%rsi, %rdx), %r11
	lea -8(%rdi, %rdx), %r10
	shrq $3, %rcx
	rep movsq
	movq %r11, (%r10)
	jmp 13f
.Lmemmove_end_forward:

	/*
	 * Handle data backward by movsq.
	 */
	.p2align 4
7:
	movq %rdx, %rcx
	movq (%rsi), %r11
	movq %rdi, %r10
	leaq -8(%rsi, %rdx), %rsi
	leaq -8(%rdi, %rdx), %rdi
	shrq $3, %rcx
	std
	rep movsq
	cld
	movq %r11, (%r10)
	jmp 13f

	/*
	 * Start to prepare for backward copy.
	 */
	.p2align 4
2:
	cmp $0x20, %rdx
	jb 1f
	cmp $680, %rdx
	jb 6f
	cmp %dil, %sil
	je 7b
6:
	/*
	 * Calculate copy position to tail.
	 */
	addq %rdx, %rsi
	addq %rdx, %rdi
	subq $0x20, %rdx
	/*
	 * We gobble 32 bytes backward in each loop.
	 */
8:
	subq $0x20, %rdx
	movq -1*8(%rsi), %r11
	movq -2*8(%rsi), %r10
	movq -3*8(%rsi), %r9
	movq -4*8(%rsi), %r8
	leaq -4*8(%rsi), %rsi

	movq %r11, -1*8(%rdi)
	movq %r10, -2*8(%rdi)
	movq %r9, -3*8(%rdi)
	movq %r8, -4*8(%rdi)
	leaq -4*8(%rdi), %rdi
	jae 8b
	/*
	 * Calculate copy position to head.
	 */
	addq $0x20, %rdx
	subq %rdx, %rsi
	subq %rdx, %rdi
1:
	cmpq $16, %rdx
	jb 9f
	/*
	 * Move data from 16 bytes to 31 bytes.
	 */
	movq 0*8(%rsi), %r11
	movq 1*8(%rsi), %r10
	movq -2*8(%rsi, %rdx), %r9
	movq -1*8(%rsi, %rdx), %r8
	movq %r11, 0*8(%rdi)
	movq %r10, 1*8(%rdi)
	movq %r9, -2*8(%rdi, %rdx)
	movq %r8, -1*8(%rdi, %rdx)
	jmp 13f
	.p2align 4
9:
	cmpq $8, %rdx
	jb 10f
	/*
	 * Move data from 8 bytes to 15 bytes.
	 */
	movq 0*8(%rsi), %r11
	movq -1*8(%rsi, %rdx), %r10
	movq %r11, 0*8(%rdi)
	movq %r10, -1*8(%rdi, %rdx)
	jmp 13f
10:
	cmpq $4, %rdx
	jb 11f
	/*
	 * Move data from 4 bytes to 7 bytes.
	 */
	movl (%rsi), %r11d
	movl -4(%rsi, %rdx), %r10d
	movl %r11d, (%rdi)
	movl %r10d, -4(%rdi, %rdx)
	jmp 13f
11:
	cmp $2, %rdx
	jb 12f
	/*
	 * Move data from 2 bytes to 3 bytes.
	 */
	movw (%rsi), %r11w
	movw -2(%rsi, %rdx), %r10w
	movw %r11w, (%rdi)
	movw %r10w, -2(%rdi, %rdx)
	jmp 13f
12:
	cmp $1, %rdx
	jb 13f
	/*
	 * Move data for 1 byte.
	 */
	movb (%rsi), %r11b
	movb %r11b, (%rdi)
13:
	RET
SYM_FUNC_END(__memmove)
EXPORT_SYMBOL(__memmove)

SYM_FUNC_ALIAS_MEMFUNC(memmove, __memmove)
EXPORT_SYMBOL(memmove)
```

### memset

```c
/* SPDX-License-Identifier: GPL-2.0 */
/* Copyright 2002 Andi Kleen, SuSE Labs */

#include <linux/export.h>
#include <linux/linkage.h>
#include <asm/cpufeatures.h>
#include <asm/alternative.h>

.section .noinstr.text, "ax"

/*
 * ISO C memset - set a memory block to a byte value. This function uses fast
 * string to get better performance than the original function. The code is
 * simpler and shorter than the original function as well.
 *
 * rdi   destination
 * rsi   value (char)
 * rdx   count (bytes)
 *
 * rax   original destination
 *
 * The FSRS alternative should be done inline (avoiding the call and
 * the disgusting return handling), but that would require some help
 * from the compiler for better calling conventions.
 *
 * The 'rep stosb' itself is small enough to replace the call, but all
 * the register moves blow up the code. And two of them are "needed"
 * only for the return value that is the same as the source input,
 * which the compiler could/should do much better anyway.
 */
SYM_FUNC_START(__memset)
	ALTERNATIVE "jmp memset_orig", "", X86_FEATURE_FSRS

	movq %rdi,%r9
	movb %sil,%al
	movq %rdx,%rcx
	rep stosb
	movq %r9,%rax
	RET
SYM_FUNC_END(__memset)
EXPORT_SYMBOL(__memset)

SYM_FUNC_ALIAS_MEMFUNC(memset, __memset)
EXPORT_SYMBOL(memset)

SYM_FUNC_START_LOCAL(memset_orig)
	movq %rdi,%r10

	/* expand byte value  */
	movzbl %sil,%ecx
	movabs $0x0101010101010101,%rax
	imulq  %rcx,%rax

	/* align dst */
	movl  %edi,%r9d
	andl  $7,%r9d
	jnz  .Lbad_alignment
.Lafter_bad_alignment:

	movq  %rdx,%rcx
	shrq  $6,%rcx
	jz	 .Lhandle_tail

	.p2align 4
.Lloop_64:
	decq  %rcx
	movq  %rax,(%rdi)
	movq  %rax,8(%rdi)
	movq  %rax,16(%rdi)
	movq  %rax,24(%rdi)
	movq  %rax,32(%rdi)
	movq  %rax,40(%rdi)
	movq  %rax,48(%rdi)
	movq  %rax,56(%rdi)
	leaq  64(%rdi),%rdi
	jnz    .Lloop_64

	/* Handle tail in loops. The loops should be faster than hard
	   to predict jump tables. */
	.p2align 4
.Lhandle_tail:
	movl	%edx,%ecx
	andl    $63&(~7),%ecx
	jz 		.Lhandle_7
	shrl	$3,%ecx
	.p2align 4
.Lloop_8:
	decl   %ecx
	movq  %rax,(%rdi)
	leaq  8(%rdi),%rdi
	jnz    .Lloop_8

.Lhandle_7:
	andl	$7,%edx
	jz      .Lende
	.p2align 4
.Lloop_1:
	decl    %edx
	movb 	%al,(%rdi)
	leaq	1(%rdi),%rdi
	jnz     .Lloop_1

.Lende:
	movq	%r10,%rax
	RET

.Lbad_alignment:
	cmpq $7,%rdx
	jbe	.Lhandle_7
	movq %rax,(%rdi)	/* unaligned store */
	movq $8,%r8
	subq %r9,%r8
	addq %r8,%rdi
	subq %r8,%rdx
	jmp .Lafter_bad_alignment
.Lfinal:
SYM_FUNC_END(memset_orig)
```
