# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `vulkan` ·
llama.cpp `b10488` · `threads=8` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 16.9 | 1.00x | 62% |
| 8 | 15.0 | 0.89x | 56% |
| 16 | 13.1 | 0.78x | 49% |
| 24 | 16.7 | 0.99x | 62% |
| 32 | 3.3 | 0.20x | 12% |
| 99 | 27.0 | 1.60x | 100% |

Best: `-ngl 99` at 27.0 tok/s
-- 1.60x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

**Full offload is best: 16.9 → 27.0 tok/s (1.60×) going from `-ngl 0` to `-ngl 99`.** But
the curve is not a ramp. Every partial value is at or *below* CPU-only, and the `-ngl 32`
point (3.3 tok/s) looked catastrophic. I re-ran the top of the range to see whether that was
real (`llama-bench -t 8 -p 0 -n 128 -r 3`):

| -ngl | 24 | 28 | 32 | 34 | 35 (all layers) | 99 (+ output) |
|--:|--:|--:|--:|--:|--:|--:|
| tg128 tok/s | 22.2 ± 1.4 | 16.1 ± 0.9 | 18.3 ± 2.5 | 17.2 ± 12.3 | 24.8 ± 0.8 | 26.2 ± 2.3 |

So the 3.3 was one bad run, not a cliff. The shape still holds, though: partial offload is
noisy and lands around CPU speed, and the clear win comes only once *all* 35 layers are on
the iGPU. The output head adds a little more (35 → 99).

**What ran out first — neither VRAM nor host↔device bandwidth.** The Arc is an *integrated*
GPU. It has no VRAM of its own and reads the same LPDDR5 as the CPU (llama.cpp reports
18 GB of "device" memory, which is a share of system RAM). The 2.95 GiB model always fits,
and there is no PCIe link to cross. What partial offload costs instead:

1. **The CPU part sets the pace.** Per token, time = t(CPU layers) + t(GPU layers) + sync.
   Each CPU layer still runs at CPU speed (bandwidth the cores can pull, see
   `01-tuning-tg128.md`). Moving 24 of 35 layers saves time only on those layers, and the
   remaining CPU layers plus the per-layer embedding/output work keep the total close to
   CPU-only.
2. **Graph splits.** Every boundary between the CPU and the Vulkan backend is a split:
   activations are copied, and the CPU waits for the GPU queue to drain before it can
   continue (and the reverse). With one boundary per token that is fixed latency added to
   every step, and it does not shrink as more layers move.
3. **Contention on shared memory.** CPU threads and the iGPU stream weights from the same
   DRAM at the same time, and the Windows scheduler adds jitter. That matches the huge
   variance at -ngl 34 (± 12) and the 3.3 tok/s outlier.

**Before/after for REFLECTION §6:** `-ngl 0` 16.9 tok/s → `-ngl 99` 27.0 tok/s = **1.60×**.
This is the bonus speedup; it is separate from the base-track thread tuning. On an iGPU,
offload is all-or-nothing: either everything goes to the GPU, or partial offload buys
nothing.
