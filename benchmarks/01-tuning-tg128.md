# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 5.5 | 29% |
| 3 | 14.2 | 76% |
| 6 | 18.6 | 100% |
| 12 | 14.9 | 80% |
| 24 | 12.3 | 66% |

**Best**: `-t 6` at 18.6 tok/s
**Slowest tested**: `-t 1` at 5.5 tok/s (3.40x spread)
**Against the physical-core default** (`-t 6`, 18.6 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## My explanation

The knee is exactly six threads, matching the six physical cores. Throughput rises
from 5.5 tok/s at one thread to 18.6 tok/s at six, then falls to 14.9 tok/s at the
12 logical threads and 12.3 tok/s at 24 threads. Changing `-t 12` to `-t 6` therefore
improves decode throughput by `18.6 / 14.9 = 1.25x`.

Decode repeatedly streams the model weights and is constrained by shared memory
bandwidth and cache capacity. SMT threads do not add memory channels or execution
cores; above six they compete for the same bandwidth, cache, and scheduling slots.
At 24 threads oversubscription adds still more context-switch and coordination cost,
which explains why throughput drops rather than flattening.
