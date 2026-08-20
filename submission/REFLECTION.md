# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Văn Đại
**Cohort:** A20-K3
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Ubuntu 24.04 on WSL2 (Linux 5.15.167.4-microsoft-standard-WSL2 x86_64)
- **CPU:** 11th Gen Intel(R) Core(TM) i7-1165G7 @ 2.80GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX-512, AVX2
- **RAM:** 7.8 GB
- **Accelerator:** NVIDIA GeForce MX350 (2048 MiB) (Prebuilt Vulkan binary chạy CPU-only)
- **llama.cpp asset đã tải:** llama-b10488-bin-ubuntu-vulkan-x64.tar.gz (build b10488)
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare)

**Chạy ở đâu:** laptop của tôi (WSL2 Ubuntu)

**Setup story** (≤ 80 chữ): Máy có 7.8 GB RAM (< 8.0 GB) nên chọn model Qwen3.5 0.8B qua LAB_MODEL=qwen35-0.8b. Runtime prebuilt Vulkan nhận GPU nhưng tắt offload do thiếu Linux CUDA binary sẵn, chạy base track hoàn toàn trên CPU. Setup tự động qua make setup hoàn tất trơn tru.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 12045 | 342 / 2659 | 43.4 / 111.7 | 2975 / 8001 / 8001 | 23.1 |
| UD-Q2_K_XL | 0.39 | 69113 | 579 / 1604 | 45.1 / 59.8 | 3398 / 5368 / 5368 | 22.2 |

**Quan sát** (≤ 60 chữ): 2-bit chậm hơn 1.04× (22.2 vs 23.1 tok/s) và TTFT tệ hơn 69%, hoàn toàn không đáng dùng. Do chạy CPU-only, chi phí dequantize vượt qua lợi ích giảm dung lượng. Q4_K_M cho câu trả lời mạch lạc và chính xác hơn hẳn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.51 | 15000 | 23000 | 24000 | 7.6 | 0.0% |
| 50 | 0.63 | 32000 | 56000 | 57000 | 19.2 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.24×
- **P95 tăng:** 2.43×
- **Effective concurrency ở 50 users:** 19.2 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.87 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa từ ≤50 users: RPS chỉ tăng 1.24× khi tải tăng 5×, Eff. concurrency (19.2) vượt 4.79× số slot (4) và deferred đạt 46 reqs. P95 tăng 2.43× hoàn toàn là queue time. Để tăng goodput@SLO, tôi sẽ tăng `--parallel 8` kết hợp `--cache-type-k q8_0` để giải phóng hàng đợi mà không tràn RAM 7.8GB.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | local WSL2 (thay vì k8s cluster) | stub |
| N17 Data pipeline | in-memory list (TOY_DOCS) | stub |
| N18 Lakehouse | in-memory dict | stub |
| N19 Vector + features | keyword overlap fallback | stub |
| N20 Serving | `llama-server` (:8080) | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 4469.5 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): LLM chiếm 100% độ trễ (4.47s), hoàn toàn khớp kỳ vọng vì retrieval là in-memory (<0.1ms). Muốn giảm latency 2×, tôi sẽ tối ưu LLM: áp dụng prompt caching để bỏ qua prefill context và bật GPU offload (CUDA) nhằm tăng tốc decode.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Giảm số lượng luồng `-t` từ 16 (oversubscription) xuống đúng 4 (bằng số physical cores của CPU i7-1165G7).

```
before:  3.4 tok/s (-t 16)
after:   24.9 tok/s (-t 4)
speedup: 7.32×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Giai đoạn decode của mô hình ngôn ngữ (autoregressive token generation) là bài toán đặc thù bị nghẽn bởi băng thông bộ nhớ (memory-bandwidth bound) chứ không phải năng lực tính toán (compute FLOPs). Với mỗi token sinh ra, toàn bộ trọng số mô hình phải được tải tuần tự từ RAM vào các thanh ghi CPU. CPU i7-1165G7 có 4 nhân vật lý và 2 kênh bộ nhớ DDR4. Khi chạy ở mức `-t 4`, 4 luồng xử lý tận dụng tối đa băng thông bộ nhớ độc lập trên từng nhân mà không tạo ra hiện tượng xung đột tài nguyên.

Khi tăng luồng lên `-t 8` (bật Hyperthreading) và đặc biệt là `-t 16` (oversubscription gấp 4 lần số nhân thực), hiệu năng sụp đổ hoàn toàn từ 24.9 xuống 3.4 tok/s (chậm hơn 7.32×). Hyperthreading không tạo thêm kênh bộ nhớ hay mở rộng bus RAM mà buộc 2 luồng logic phải chia sẻ chung bộ đệm L1/L2 và execution units trên cùng 1 nhân vật lý, dẫn đến cache thrashing nặng nề. Trong môi trường ảo hóa WSL2, 16 luồng liên tục tranh chấp khóa (lock contention) và gây quá tải bộ lập lịch hệ điều hành do overhead chuyển ngữ cảnh (context-switch overhead) khổng lồ, biến CPU từ trạng thái xử lý dữ liệu sang trạng thái chờ tranh chấp bus và scheduler.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

**Đã làm:** B1 (Biên dịch llama.cpp từ mã nguồn) & B4 Challenge C7 (Khảo sát tập lệnh Native AVX-512 CPU vs Prebuilt binary)

**Numbers:**

```
before:  31.2 tok/s (prebuilt release)
after:   29.5 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 0.94× (prebuilt nhanh hơn 1.06×)
```

**Điều này nói lên gì mà deck chưa nói:**

Không phải cứ biên dịch mã nguồn với toàn bộ cờ tối ưu phần cứng (`-DGGML_NATIVE=ON`) là sẽ đạt tốc độ cao hơn. Trên các vi xử lý laptop bị giới hạn nhiệt và công suất tiêu thụ (TDP ~28W như i7-1165G7), việc ép trình biên dịch sinh mã vector 512-bit diện rộng sẽ kích hoạt cơ chế hạ xung nhịp tự động (AVX-512 thermal/frequency downclocking) của CPU. Hơn nữa, vì decode bị giới hạn bởi băng thông bộ nhớ (memory-bandwidth bound), số lượng ALU FLOPs bổ sung từ AVX-512 không giúp ích mà còn làm giảm xung nhịp thực thi. Bản prebuilt với cơ chế runtime dispatch tới các assembly microkernels được tinh chỉnh thủ công (hand-tuned) đem lại hiệu năng thực tế cao hơn và mát hơn.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Hai kết quả thực nghiệm bất ngờ nhất là: (1) Lượng tử hóa 2-bit chậm hơn 4-bit trên CPU do chi phí dequantize vượt qua lượng byte tiết kiệm được, và (2) Bản tự build Native AVX-512 chậm hơn Prebuilt do giới hạn nhiệt CPU di động. Điều này chứng minh tối ưu hóa inference luôn phải dựa trên phép đo đạc thực nghiệm gắn liền với giới hạn phần cứng (memory bus & thermal limits) thay vì chỉ nhìn vào lý thuyết số bit hay độ rộng vector.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo GitHub ở chế độ **public**
- [x] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
