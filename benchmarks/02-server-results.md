# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 30 | 0.51 | 15000 | 23000 | 24000 | 7.6 | 0.0% |
| 50 | 41 | 0.63 | 32000 | 56000 | 57000 | 19.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.24x** (25% of linear) |
| P95 latency | **2.43x** |
| Effective concurrency at 50 users | 19.2 vs `--parallel 4` slots (occupancy/slot ratio 4.79) |

**Saturated.** Throughput delivered only 1.24x for 5x the offered load, and effective concurrency (19.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.24x while P95 moved 2.43x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Server bão hòa mạnh mẽ ngay dưới ngưỡng 50 users (thực tế đã bắt đầu bão hòa từ 10 users). 

**Bằng chứng thuyết phục:**
1. **Throughput plateau:** Khi offered load tăng gấp 5× (10 lên 50 users), throughput thực tế chỉ tăng **1.24×** (0.51 lên 0.63 RPS).
2. **Effective Concurrency (Little's Law):** Đạt **19.2** tại 50 users, vượt xa số slot vật lý `--parallel 4` (tỷ lệ occupancy/slot = 4.79×). Kết hợp với `requests_deferred = 46`, điều này chứng minh độ trễ P95 tăng vọt **2.43×** (23s lên 56s) hoàn toàn là **queue time** (thời gian nằm chờ trong hàng đợi để có slot trống), chứ không phải compute time.

**Giải pháp nâng cao Goodput@SLO:**
Knob đầu tiên tôi sẽ thay đổi là tăng số slot song song **`--parallel 8`** (hoặc `--parallel 6`) kết hợp với **`--cache-type-k q8_0 --cache-type-v q8_0`** (lượng tử hóa KV cache). Vì model Qwen3.5 0.8B rất nhỏ (0.5 GB), việc lượng tử hóa KV cache cho phép nhét vừa 8 slots vào 7.8 GB RAM mà không gây OOM. Tăng số slots sẽ trực tiếp mở rộng diện mạo phục vụ của scheduler, tiêu tán hàng đợi chờ (`deferred requests`) và giảm triệt để queue time tại P95.
