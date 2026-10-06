# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **16 physical · 22 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 5.6 | 35% |
| 8 | 15.8 | 100% |
| 16 | 13.5 | 85% |
| 22 | 9.6 | 60% |
| 44 | 5.5 | 35% |

**Best**: `-t 8` at 15.8 tok/s
**Slowest tested**: `-t 44` at 5.5 tok/s (2.85x spread)
**Against the physical-core default** (`-t 16`, 13.5 tok/s): 1.17x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

**Why `ngl=0`:** the Vulkan build sees the Arc iGPU and offloads everything by default, and
then `-t` barely matters. I ran this sweep with `LAB_N_GPU_LAYERS=0` on purpose, so it
measures what the thread count does to CPU decode.

**Supplementary points** (same model, `llama-bench -ngl 0 -p 0 -n 128 -r 4`, run right
after `make tune`):

| -t | 4 | 6 | 8 | 12 | 16 |
|--:|--:|--:|--:|--:|--:|
| tg128 tok/s | 14.65 ± 0.48 | 9.20 ± 5.62 | 16.47 ± 0.92 | 16.70 ± 1.26 | 13.46 ± 11.64 |

**The knee is at ~8 threads, not at the 16 "physical cores".** Gains stop early: 4 threads
already give 14.7 tok/s (~90 % of the best), and 8–12 threads plateau at ~16.5. A
1-thread → 4-thread speedup of 2.6× followed by a flat line is the signature of
bandwidth-bound decode. Every generated token streams the whole active weight set
(~1.3 GB at Q4) out of DRAM, and once a few cores keep the memory controller busy, more
cores have nothing to compute while they wait.

**Why 16 is worse than 8, and why it is so noisy.** The Core Ultra 7 155H is a hybrid chip:
6 P-cores (with HT), 8 E-cores and 2 low-power E-cores on the SoC tile = 16 cores / 22
threads. llama.cpp splits every mat-mul evenly across `-t` threads, with a barrier after
each op, so each step runs at the speed of the **slowest** thread. At `-t 16` some threads
land on E/LP-E cores, which are slower and further from the P-cores' L3. The barrier then
waits for those stragglers, and when the Windows scheduler also moves a thread around, a
whole run drops (16 threads: 13.5 ± 11.6). At 22 threads the HT siblings share P-core
execution units, and at 44 the threads oversubscribe the cores and spin on barriers (9.6 →
5.5 tok/s, same as 1 thread).

**Something that differs from the deck:** at ~16.5 tok/s × ~1.3 GB, CPU decode pulls only
~20 GB/s, far below the LPDDR5x's theoretical peak. So "bandwidth-bound" here means bound
by how much bandwidth the CPU cores *can actually pull*, plus the barrier/sync cost — not
the DRAM spec number. The Arc iGPU, on the same memory, reaches 24 tok/s (`ngl=99`,
`01-quickstart-results.md`). That fits: a GPU keeps many more memory requests in flight
than a handful of CPU cores.

**Decision:** for CPU runs use `LAB_N_THREADS=8` (1.17× over the default 16, and far more
stable). On this machine the faster setup is still the iGPU with `ngl=99`.
