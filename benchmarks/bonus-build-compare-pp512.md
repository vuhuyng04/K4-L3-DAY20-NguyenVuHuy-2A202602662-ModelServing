# Bonus B1 - Prebuilt vs source build

Host `Windows-AMD64` · CPU `Intel(R) Core(TM) Ultra 7 155H`
Vector extensions detected: none
llama.cpp `b10488` both sides · `threads=8` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `pp512`, 3 repetitions

> **Backend mismatch, handled.** The prebuilt binary sees
> `['Vulkan0: Intel(R) Arc(TM) Graphics (18311 MiB, 17543 MiB free)']` and your source build sees `(no devices)`.
> Left at `-ngl 99` this comparison would have measured the accelerator and printed
> it under a compiler headline, so both sides were pinned to `-ngl 0`.

| Binary | Built for | pp512 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 343.4 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 100.3 | 0.29x |

On this machine, the prebuilt binary is **3.42x faster**.

before: 343.4 tok/s (prebuilt release)
after:  100.3 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 0.29x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.



## Your explanation

**The prebuilt binary wins prefill by 3.42×, and the line above ("the only difference is
what the compiler was allowed to assume") is not true on Windows.** The two binaries differ
in three ways, not one:

| | prebuilt `llama-b10488-bin-win-vulkan-x64` | my source build |
|:--|:--|:--|
| Compiler | Clang 20.1.8 | MSVC 19.44 (VS 2022 Build Tools) |
| CPU code | `GGML_CPU_ALL_VARIANTS`: 14 `ggml-cpu-*.dll`, picked at runtime → loads **`ggml-cpu-alderlake.dll`** (AVX2 + FMA + F16C + **AVX-VNNI** + BMI2) | one `ggml-cpu.dll`, `/arch:AVX2` (+FMA, F16C). MSVC "native" detection found AVX/AVX2 only, **no VNNI** |
| OpenMP | LLVM `libomp140` | MSVC `vcomp` (`-openmp`) |

So "prebuilt = generic baseline" does not hold here. Upstream's Windows release *already*
ships a Meteor-Lake-class kernel set and dispatches to it at load time (`load_backend:
loaded CPU backend from ...ggml-cpu-alderlake.dll`).

**Why the gap is large on prefill but not on decode** (`bonus-build-compare-tg128.md`:
17.1 vs 19.3 tok/s): pp512 is a 512-row mat-mul, **compute-bound**. Every weight byte
loaded is reused across 512 tokens, so speed tracks how good the inner AVX2 kernel is.
tg128 does one token at a time and is bandwidth/sync-bound, so any decent AVX2 kernel hits
the same ceiling.

**What I ruled out with extra builds (C7, `bonus-c7-isa-survey.md`):** I built MSVC with
AVX-VNNI explicitly on (`-DGGML_AVX_VNNI=ON`, which defined `__AVXVNNI__`), and prefill
did **not** improve (69 / 55 tok/s vs 88 / 58 for plain AVX2), so VNNI alone is not the
3.4×. Thread synchronisation is not it either: decode, which runs just as many barriers per
token, is a tie. The remaining difference is the compiler's code generation for the
quantised AVX2 dot-product/mat-mul kernels (Clang vs MSVC). I could not prove this
directly because Clang (clang-cl) is not installed in my Build Tools. The next experiment
would be the same source with `-T ClangCL`. Practical takeaway: on Windows, "build it
yourself with -DGGML_NATIVE=ON" makes prefill *slower* unless you also use the compiler
upstream uses.
