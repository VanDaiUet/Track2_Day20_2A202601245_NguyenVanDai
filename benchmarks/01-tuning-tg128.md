# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 15.1 | 61% |
| 2 | 24.5 | 99% |
| 4 | 24.9 | 100% |
| 8 | 10.3 | 41% |
| 16 | 3.4 | 14% |

**Best**: `-t 4` at 24.9 tok/s
**Slowest tested**: `-t 16` at 3.4 tok/s (7.38x spread)
**Against the physical-core default** (`-t 4`, 24.9 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

Đỉnh hiệu năng (knee) đạt tại **-t 4** (24.9 tok/s), trùng khớp chính xác với số **physical cores (4 nhân)** của CPU i7-1165G7. 

**Cơ chế giải thích:**
1. **Từ 1 -> 2 và 4 threads:** Tốc độ tăng vọt từ 15.1 lên 24.5 tok/s (+62%) khi tận dụng thêm nhân thực và các kênh bộ nhớ khả dụng. Tăng từ 2 lên 4 threads chỉ tăng nhẹ từ 24.5 lên 24.9 tok/s cho thấy băng thông bộ nhớ RAM DDR4 của hệ thống đã bắt đầu chạm ngưỡng bão hòa ngay từ 2–4 luồng.
2. **Từ 4 -> 8 threads (Logical Cores / Hyperthreading):** Hiệu năng sụt giảm nghiêm trọng từ 24.9 xuống 10.3 tok/s (-59%). Trong tác vụ sinh token (autoregressive decode), quá trình đọc trọng số là memory-bandwidth bound. Hyperthreading không bổ sung thêm memory bus hay bộ đệm L2 mà buộc 2 luồng phải tranh chấp cùng pipeline và cache của 1 nhân vật lý, gây ra cache thrashing.
3. **Tại 16 threads:** Hiệu năng sụp đổ hoàn toàn xuống 3.4 tok/s (chậm hơn 7.32× so với -t 4). Hiện tượng thread oversubscription trong môi trường ảo hóa WSL2 tạo ra overhead chuyển ngữ cảnh (context switching) và lock contention khổng lồ giữa các worker threads của llama.cpp.

**Kết luận:** Đặt `LAB_N_THREADS=4` là cấu hình tối ưu tuyệt đối cho máy này.
