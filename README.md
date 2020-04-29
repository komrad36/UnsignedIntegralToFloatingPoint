# Notes on and optimal approaches for conversion between floating-point and integral types on x86-64 #

## Requirements of the Standard ##

According to the C++ standard (conv.fpint) and IEEE 754:

Floating-point types to integral types: conversion truncates, regardless of current FPU rounding mode. Undefined behavior occurs if the truncated value cannot fit.

Integral types to floating-point types: if IEEE arithmetic is supported, the current rounding mode is used to determine the nearest representable value. Undefined behavior occurs if the truncated value cannot fit.

## Complications for conversion to/from unsigned integers on x86 ##

With the arrival of AVX512F, unsigned 32- and 64-bit integers (referred to as U32s and U64s from here on out, and similarly for their signed friends, I32s and I64s) can be converted directly to/from float or double.

However, at the time of writing most of us don't have processors with AVX512F support; nor do the current nor the next generation of game consoles, the Xbox Series X and the PS5.

Therefore, it's not necessarily straightforward to perform conversions between float/double and an unsigned integral type, while conforming to the requirements of the Standard.

As evidence of this, CL (MSVC), the Microsoft C Compiler, had a codegen bug all the way until MSVC 2019 (CL version 19.20), which I will discuss in more detail later, in which conversion from U64 to float and U64 to double produced codegen that computes the WRONG ANSWER.

Furthermore, even though none of the three major compilers generates incorrect code anymore, NONE of them generates optimal code for all cases, either.

This document discusses the fastest Standard-compliant approaches the author has derived of all the various permutations of conversions between unsigned integral types and floating-point types for x86-64 processors without AVX512F, as well as the state of compiler codegen for each.

## Good news: everybody does U32s optimally ##

Because hardware support for conversion between I64 and float/double is available via the cvtsi2ss, cvtsi2sd, cvttss2si, and cvttsd2si instructions, and because a U32 can fit in an I64, all three major compilers generate correct, optimal codegen for conversion to/from U32, by simply treating the U32 as zero-extended out into an I64.

## Really bad news: MSVC prior to MSVC 2019 (CL 19.20) generates INCORRECT code ##

This is an example of a U64 that is converted improperly to both float AND double by MSVC. Note that the conversion is ONLY incorrect at runtime; at compile-time it's okay.

Example value:

`9223372586610590721 == 0x8000008000000401`

In each case below, the bit pattern of this value is shown first, then the result after casting, broken up into bit pattern and lined up for easy verification. It is clear that for both float and double, in this example, with rounding mode set to nearest (the default), the result should have been rounded up such that the low bit of the mantissa is set.

It is unset after casting in both cases.

```
Cast to float (incorrect):
          1000000000000000000000001000000000000000000000000000010000000001
0 10111110 00000000000000000000000

Correct:
          1000000000000000000000001000000000000000000000000000010000000001
0 10111110 00000000000000000000001


Cast to double (incorrect):
             1000000000000000000000001000000000000000000000000000010000000001
0 10000111110 0000000000000000000000010000000000000000000000000000

Correct:
             1000000000000000000000001000000000000000000000000000010000000001
0 10000111110 0000000000000000000000010000000000000000000000000001
```

This is because MSVC's approach to converting U64s with the high bit set was to convert as I64, then if the high bit was set, add the floating-point value of 2 to the 64th power to the conversion result to reconstitute the unsigned result. This doesn't work; it doesn't round correctly.

If you hit this problem, you'll need to either update to MSVC 2019 or construct an explicit replacement using the ideas below.


## Bad news: nobody does U64s optimally ##

Let's consider each permutation in turn.

### float to U64 ###

All three major compilers can be outperformed on this task. gcc and MSVC hit memory and have branches. Clang hits memory but does not branch. Clang's codegen is the best of the three. However, the following code still outperforms.

There are two good options, one that loads a float constant from memory to help it do its thing, and one that does not touch data memory at all.

The typical tradeoff applies here: the one that touches memory is slightly faster IF the memory is hot in cache, e.g. on repeated frequent calls to this function. However for sporadic calls where the memory is likely to be cold, the otherwise slightly slower non-memory version may be preferable. I present both.

Neither of these can be expressed in C++ in such a way that compilers generate optimal codegen, unfortunately. Assembly only for now.

```
Memory:

<const data section>
kC DWORD 5f000000h

<code>
vcvttss2si rax, xmm0
vsubss xmm0, xmm0, dword ptr [kC]
vcvttss2si rcx, xmm0
bts rcx, 63
cmovnc rax, rcx
ret

No memory:

mov eax, 5f000000h
vmovd xmm1, eax
vcvttss2si rax, xmm0
vsubss xmm1, xmm0, xmm1
vcvttss2si rcx, xmm1
bts rcx, 63
cmovnc rax, rcx
ret
```

### double to U64 ###

All three major compilers can be outperformed on this task. gcc and MSVC hit memory and have branches. Clang hits memory but does not branch. Clang's codegen is the best of the three. However, the following code still outperforms.

There are two good options, one that loads a double constant from memory to help it do its thing, and one that does not touch data memory at all.

This time, I generally recommend the nonmemory version as the latency is well-hidden. I present both.

Neither of these can be expressed in C++ in such a way that compilers generate optimal codegen, unfortunately. Assembly only for now.

