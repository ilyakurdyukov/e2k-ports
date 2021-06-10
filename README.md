## e2k-ports

Performance patches and build fixes for Elbrus (e2k) architecture.

This is my personal repository so that patches won't get lost.

### Elbrus porting cheat sheet:

Elbrus 2000 (aka e2k) is a 64-bit little-endian architecture.  
The compiler is mostly GCC compatible (defines `__GNUC__`), EDG frontend.

#### detection

- shell: `uname -m` returns `e2k`
- cmake: `if({CMAKE_SYSTEM_PROCESSOR} STREQUAL "e2k")`
- C preprocessor: `if defined(__e2k__)`
- compiler version: if `__LCC__ = 125` and `__LCC_MINOR__ = 9` then it's "LCC 1.25.09"
- architecture version: defined in `__iset__` (less than 3 is obsolete, 6 is the latest at the moment)

#### intrinsics

- MMX, SSE2, SSSE3, SSE4.1* - native support
- AVX, AVX2 - supported, but not recommended, uses too much CPU registers
- SSE4.2 and _mm_dp_ps (from SSE4.1) - emulated, slow, do not use

The compiler enables MMX to AVX2 support by default, pass `-mno-avx` (`-mno-sse4.2`) if code depends on the presence of macros (e.g. `#if defined(__AVX2__)`).

#### builtins

- __sync*, __atomic* - supported by the compiler
- count leading/trailing zeros - supported (__builtin_clz, __builtin_ctz)
- memory fence - supported (need to include `x86intrin.h` first)
    - __builtin_ia32_mfence, __builtin_ia32_lfence, __builtin_ia32_sfence


#### cpuid

Use compile time CPU detection, select the best SIMD up to SSE4.1.

#### rdtsc

```c
#include <x86intrin.h>
uint64_t time = __rdtsc();
// same: unsigned aux; uint64_t time = __rdtscp(&aux);
```

#### useful pragmas

`_Pragma("name")` - to use from macros.

Use before the loop:

- `#pragma ivdep` - ignore data dependencies inside the loop
- `#pragma unroll(n)` - unroll cycle N times

#### makecontext

Instead of `makecontext(ctx, ...)` use `makecontext_e2k(ctx, ...)`, returns a negative integer on error. Allocates extra resources that need to be freed using `freecontext_e2k(ctx)`.

#### nop

Use `__asm__ __volatile__ ("nop")` or `_mm_pause()` for a little delay.
