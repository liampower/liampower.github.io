---
title: "Brief Intro to SIMD"
date: 2020-06-03T10:49:24+01:00
draft: true
summary: "A (very) brief introduction to SIMD programming in C, using the Intel SSE instruction set. "
---

As microprocessors become more and more gated by heat, programmers seeking to
squeeze the most performance out of the hardware may need to explore more creative
techniques. SIMD (Single Instruction Multiple Data) is one such technique. SIMD
allows you to perform operations on multiple data at once (hence the name).
Consider the following example:

```c
// We want to compute the sum of these two arrays
void ComputeSums(float A[4], float B[4], float Out[4])
{
    Out[0] = A[0] + B[0];
    Out[1] = A[1] + B[1];
    Out[2] = A[2] + B[2];
    Out[3] = A[3] + B[3];
}
```

A compiler might generate the following assembly for this function:

```asm
ComputeSums(float*, float*, float*):
  movss   xmm0, dword ptr [rdi]
  addss   xmm0, dword ptr [rsi]
  movss   dword ptr [rdx], xmm0
  movss   xmm0, dword ptr [rdi + 4]
  addss   xmm0, dword ptr [rsi + 4]
  movss   dword ptr [rdx + 4], xmm0
  movss   xmm0, dword ptr [rdi + 8]
  addss   xmm0, dword ptr [rsi + 8]
  movss   dword ptr [rdx + 8], xmm0
  movss   xmm0, dword ptr [rdi + 12]
  addss   xmm0, dword ptr [rsi + 12]
  movss   dword ptr [rdx + 12], xmm0
  ret
```
Even without knowing assembly, you can probably follow this code pretty well;
each element is copied from memory into the `xmm0` register, then we perform
the additions with the `addss` instruction. There are four adds and eight moves,
not great.

So how would we go about optimising this function? There aren't any algorithmic
changes we can make, this is about as brain-dead simple as you can get. The key
is noticing that the instructions don't change, only the data does. This is
exactly the type of problem SIMD instructions were built to solve.

Let's begin to SIMD-ify our ComputeSums function. To use SIMD, we could re-write
our function in assembly and use the SSE instructions directly, though this
would be pretty time consuming and introduce unnecessary complexity. Fortunately, 
we can stick with C by using *intrinsics*, special functions that map directly to
machine instructions. For Clang, the SSE instrinsics are defined in the `xmmintrin.h`
header file. 

```c
#include <xmmintrin.h>

void ComputeSums_SSE(float A[4], float B[4], float Out[4])
{
    // Load our data into the SSE registers
    __m128 Va = _mm_load_ps(A);
    __m128 Vb = _mm_load_ps(B);

    // Perform the computation
    __m128 Vout = _mm_add_ps(Va, Vb);

    // Write the results back out to memory
    _mm_store_ps(Vout, Out);
}
```

Now lets see how the assembly changed:
```asm
ComputeSums_SSE(float*, float*, float*):
  movaps  xmm0, xmmword ptr [rdi]
  addps   xmm0, xmmword ptr [rsi]
  movaps  xmmword ptr [rdx], xmm0
  ret
```

Much better --- only three instructions! Notice the 'ps' prefix on all the instructions,
that's how we know we're in vector-land. PS means "packed single", i.e. multiple 
single-precision floating point numbers packed together. Contrast this to the 'ss' prefix
we saw in the earlier scalar code; SS means "scalar single", so one float at a time.


## The Critical Mistake
When people first learn about SIMD programming, almost everyone experiences a strong 
instinct to start doing this:
```c
// Old scalar version
typedef struct vec3
{
    float X, Y, Z;
} vec3;

// Shiny new SSE version. 4x Speed :D :D
typedef struct vec3
{
    __m128 XYZ;
} vec3;

vec3 AddVec3(vec3 A, vec3 B)
{
    return _mm_add_ps(A.XYZ, B.XYZ);
}

// Sub, mul, etc.
```
That is, replacing traditional vector data types with SIMD versions and using all
the SIMD instructions instead of scalar operations. At first glance, this seems like a
no-brainer: vector operations tend to be heavily used in multimedia applications (like
games), so surely we would want to speed these operations up by using the vector
instructions...right? SSE instructions map really well to vertical operations like
addition and subtraction, but things start to break down when you get to the dot product:

```c
float DotVec3(vec3 A, vec3 B)
{
    __m128 P = _mm_mul_ps(A.XYZ, B.XYZ); // Easy!

    float HorzSum = // ... uhhh, wait a minute

    return HorzSum;
}
```
This is because the dot product requires a *horizontal* operation: summing up all the
products. Horizontal operations are always a huge pain in SIMD programming; for SSE 
versions below SSE3, the only way to accomplish a horizontal add was with a complex
and error-prone series of shuffles (SSE3 added the `haddps` instruction, but it's so
slow you might lose the speedup gained from using SIMD in the first place). 


So what's the alternative? Is it impossible to do horizontal operations efficiently in
SIMD? No. We just need to shift our perspective a little. Instead of defining our data
like this:
```c
typedef struct vec3
{
    __m128 XYZ;
} vec3;
```
Instead, we can define four vec3s at once, like this:
```c
typedef struct vec3x4
{
    __m128 Xs;
    __m128 Ys;
    __m128 Zs;
} vec3x4;
```
I've heard this technique called "homogenous vector aggregate" (HVA).
Now the dot product becomes much easier:

```c
// Compute four dot products together
__m128 Dot3x4(vec3x4 A, vec3x4 B)
{
    __m128 ProdXs = _mm_mul_ps(A.Xs, B.Xs);
    __m128 ProdYs = _mm_mul_ps(A.Ys, B.Ys);
    __m128 ProdZs = _mm_mul_ps(A.Zs, B.Zs);

    __m128 HorzSum = _mm_add_ps(_mm_add_ps(ProdXs, ProdYs), ProdZs);

    return HorzSum;
}
```
