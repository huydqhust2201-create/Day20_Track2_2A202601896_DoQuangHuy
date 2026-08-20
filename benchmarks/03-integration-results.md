# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.2 | 8516.5 | 8516.9 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 4983.8 | 4983.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 11594.1 | 11594.2 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **8364.8** · total **8365.0**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it filters out requests that do not meet specific targets (TTFT and TPOT), whereas raw throughput ignores SLOs.

The context explicitly states:
> "Throughput at saturation ignores SLOs."

This indicates that Goodput provides a more accurate and aligned measure of performance by adhering to Service Level Objective

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** by storing the Key-Value (KV) cache in non-contiguous pages.

This design allows the engine to remove the internal fragmentation that typically wastes most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps to **avoid computing the same token multiple times across different decoding steps**.

Here is the breakdown based on the provided context:

*   **Prefill is compute-bound:** It requires significant processing power (CPU/GPU) to calculate the model's parameters for every token.
*   **Decode is memory-bound:** It requires significant bandwidth to read data from me


## Which N16-N19 pieces are real

- **N16 (Cloud/IaC)** - stubbed. No k8s cluster or Compose stack; `llama-server`
  runs directly on `localhost:8080` on this laptop.
- **N17 (Data pipelines)** - stubbed. No Airflow DAG or batch job; the corpus
  is the static `TOY_DOCS` list defined in `pipeline.py`, loaded in-process.
- **N18 (Lakehouse)** - stubbed. No Delta/Iceberg table; `TOY_DOCS` is a plain
  Python list, not even the SQLite fallback the lab allows.
- **N19 (Vector + features)** - stubbed, and shown as its literal fallback:
  `embed_url` was not set, so `embed()` returned `None` and `retrieve()` fell
  back to keyword-overlap scoring instead of cosine similarity over real
  embeddings. That is why every `embed (ms)` in the table above is `0.0` - no
  embedding call was ever made, not a fast one.
- **N20 (Serving)** - real. This is the one piece the lab requires to be real,
  and it is: `llama-server` (Qwen3.5 0.8B, Q4_K_M) served all three completions
  over `/v1/chat/completions` on the running instance from track 02, and
  `/metrics`'s `tokens_predicted_total` moved with each call.

Dominant stage is **llm at 100%** - fully expected, and for a slightly more
extreme reason than "the LLM usually dominates": with `embed_url` unset,
`embed` is not merely small, it is exactly zero (no network call happens at
all), and keyword-overlap `retrieve` over 6 short toy docs is sub-millisecond
(0.1-0.2ms). So `llm` is not just the largest stage, it is effectively the
*only* stage with any real cost in this configuration - the RAG "framing" is
here, but the RAG *cost* is not, because nothing outside the LLM is doing real
work yet.

If I had to halve this pipeline's latency, I would attack **decode**, not
retrieval or embedding - because retrieval/embedding are currently ~0% of the
budget, so touching them wins nothing. `llm` time is dominated by decode, not
prefill (query 3: 307ms prefill vs 8634ms decode for 153 tokens - decode is
>25x the prefill cost there). Two concrete levers, in order of what this
lab already measured: (1) generate fewer tokens - `max_tokens=200` here is
generous for a factual QA answer and query 3's decode alone was 8.6s of the
11.6s total; capping around 80-100 tokens would cut the dominant stage
directly. (2) the `-t 1` thread-pool finding from track 01 (benchmarks/01-tuning-tg128.md)
applies here too, since decode is GPU-bound (`ngl=99`) and excess CPU threads
were pure coordination overhead on this machine - the server should be
launched with the tuned thread count, not the physical-core default, to avoid
paying that overhead on every one of these calls as well.
