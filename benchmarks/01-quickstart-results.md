# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 12045 | 342 / 2659 | 43.4 / 111.7 | 2975 / 8001 / 8001 | 23.1 |
| UD-Q2_K_XL | 0.39 | 69113 | 579 / 1604 | 45.1 / 59.8 | 3398 / 5368 / 5368 | 22.2 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.04x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

UD-Q2_K_XL decode chậm hơn Q4_K_M (22.2 vs 23.1 tok/s, 1.04x slower) dù nhỏ hơn 0.11 GB (22%). Đồng thời TTFT P50 của Q2 cũng cao hơn đáng kể (579 ms vs 342 ms, +69%).

Nguyên nhân: Do chạy CPU-only (ngl=0) trên 4 physical cores, máy rơi vào trạng thái compute-limited đối với khâu giải mã lượng tử hóa phức tạp. Chi phí tính toán dequantization của định dạng 2-bit (Unsloth Dynamic) vượt qua lợi thế băng thông bộ nhớ từ việc giảm kích thước tệp.

Kết luận: Trên cấu hình máy này, UD-Q2_K_XL hoàn toàn không đáng dùng — vừa chậm hơn về TTFT lẫn Decode, vừa suy giảm chất lượng đầu ra mà chỉ tiết kiệm được 110 MB RAM. Q4_K_M là lựa chọn tối ưu vượt trội.
