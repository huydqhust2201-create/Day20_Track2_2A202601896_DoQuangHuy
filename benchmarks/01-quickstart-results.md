# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2717 | 308 / 329 | 34.1 / 41.5 | 2423 / 2942 / 2942 | 29.3 |
| UD-Q2_K_XL | 0.39 | 3117 | 342 / 408 | 49.4 / 62.2 | 3467 / 4143 / 4143 | 20.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.44x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Not worth it on this machine. `UD-Q2_K_XL` is only 0.11 GB smaller than
`Q4_K_M` (0.39 GB vs 0.50 GB - both already tiny at 0.8B params) but decodes
1.44x **slower** (20.3 vs 29.3 tok/s) and its TPOT P95 is also worse (62.2ms
vs 41.5ms). That inversion is the "compute-limited" branch of the note above,
not the "bandwidth-limited" branch: both quantizations run with `ngl=99`,
i.e. fully offloaded onto this machine's GPU (NVIDIA GeForce MX330, 2GB VRAM,
Pascal-class, compute capability 6.1 - no tensor cores, weak throughput).
`Q2_K` packs weights into odd bit-widths that need more per-weight
unpacking/dequant arithmetic per matmul than `Q4_K`'s cleaner nibble layout.
On a modern/fast GPU that dequant cost is hidden under abundant compute and
the smaller footprint wins on bandwidth; on the MX330 the GPU itself is the
bottleneck, so the extra unpacking work costs more cycles than the 110MB of
saved memory traffic buys back. This run's numbers are faster across the
board than an earlier cold-cache run of the same benchmark (weights already
resident in the OS page cache / GPU driver warmed up), which changes the
absolute numbers but not the ranking - `Q2_K_XL` was 1.88x slower cold and is
still 1.44x slower warm, so the compute-bound explanation holds regardless of
cache state.

I also compared answer quality directly: served each quant on :8080 and sent
the identical prompt ("Explain what a KV cache is in one paragraph.",
`max_tokens=150`, `temperature=0.2`) via `/v1/chat/completions`.

- `Q4_K_M`: 80 tokens, `finish_reason: stop` - one clean paragraph. It gets the
  *concept* wrong for the domain this course means ("KV cache" = attention
  key/value cache) and instead describes a generic key-value store/cache,
  but the prose itself is coherent and stops naturally.
- `Q2_K_XL`: made the same conceptual mistake, but at `max_tokens=150` it hit
  `finish_reason: length` - the model emitted a stray `</think>` tag (a
  reasoning-format artifact leaking into a non-reasoning answer) and then
  restarted the same paragraph almost verbatim instead of stopping, burning
  its whole token budget on a repeat.

So on this machine the 2-bit quant is strictly worse on the one axis that
matters for "worth it": it is slower, has a fatter P95 tail, and its output
quality regressed further - looping/repeating instead of terminating cleanly -
rather than being merely "a bit less accurate". Neither quant is a great
teacher on what a KV cache actually is (expected at 0.8B - this is a
serving-stack lab, not a quality benchmark), but between the two, `Q4_K_M` is
the only one worth deploying here.
