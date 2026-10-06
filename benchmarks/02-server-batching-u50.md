# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.93 of 4 slots (98%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 4429 |

Highest sampled value was **3.93 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

**Peak batch width: 3.93 of 4 slots.** Every sample during the run shows 3.87–3.93, with
`requests_processing = 4` and `requests_deferred` holding at 42–46. Continuous batching
was saturated for the whole test: all 4 slots decoded together in each step, and about 45
requests always waited behind them.

**Does it match the effective concurrency (21.4) in `02-server-results.md`?** It does not
need to, and the two numbers are consistent. Little's Law counts everything in flight,
including the queue. The server gauge counts only what is in a decode step.
The server saw `4 (processing) + ~46 (deferred) ≈ 50` requests in the system, which is
exactly the 50 users. The Little's-Law estimate (21.4) is lower mainly because the test is
too short for steady state: the average latency (~31 s) is half the 60 s run, so many
requests were still in flight when locust stopped, and only the 40 that finished count
towards RPS. Smaller effects are locust's `wait_time = between(0.2, 1.5)` and each user's
first connection paying the `localhost` → IPv6 fallback (~2 s, see
`03-integration-results.md`). I trust the server gauge for *slot utilisation* and use
Little's Law only as a lower bound on how much queueing there was.

**What batching bought:** from the counters, the server produced **40.8 tok/s** aggregate
during the run (10.9 decode steps/s × ~3.9 sequences), against 24.4 tok/s for a single
stream in `make bench`. That is **1.7× more throughput**, but each decode step slowed from
~41 ms to ~92 ms. With 4 sequences per step, the iGPU is no longer only streaming weights
(which would make a batch of 4 almost free). The KV reads and attention for 4 sequences,
plus interleaved prefill of the long-RAG prompts, push the step time up. So each user sees
TPOT get worse while the server as a whole goes faster: the throughput-vs-latency trade.