```
Memory:

<const data section>
kC QWORD 43e0000000000000h

<code>
vcvttsd2si rax, xmm0
vsubsd xmm0, xmm0, qword ptr [kC]
vcvttsd2si rcx, xmm0
bts rcx, 63
cmovnc rax, rcx
ret

No memory:

mov rax, 43e0000000000000h
vmovq xmm1, rax
vcvttsd2si rax, xmm0
vsubsd xmm1, xmm0, xmm1
vcvttsd2si rcx, xmm1
bts rcx, 63
cmovnc rax, rcx
ret
```

### U64 to float ###

All three major compilers actually generate good code for this, and they all generate the same thing. Their solution does not hit memory but it does branch.

NOTE that MSVC generates INCORRECT CODEGEN for this task prior to MSVC 2019 (CL 19.20).

There are two good options. Neither needs to touch memory. One branches, one is branchless but slower. The branch condition is whether the high bit of the U64 is set. I recommend the branchy version if the branch is likely to be predicted well, and the branchless for random or otherwise unpredictable data. If in doubt, just go with the branchy version, which is what compilers generate. I present both.

A good C++ equivalent is available for the branchy version which produces optimal codegen. No such option is available for branchless.

If you're just stuck on old MSVC and are trying to workaround the compiler bug without having to write assembly, just use the C++ version.

```
Branchy:

C++:

static inline float FastU64ToFloat(U64 x)
{
    if ((I64)x >= 0)
        return (float)(I64)x;

    const float f = (float)(I64)((x >> 1) | (x & 1));
    return f + f;
}

ASM, Microsoft x64 calling convention (Windows...)

vxorps xmm0, xmm0, xmm0
test rcx, rcx
js L1
vcvtsi2ss xmm0, xmm0, rcx
ret
L1:
mov rax, rcx
shr rax, 1
and ecx, 1
or rax, rcx
vcvtsi2ss xmm0, xmm0, rax
vaddss xmm0, xmm0, xmm0
ret

ASM, System V x64 calling convention (Linux, BSD...)

vxorps xmm0, xmm0, xmm0
test rdi, rdi
js L1
vcvtsi2ss xmm0, xmm0, rdi
ret
L1:
mov rax, rdi
shr rax, 1
and edi, 1
or rax, rdi
vcvtsi2ss xmm0, xmm0, rax
vaddss xmm0, xmm0, xmm0
ret

Branchless:

ASM, Microsoft x64 calling convention (Windows...)

mov rdx, rcx
mov rax, rcx
shr rax, 1
and ecx, 1
or rax, rcx
xor ecx, ecx
test rdx, rdx
cmovs rdx, rax
cmovs rcx, rax
vcvtsi2ss xmm0, xmm0, rdx
vcvtsi2ss xmm1, xmm1, rcx
vaddss xmm0, xmm0, xmm1
ret

ASM, System V x64 calling convention (Linux, BSD...)

mov rdx, rdi
mov rax, rdi
shr rax, 1
and edi, 1
or rax, rdi
xor edi, edi
test rdx, rdx
cmovs rdx, rax
cmovs rdi, rax
vcvtsi2ss xmm0, xmm0, rdx
vcvtsi2ss xmm1, xmm1, rdi
vaddss xmm0, xmm0, xmm1
ret
```

### U64 to double ###

Only Clang generates optimal code for this.

NOTE that MSVC generates INCORRECT CODEGEN for this task prior to MSVC 2019 (CL 19.20).

There are two good options: mem or no mem. Neither needs to branch.

A good C++ equivalent is available for the mem version which produces optimal codegen. No such option is available for nonmem.

Clang generates the mem version.

If you're just stuck on old MSVC and are trying to workaround the compiler bug without having to write assembly, just use the C++ version.

```
Mem:

C++:

static inline double FastU64ToDouble(U64 x)
{
	const __m128i k1 = _mm_set_epi64x(0x0000000000000000, 0x4530000043300000);
	const __m128d k2 = _mm_castsi128_pd(_mm_set_epi64x(0x4530000000000000, 0x4330000000000000));
	__m128d v = _mm_castsi128_pd(_mm_unpacklo_epi32(_mm_cvtsi64_si128(x), k1));
	v = _mm_sub_pd(v, k2);
	v = _mm_hadd_pd(v, v);
	return _mm_cvtsd_f64(v);
}

ASM:

<const data section>
kC1 OWORD 00000000000000004530000043300000h
kC2 OWORD 45300000000000004330000000000000h

<code>

Microsoft x64 calling convention (Windows...)

vmovq xmm0, rcx
vpunpckldq xmm0, xmm0, xmmword ptr [kC1]
vsubpd xmm0, xmm0, xmmword ptr [kC2]
vhaddpd xmm0, xmm0, xmm0
ret

System V x64 calling convention (Linux, BSD...)

vmovq xmm0, rdi
vpunpckldq xmm0, xmm0, xmmword ptr [kC1]
vsubpd xmm0, xmm0, xmmword ptr [kC2]
vhaddpd xmm0, xmm0, xmm0
ret


No mem:

ASM, Microsoft x64 calling convention (Windows...)

vmovq xmm0, rcx
mov rax, 4530000043300000h
vmovq xmm1, rax
vpunpckldq xmm0, xmm0, xmm1
vpshufd xmm1, xmm1, 72h
vsubpd xmm0, xmm0, xmm1
vhaddpd xmm0, xmm0, xmm0
ret

System V x64 calling convention (Linux, BSD...)

vmovq xmm0, rdi
mov rax, 4530000043300000h
vmovq xmm1, rax
vpunpckldq xmm0, xmm0, xmm1
vpshufd xmm1, xmm1, 72h
vsubpd xmm0, xmm0, xmm1
vhaddpd xmm0, xmm0, xmm0
ret
```
