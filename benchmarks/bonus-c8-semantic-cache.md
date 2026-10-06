# Bonus C8 (B5) - Semantic cache in front of llama-server

Host `Windows-AMD64` (Core Ultra 7 155H, Arc iGPU, Vulkan) · llama.cpp `b10488`
Chat: `gemma-4-E2B-it-UD-Q4_K_XL.gguf` on :8080 (`ngl=99`, 4 slots).
Embeddings: the same Gemma model in pooling mode on :8081 (`serve.py --embedding`); the lab ships no dedicated embedder.
Command: `.venv\Scripts\python bonus\serving-regimes\semantic-cache-demo.py --sweep --chat-url http://127.0.0.1:8080/v1 --embed-url http://127.0.0.1:8081/v1 [--threshold T]`
(`127.0.0.1` instead of `localhost` on purpose: on this Windows box `localhost` adds ~2 s per new connection, see `03-integration-results.md`.)
Screenshot: `submission/screenshots/09-bonus-semantic-cache.png`.

## Numbers

Similarity is the best cosine against everything already in the cache. The values were identical across all three real runs.

| # | Prompt | Truth | sim | T = 0.80 | T = 0.86 |
|--:|:--|:--|--:|:--|:--|
| 1 | What is goodput at SLO? | new | 0.00 | miss (3.9 s) | miss |
| 2 | Explain TTFT and TPOT. | new | 0.55 | miss | miss |
| 3 | Can you define goodput@SLO? | paraphrase of #1 | **0.72** | miss **(false miss)** | miss |
| 4 | What does time to first token mean? | paraphrase of #2 | **0.75** | miss **(false miss)** | miss |
| 5 | How does PagedAttention work? | new | 0.64 | miss | miss |
| 6 | Tell me what goodput@SLO is. | paraphrase of #1 | 0.80 | HIT (0 ms) | miss |
| 7 | What is prefix caching? | **new topic** | **0.85** | HIT **(false hit)** | miss |
| 8 | Describe how PagedAttention works. | paraphrase of #5 | 0.85 | HIT (0 ms) | miss |
| | **Correct decisions** | | | 5/8 (2 false misses, 1 false hit) | 4/8 (all 4 paraphrases missed) |

Before/after for the LLM call a hit replaces: a miss cost **3.6–4.5 s** of full inference, while a hit returned in **0 ms**. That saving only counts when the hit is correct.

Offline control (`--offline --sweep`, bag-of-words stub embedder): the same 3/8 hits at every threshold 0.70–0.95, because bag-of-words similarity is either 1.0 or 0.0. It catches #3, #6 and #8 (shared content words), misses #4 ("TTFT" vs "time to first token" share no word), and correctly rejects #7.

## Analysis

**No single threshold works.** The false hit (#7, *prefix caching*, sim **0.85**) scores
*higher* than a true paraphrase (#6, sim **0.80**) and ties another (#8, **0.85**).
- Any threshold that rejects #7 (T > 0.85) also rejects #6 and #8. At T = 0.86 the cache
  never hits at all.
- Catching the false misses #3 (0.72) and #4 (0.75) needs T ≤ 0.72, and #7 (0.85) stays a
  false hit at any such threshold, along with any other "What is <term>?" question that
  scores in the same 0.7–0.85 band.

The scores are not ordered by meaning, so moving the threshold only trades one error for
the other.

**Why: a next-token decoder is not a sentence encoder.** Gemma was trained to predict the
next token. Its hidden states encode "what comes next given this prefix", and mean-pooling
them mostly captures the *shape* of the prompt: a short English question about an LLM
serving term. That is why every pair lands in a narrow 0.55–0.85 band and "What is X?"
looks like "What is Y?". A dedicated embedder (Qwen3-Embedding, BGE-M3, EmbeddingGemma) is
trained contrastively, pulling paraphrase pairs together and pushing unrelated pairs
apart, often with instruction prefixes and a pooled/normalised output head. It is
optimised for exactly the separation this cache needs. Interestingly, the crude
bag-of-words stub made *fewer* errors here (1 vs 3) because it at least cannot confuse two
different terms. It just cannot see synonyms (#4).

**What this means for the 3-layer cache stack.** Layer 1 (semantic) saves 100 % of the
compute on a hit, but a wrong hit returns a wrong answer with full confidence, which is
worse than a slow correct one. With this embedder I would not enable it. With a proper
embedder I would still gate it per intent (FAQ-like traffic only) and keep layer 2
(llama.cpp's prefix/KV cache) as the safe default. Layer 2 only reuses byte-identical
prefixes, so it can never return a wrong answer. In `03-integration-results.md` it cut
prefill from ~113–149 tokens to 5 on a repeated prompt.

**Security.** A semantic cache (or a shared prefix cache) shared across users is a timing
side channel: a 0 ms response reveals that *someone* asked a semantically close question.
Production systems salt or partition the cache key per tenant, so one user's hits can
never be another user's evidence.
