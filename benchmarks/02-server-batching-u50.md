# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 24 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.87 of 4 slots (97%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 4303 |

Highest sampled value was **3.87 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Đỉnh `n_busy_slots_per_decode` ghi nhận đạt **3.87 / 4 slots (97% slot utilization)**, chứng minh thuật toán Continuous Batching của llama-server hoạt động tối đa: gom các request đang hoạt động vào chung các bước decode ma trận.

Đồng thời, `requests_processing = 4` liên tục và `requests_deferred` duy trì ở mức 42–46 request trong suốt thời gian load test. Điều này cho thấy hệ thống đã bão hòa tải hoàn toàn và đang xuất hiện hàng đợi chờ slot (queue backlog). Gauge `n_busy_slots_per_decode` đo trực tiếp trạng thái decode nội tại của engine (phản ánh concurrency phần cứng thực tế ≤ 4 slots), trong khi Little's Law tính toán tổng số request đang nằm trong toàn bộ hệ sinh thái (bao gồm cả hàng đợi chờ slot). Cả hai số liệu bổ trợ cho nhau và đều xác thực server đã chạm ngưỡng bão hòa tuyệt đối.
