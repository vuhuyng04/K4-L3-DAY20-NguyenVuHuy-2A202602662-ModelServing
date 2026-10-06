# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=16` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 32 | 0.55 | 12000 | 33000 | 34000 | 8.6 | 0.0% |
| 50 | 40 | 0.68 | 34000 | 56000 | 58000 | 21.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.23x** (25% of linear) |
| P95 latency | **1.70x** |
| Effective concurrency at 50 users | 21.4 vs `--parallel 4` slots (occupancy/slot ratio 5.34) |

**Saturated.** Throughput delivered only 1.23x for 5x the offered load, and effective concurrency (21.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.23x while P95 moved 1.70x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**The server is already saturated at 10 users.** The number that convinced me is
**effective concurrency 8.6 at 10 users vs 4 slots**: twice as many requests were in the
system as could be decoding. Going to 50 users then bought almost nothing: RPS
0.55 → 0.68 (1.23× for 5× the offered load), while P50 nearly tripled (12 → 34 s) and P95
rose 33 → 56 s. During the 50-user run `make metrics` showed 3.93/4 busy slots and ~46
deferred requests for the whole minute (`02-server-batching-u50.md`).

**The extra latency is queue time, not compute.** A short request costs ~48 decode
steps. At the measured ~92 ms per batched step that is ~4.5 s of compute, and the fastest
short request under load took 5.6 s (10 users) and 6.3 s (50 users). The service time did
not change between the two runs, but the P50 for short requests went 11 s → 29 s. The
difference is time spent waiting for a slot, as `requests_deferred ≈ 46` shows directly.
With all 4 slots busy, the server produced ~41 tok/s aggregate (from the counters during
the 50-user run); with no free slot, extra users can only join the queue.

**Goodput at an SLO.** I pick an SLO of *end-to-end ≤ 15 s for a short chat request*.
- At 10 users: ~66 % of short requests finish within 12 s and 75 % take ≥ 26 s, so about
  70 % meet it → goodput ≈ 0.48 × 0.7 ≈ **0.34 req/s**.
- At 50 users: the short P50 is 29 s, so almost none meet it → goodput **≈ 0**, even though
  raw RPS went *up*.

That is the goodput argument in one line: throughput rose 23 %, useful throughput
collapsed.

**What I would change first: `--parallel` 4 → 8, together with `--ctx-size` 2048 → 4096.**
Batching is working and has headroom. Going from 1 to 4 sequences per step raised the step
time only 2.2× (41 → 92 ms) for 4× the tokens, so the iGPU is not yet compute-bound, and
more slots should convert some queue time into throughput. The context must grow with the
slots: llama-server divides `--ctx-size` across slots (`n_ctx_slot = 512` today), and 256
tokens per slot would truncate the long-RAG prompts. I would pick this before a smaller
quant, because Q2 is 11× *slower* on this iGPU (`01-quickstart-results.md`), and before
more threads, because with `ngl=99` the CPU threads are not the bottleneck. If that is not
enough, the next lever is admission control: rejecting or shedding requests past ~10
in-flight keeps the admitted ones inside the SLO instead of making everyone miss it.
