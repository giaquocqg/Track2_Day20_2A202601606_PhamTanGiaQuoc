# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 62 | 1.07 | 5100 | 24000 | 30000 | 8.7 | 0.0% |
| 50 | 87 | 1.48 | 28000 | 32000 | 34000 | 35.3 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.39x** (28% of linear) |
| P95 latency | **1.33x** |
| Effective concurrency at 50 users | 35.3 vs `--parallel 4` slots (occupancy/slot ratio 8.84) |

**Saturated.** Throughput delivered only 1.39x for 5x the offered load, and effective concurrency (35.3) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (1.33x vs 1.39x), so this server still has headroom at 50 users.

## Your reading

**Server saturated ở ~50 users.** Evidence: (1) Throughput chỉ tăng 1.39× cho 5× load, (2) Effective concurrency 35.3 >> 4 slots (ratio 8.84×), (3) Peak busy_slots = 3.93/4. Để tăng goodput@SLO: tăng `--parallel` từ 4 lên 8+ trước. Knob này vì bottleneck hiện tại là số decode slots, không phải compute — thêm slots cho phép nhiều request xử lý đồng thời.
