# Bonus C7 (B4) - CPU instruction-set survey

Host: Intel Core Ultra 7 155H (Meteor Lake, 6P + 8E + 2LP-E), Windows 11, 31.4 GB LPDDR5x.
Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf`. llama.cpp `b10488` (same commit `9d77fa172` for every binary).
All runs CPU-only (`-ngl 0`), `-t 8` (best from `make tune`), `-p 512 -n 128 -r 3`, **Release** builds only.

## What the CPU supports

- From the MSVC feature probe at configure time: `HAS_AVX_1` ✓, `HAS_AVX2_1` ✓, `HAS_AVX512_1/2` ✗.
- Meteor Lake adds AVX-VNNI (256-bit VEX encoding), BMI2, FMA and F16C, and has no AVX-512.
- The prebuilt runtime agrees: its CPU-variant dispatcher picks `ggml-cpu-alderlake.dll` on this machine (`llama-bench -v` → `load_backend: loaded CPU backend from ...ggml-cpu-alderlake.dll`).

## Builds compared

| Binary | Compiler | CPU flags actually compiled in (from CMake output) |
|:--|:--|:--|
| prebuilt release | Clang 20.1.8 | runtime dispatch → alderlake variant (AVX2, FMA, F16C, AVX-VNNI, BMI2) |
| `-DGGML_NATIVE=ON` | MSVC 19.44 | `/arch:AVX2 GGML_AVX2;GGML_FMA;GGML_F16C` |
| `-DGGML_NATIVE=OFF` (defaults) | MSVC 19.44 | `/arch:AVX2 GGML_AVX2;GGML_FMA;GGML_F16C;__BMI2__;GGML_BMI2` |
| `NATIVE=OFF` + `-DGGML_AVX_VNNI=ON` | MSVC 19.44 | above + `__AVXVNNI__;GGML_AVX_VNNI` |
| SSE4.2 only (`AVX/AVX2/FMA/F16C/BMI2=OFF`) | MSVC 19.44 | SSE4.2 |

## Numbers (tok/s, mean ± sd of 3 reps; two rounds, round 2 in reverse order)

| Binary | pp512 R1 | pp512 R2 | tg128 R1 | tg128 R2 |
|:--|--:|--:|--:|--:|
| prebuilt (Clang, alderlake DLL) — *op offload to iGPU on* | 351.1 ± 4.4 † | 285.7 ± 34.3 † | 18.89 ± 0.24 | 7.42 ± 7.10 |
| prebuilt, `-nopo 1` (op offload off, separate run) | — | 69.3 ± 3.4 | — | — |
| prebuilt, `-dev none` (separate run) | — | 89.2 ± 1.9 | — | — |
| MSVC NATIVE=ON (AVX2) | 87.7 ± 1.7 | 57.6 ± 8.7 | **19.68 ± 0.01** | 15.17 ± 3.76 |
| MSVC NATIVE=OFF (AVX2+BMI2) | 80.6 ± 1.2 | 101.1 ± 4.3 | 18.96 ± 0.57 | **18.68 ± 1.55** |
| MSVC + AVX-VNNI explicit | 68.9 ± 10.3 | 54.7 ± 12.8 | 17.72 ± 1.21 | 14.20 ± 2.79 |
| MSVC SSE4.2 only | 4.5 ± 0.2 | 4.1 ± 0.8 | 2.72 ± 0.06 | 2.49 ± 0.90 |

† Not a CPU number: even at `-ngl 0` the prebuilt (which carries the Vulkan backend) offloads large-batch prefill ops to the Arc iGPU. The two extra rows turn that off.

The official `make compare-builds` runs: tg128 17.1 (prebuilt) vs 19.3 (native), a tie within noise; pp512 343.4 vs 100.3,
which is the same op-offload artefact † (`bonus-build-compare-*.md`).

## Analysis

1. **Vector width matters up to AVX2.** SSE4.2 → AVX2 is the one ISA step that matters:
   **~7× for decode** (2.6 → ~19 tok/s) and **~15–20× for prefill** (4.5 → 60–100). Without
   256-bit FMA even single-token decode is instruction-bound. With it, decode hits the
   bandwidth/sync ceiling, and every AVX2 build (MSVC or Clang) lands at ~19 tok/s.
2. **Past AVX2, nothing in the ISA or the compiler moves the needle; the GPU does.** Turning
   AVX-VNNI on under MSVC did not help (within noise or slightly worse). The Clang-built
   prebuilt *seemed* 3.4–4× faster on prefill. I first read that as a code-generation gap,
   but with op offload disabled (`-nopo 1`: 69 tok/s; `-dev none`: 89 tok/s) it is level with
   MSVC (58–101 tok/s across runs). The extra prefill speed was the iGPU being used for
   large-batch prefill mat-muls even at `-ngl 0`. Decode (batch 1) is not offloaded, and there all
   AVX2 builds tie at ~19 tok/s. Clang vs MSVC and VNNI vs no VNNI: no measurable
   difference on this CPU for this model.
3. **"Native" is not automatically the best kernel, and "`-ngl 0`" is not automatically
   CPU-only.** With MSVC, `GGML_NATIVE=ON` only probes AVX/AVX2/AVX-512 (`HAS_*` tests). It
   silently skipped AVX-VNNI and BMI2, which `NATIVE=OFF` with defaults actually enabled.
   And a binary that ships a GPU backend uses it for prefill unless told not to (`-nopo 1` /
   `-dev none`). This is the same lesson as "FA3 for Hopper, FA4 for Blackwell": check what
   the build compiled in and which device actually ran the op, rather than trust the flag
   name.
4. **Noise is a first-class result on a hybrid CPU.** Round 2's prebuilt decode
   (7.4 ± 7.1) is the Windows scheduler putting some of the 8 threads on E/LP-E cores, which
   then hold up every barrier. Any build-vs-build claim under ~1.2× on this laptop needs
   several rounds before I would believe it.

**Recommendation for this machine:** keep the upstream prebuilt. Its CPU kernels are as good
as a local MSVC build, and because it carries the Vulkan backend it gets iGPU prefill even at
`-ngl 0`. For serving, use `ngl=99` on the Arc iGPU (`bonus-gpu-offload-sweep.md`). Building
from source only pays off here if it adds a backend (e.g. SYCL for Arc), not for CPU flags.
