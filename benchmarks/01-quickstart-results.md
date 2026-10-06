# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=16` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 15741 | 455 / 2608 | 40.9 / 41.4 | 3034 / 5175 / 5175 | 24.4 |
| UD-Q2_K_XL | 2.24 | 9721 | 1199 / 13794 | 451.2 / 457.9 | 29641 / 37585 / 37585 | 2.2 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **11.09x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

**On this machine 2-bit is not worth it — it is 11× slower, not faster.** UD-Q2_K_XL saves
0.73 GB (−25 %) and loads faster (9.7 s vs 15.7 s), but decode drops from 24.4 to 2.2 tok/s
(TPOT P50 40.9 → 451 ms) and TTFT P50 goes 455 → 1199 ms.

The deck's "fewer bits → fewer bytes per token → faster decode" argument assumes the kernel
for that format is as efficient as the 4-bit one. Here it is not. This run used the Vulkan
build with all layers offloaded to the Intel Arc iGPU (`ngl=99`). I isolated the cause with
`llama-bench` (`-t 8 -p 128 -n 64`):

| | tg64 GPU (`ngl=99`) | tg64 CPU (`ngl=0`) | pp128 GPU (`ngl=99`) | pp128 at `ngl=0` ‡ |
|:--|--:|--:|--:|--:|
| UD-Q4_K_XL | 23.2 | 16.5 | 405 | 196 |
| UD-Q2_K_XL | 6.0 | 16.8 | 175 | 122 |

‡ At `ngl=0` this Vulkan build still offloads large-batch prefill ops to the iGPU (op offload, found later in `bonus-build-compare-pp512.md`), so this column is not pure CPU. The decode columns are unaffected (batch 1).

- On **CPU** the two quants decode at the same speed (16.5 vs 16.8 tok/s): 25 % fewer bytes
  is paid back by the more expensive Q2_K/IQ dequantization, so decode is not purely
  bandwidth-bound there.
- On the **Arc iGPU** Q2 collapses (6 tok/s in llama-bench, 2.2 under llama-server with 4
  slots). The Vulkan mat-vec path for Q2_K on this iGPU is far slower than the Q4_K one.
  The iGPU shares the same LPDDR5 as the CPU, so there is no bandwidth win to compensate.

**Quality:** I asked both servers the same questions (`temperature=0`): a time sum
(14:35 + 2 h 50 → both said 17:25), primes between 30 and 60 (both correct), and "why is
decode bandwidth-bound" (both coherent; Q2 was more repetitive). On short prompts I saw no
quality loss, but that is a small sample.

**Verdict:** keep Q4. Q2 only makes sense when the 0.7 GB matters (a < 4 GB RAM box) — and
on this hardware it should then run on the CPU (`ngl=0`), not the iGPU.
