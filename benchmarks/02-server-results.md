# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 19 | 0.33 | 22000 | 37000 | 37000 | 7.3 | 0.0% |
| 50 | 33 | 0.57 | 32000 | 53000 | 57000 | 17.8 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.71x** (34% of linear) |
| P95 latency | **1.43x** |
| Effective concurrency at 50 users | 17.8 vs `--parallel 4` slots (occupancy/slot ratio 4.44) |

**Saturated.** Throughput delivered only 1.71x for 5x the offered load, and effective concurrency (17.8) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew 1.43x while throughput grew 1.71x. That ratio alone does not prove headroom:
the separate u50 metrics sample reached 3.90/4 busy slots with 46 deferred requests,
and Little's Law gives 17.8 effective concurrent requests against only four slots.

> **Small sample.** Only 19 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## My reading

The server is decisively saturated by 50 users. Increasing offered users 5x raises
delivered throughput only 1.71x (0.33 to 0.57 RPS), while P95 rises from 37 to 53
seconds. Little's Law gives 17.8 effective concurrent requests against four slots;
an independent u50 metrics sample observed 3.90/4 busy slots and 46 deferred requests.
The excess occupancy is therefore queue time, not extra compute capacity.

For a P95 SLO of 45 seconds, I would first apply admission control or cap concurrency
near the saturation knee rather than add slots. All four decode slots are already
busy, and more simultaneous CPU decode streams would compete for the same memory
bandwidth. This limits offered load but preserves more requests as goodput within SLO.
