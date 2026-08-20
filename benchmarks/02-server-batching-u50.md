# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.82 of 4 slots (96%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 3045 |

Highest sampled value was **3.82 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width here is **3.82 of 4 slots**, but effective concurrency from
`02-server-results.md` (Little's Law: RPS x avg latency) at 50 users is
**13.1**. These disagree by more than 3x, and that is expected, not a
contradiction, because the two numbers measure different things:
`n_busy_slots_per_decode` is bounded above by `--parallel` (there are only 4
slots, so it can never exceed 4 - 3.82/4 means the server is running with
essentially no idle slot capacity, i.e. genuinely saturated). Effective
concurrency via Little's Law has no such ceiling - it counts every request
"in the system", including ones sitting in the deferred queue waiting for a
slot to free up, not just the ones actively occupying a decode slot right now.

I trust `n_busy_slots_per_decode` for "is the server saturated" (yes - 3.82/4
is about as pinned as 4 discrete slots can read on an average gauge) and I
trust the Little's-Law effective-concurrency number for "how much backlog is
queued behind that saturation" (13.1 against 4 slots means roughly 3x more
requests were queued/in-flight than could be actively served at once). Read
together rather than against each other: the slot gauge says the *server* is
full, and `requests_deferred` sitting at 33-46 for the whole 60s window plus
effective concurrency at 3.28x the slot count says how large the backlog
behind that full server actually is. That backlog, not compute, is exactly
what turned into the 1.89x P95 growth in `02-server-results.md`.
