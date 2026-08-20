# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 34 | 0.61 | 15000 | 21000 | 22000 | 8.6 | 0.0% |
| 50 | 32 | 0.60 | 25000 | 53000 | 53000 | 16.6 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.98x** (20% of linear) |
| P95 latency | **2.52x** |
| Effective concurrency at 50 users | 16.6 vs `--parallel 4` slots (occupancy/slot ratio 4.15) |

**Saturated.** Throughput delivered only 0.98x for 5x the offered load, and effective concurrency (16.6) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.98x while P95 moved 2.52x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server is already saturated **at 10 users**, not somewhere between 10 and
50. The number that convinced me: RPS barely moves between the two runs (0.61
-> 0.60 req/s, flat within noise) while offered load went up 5x. If there were
headroom left at 10 users, throughput at 50 would have grown with it; instead
it is flat, which is the signature of a system already at its ceiling before
the second run started. The `n_busy_slots_per_decode` gauge from `make
metrics` (sampled during a 50-user run) confirms it directly: it sat at
**3.7-3.82 of 4 slots** for the entire 60s window, i.e. essentially all 4
decode slots busy essentially all the time - no spare slot capacity to absorb
more concurrent requests, and `requests_deferred` sat at 33-46 throughout the
same window (see `benchmarks/02-server-batching-u50.md`). Effective
concurrency via Little's Law backs this up too (16.6 "in flight" against 4
real slots, a 4.15x occupancy/slot ratio at 50 users), but the slot gauge is
the stronger evidence because it is the server's own accounting, not an
estimate.

The cost of that saturation is entirely on the tail: P95 grew **2.52x** (21s
-> 53s) for a throughput gain of 0.98x. Every millisecond of that extra P95 is
queue time behind the same 4 busy slots, not additional compute - the compute
cost per token did not change, only how long a request waits for a slot to
open up.

If I had to pick one knob to raise goodput@SLO (say a 15s P95 target, which
both runs already blow through), it would be **`--parallel`**, i.e. more
concurrent decode slots, not context size or quantization. The bottleneck
measured here is slot occupancy specifically (`n_busy_slots_per_decode`
pinned near its ceiling with a nonzero deferred queue), which is exactly what
`--parallel` controls. Raising `--ctx-size` would only make each slot's KV
cache larger without adding slots, and switching quantization would change
per-token compute cost, not the number of requests that can be in flight at
once - neither addresses what these metrics say is actually full. The tradeoff
is that more parallel slots shrink the *per-slot* KV cache budget (context
window is shared across `--parallel` slots) and, on hardware this small (2GB
VRAM GPU, 0.8B model already near the ceiling), more concurrent decode steps
per GPU pass may themselves become the new compute bottleneck - so I would
re-run `make load-50` at `--parallel 8` and re-check the slot gauge before
assuming the win is free.
