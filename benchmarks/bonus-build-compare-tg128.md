# Bonus B1 - Prebuilt vs source build

Host `Windows-AMD64` · CPU `Intel(R) Core(TM) Ultra 7 155H`
Vector extensions detected: none
llama.cpp `b10488` both sides · `threads=8` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `tg128`, 3 repetitions

> **Backend mismatch, handled.** The prebuilt binary sees
> `['Vulkan0: Intel(R) Arc(TM) Graphics (18311 MiB, 17543 MiB free)']` and your source build sees `(no devices)`.
> Left at `-ngl 99` this comparison would have measured the accelerator and printed
> it under a compiler headline, so both sides were pinned to `-ngl 0`.

| Binary | Built for | tg128 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 17.1 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 19.3 | 1.13x |

On this machine, the source build is **1.13x faster**.

before: 17.1 tok/s (prebuilt release)
after:  19.3 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 1.13x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.



## Your explanation

**The 1.13× is within noise. For decode the two builds are effectively tied.** I re-ran
both binaries twice more in the C7 matrix (`bonus-c7-isa-survey.md`, `-t 8 -ngl 0 -r 3`):
- Round 1: prebuilt 18.9 ± 0.2, MSVC native 19.7 ± 0.0 (1.04×).
- Round 2: prebuilt 7.4 ± 7.1, MSVC native 15.2 ± 3.8. The scheduler noise on this hybrid
  CPU swamps the difference (see `01-tuning-tg128.md`).

A ~1.05–1.13× edge for decode is real-ish at best.

**Why decode cannot show a big compiler gain here:** my CPU (Core Ultra 7 155H) has AVX2,
FMA, F16C and AVX-VNNI, and *both* binaries use 256-bit AVX2 kernels. The prebuilt is not a
generic baseline: its Windows release dispatches at runtime to `ggml-cpu-alderlake.dll`.
Once the inner loop is 256-bit SIMD, single-token decode is limited by how fast the cores
pull ~1.3 GB of weights per token plus the per-op barrier, not by instructions, so better
code generation has nothing to speed up. The same thing seen from the other side: an
**SSE4.2-only** MSVC build drops to **2.6 tok/s (~7× slower)**. Without 256-bit FMA,
decode *does* become instruction-bound. The vector width matters up to AVX2, and past that
point bandwidth takes over.

**If anything favours MSVC here** it is small and plausibly the OpenMP runtime (MSVC
`vcomp` vs LLVM `libomp140` in the prebuilt) or which cores the threads land on. With ±1–7
tok/s of run-to-run noise I would not claim it. The decisive B1 result is the **prefill**
comparison (`bonus-build-compare-pp512.md`), where the prebuilt Clang build is 3.42×
faster.
