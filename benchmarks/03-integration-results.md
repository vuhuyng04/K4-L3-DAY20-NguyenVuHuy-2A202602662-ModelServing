# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 4100.4 | 4100.5 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 3682.1 | 3682.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 3657.1 | 3657.2 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **3813.2** · total **3813.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Day | Piece | Real or stub |
|:--|:--|:--|
| N16 Cloud/IaC | everything runs locally on my laptop, no provisioned infra | **stub** (not used) |
| N17 Data pipeline | the 6 `TOY_DOCS` are hard-coded in `pipeline.py`, no ingestion job | **stub** |
| N18 Lakehouse | no table/storage layer, docs live in memory | **stub** |
| N19 Vector + features | `retrieve()` is the keyword-overlap fallback, no embedding server, no vector index (`embed` = 0 ms) | **stub** |
| N20 Serving | `llama-server` b10488, Gemma 4 E2B UD-Q4_K_XL on the Arc iGPU | **real** |

**Is the dominant stage what I expected?** That the LLM is ~100 % was expected:
retrieval over 6 in-memory docs costs 0.1 ms. The *size* of the LLM stage was not. The
server reports only ~1.4–1.6 s per query (e.g. prefill 149 tok / 486 ms + decode 30 tok /
1102 ms), but the pipeline measured 3.7–4.1 s. I traced the missing ~2 s to name
resolution, not to the model:

```
urllib GET http://localhost:8080/health   -> 2067 ms, 2041 ms
urllib GET http://127.0.0.1:8080/health   ->    3 ms,    2 ms
```

llama-server binds only `127.0.0.1`. On Windows, `localhost` resolves to `::1` first, the
IPv6 connect is refused, and the retry on IPv4 costs ~2 s on every new connection. Re-running
the same pipeline with `--base-url http://127.0.0.1:8080` gave a mean llm stage of
**1340 ms** instead of 3813 ms. About 2.05 s of that drop is the IPv6 fallback. The rest
(~0.4 s) is llama.cpp's prompt cache: on the identical re-run, prefill fell from ~113–149
tokens to 5 tokens (~125 ms) because the prompt prefix was already in the slot's KV cache.

**To halve this pipeline's latency** I would attack, in order:
1. **Connection setup:** use `127.0.0.1` or a shared `httpx.Client` with keep-alive (`call_llm`
   uses module-level `httpx.post`, which opens a new connection per query). That alone is −65 %, zero model change.
2. **Prefill, via prefix caching:** put the fixed system prompt and the most-reused
   contexts first, so the cached prefix is reused across queries (`cache_prompt`). The
   re-run shows prefill dropping ~4×.
3. **Decode length:** the answers are one or two sentences, so a lower `max_tokens` and a
   "be brief" instruction cut decode, which is now ~70 % of the true LLM time.

With real N19 retrieval (embedding + vector search) the retrieve/embed stages would show up,
but they would still be tens of milliseconds against seconds of LLM time.
