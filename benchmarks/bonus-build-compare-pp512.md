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

**The prebuilt "wins" prefill by 3.42×, but that is not a compiler result. It is the iGPU,
used even at `-ngl 0`.** I first blamed Clang vs MSVC code generation. Then I tested the one
difference that `-ngl 0` does not remove: the prebuilt Windows asset carries the Vulkan
backend, my MSVC build does not. llama.cpp has *op offload*: even with every layer's weights
on the CPU, large-batch ops (prefill mat-muls) can be sent to an available GPU backend,
with the weights copied over on the fly. Decode (batch 1) stays on the CPU.

Same prebuilt binary, `-ngl 0 -t 8 -r 3` (`llama-bench`):

| Prebuilt run | pp128 | pp512 |
|:--|--:|--:|
| default (op offload on) | 221.6 ± 2.5 | **309.0 ± 15.9** |
| `-nopo 1` (op offload off) | 80.5 ± 1.4 | **69.3 ± 3.4** |
| `-dev none` (no GPU device at all) | - | **89.2 ± 1.9** |
| **MSVC native source build** (no GPU backend) | 95.7 ± 9.1 | **82.1 ± 1.2** |

With op offload disabled, the prebuilt and my build are **within noise of each other** (69–89
vs 82 tok/s). The fingerprint was in the data from the start. Pure-CPU prefill is flat or
slightly *down* from pp128 to pp512 (MSVC 96 → 82). The prebuilt *rises* from 222 to 309
because a bigger batch amortises the cost of shipping weights to the GPU better.

**So the real B1 finding has two parts:**
1. **Compiler/ISA (what B1 is meant to measure):** prefill is a tie. Both builds run 256-bit
   AVX2 kernels; the prebuilt dispatches to `ggml-cpu-alderlake.dll` (AVX2 + AVX-VNNI) at
   runtime, my MSVC build is `/arch:AVX2`. On this CPU and this workload the extra
   VNNI/Clang path buys nothing measurable.
2. **The comparison harness has a blind spot.** `compare-builds.py` pins both sides to
   `-ngl 0` on the assumption that this "isolates the compiler". That assumption fails
   whenever one binary carries a GPU backend, because op offload still uses the GPU for
   prefill. A fair CPU-vs-CPU comparison needs `-nopo 1` or `-dev none` on the
   GPU-capable side. (I did not change the lab script; the table above is the corrected
   comparison.)

Practical takeaway for an iGPU laptop: even "CPU-only" serving on the Vulkan build gets
~3–4× faster prefill for free from op offload, while decode is untouched.
