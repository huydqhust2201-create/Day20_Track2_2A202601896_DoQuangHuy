# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 22.5 | 100% |
| 2 | 12.3 | 55% |
| 4 | 10.0 | 45% |
| 8 | 6.6 | 29% |
| 16 | 7.8 | 35% |

**Best**: `-t 1` at 22.5 tok/s
**Slowest tested**: `-t 8` at 6.6 tok/s (3.41x spread)
**Against the physical-core default** (`-t 4`, 10.0 tok/s): 2.24x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Your explanation

This does **not** match the expected shape (climb to physical-core count, then
flatten/drop). Here the curve is best at `-t 1` and gets monotonically *worse*
all the way to `-t 8`, with only a small, noisy uptick at `-t 16`. The reason
is `ngl=99`: every layer of this 0.8B model is offloaded to the GPU (NVIDIA
MX330). Decode's actual matmuls run on CUDA, not on the CPU, so `-t` here is
not sweeping "cores doing the memory-bandwidth-bound work" the way it would on
a CPU-only run - it is sweeping the size of the CPU-side thread pool that
llama.cpp spins up to handle sampling, KV-cache bookkeeping, and dispatching
work to the GPU queue between token steps.

With only one active GPU stream doing the real work, extra CPU threads add
pure coordination overhead - thread wake-ups, mutex/condvar contention around
the shared decode loop, and OS scheduler churn on a 4-core/8-thread laptop CPU
that is also feeding a GPU queue - without adding any compute the GPU decode
step could use. So more threads = more overhead for zero extra useful work,
and throughput falls monotonically. The `-t 16` uptick versus `-t 8` is inside
noise (rep count is small); it does not represent a second regime.

The mechanism, stated plainly: **when `ngl=99`, decode is GPU-bound, not
CPU-bandwidth-bound, so the CPU thread-count knob stops being the lever the
deck's "physical core count" heuristic assumes.** The single change that
mattered most on this machine was not "pick the right thread count" in the
CPU sense - it was already implicitly made by GPU offload itself. Given that,
the correct thread setting is the smallest one that keeps the coordination
thread pool lean: `-t 1`, a 2.24x win over the naive physical-core default of
`-t 4`.
