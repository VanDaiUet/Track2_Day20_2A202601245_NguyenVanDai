# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 6065.2 | 6065.2 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 3531.2 | 3531.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 3812.1 | 3812.2 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **4469.5** · total **4469.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it filters out requests that do not meet the specified targets (specifically the Time-to-First-Throughput, or TTFT, and Time-to-First-Throughput-Overhead, or TPOT).

While raw throughput measures the total requests per second, Goodput calculates only the requests that successfully met these targets. This means Go

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing the Key-Value (KV) cache in non-contiguous pages.

By using non-contiguous pages, the model avoids the wasted space that would exist if all KV data were packed into a single contiguous block. This allows the system to utilize more of the GPU's available memory efficiently.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**.

This is because the context explicitly states that prefill is compute-bound (requiring significant processing power) and decode is memory-bound (requiring significant bandwidth). By splitting these operations into separate pools, the system can utilize parallel processing for the compute-heavy part an


## Which N16-N19 pieces are real

| Day | Thành phần | Trạng thái | Ghi chú |
|---|---|---|---|
| N16 | Cloud / IaC | **Stub** | Chạy local WSL2 thay vì Kubernetes cluster / Compose stack phân tán |
| N17 | Data pipeline | **Stub** | Danh sách in-memory `TOY_DOCS` |
| N18 | Lakehouse | **Stub** | In-memory dictionary / list |
| N19 | Vector + features | **Stub** | Fallback keyword overlap (không dùng embedding server bên ngoài) |
| N20 | Serving | **Real** | `llama-server` chạy cục bộ trên port 8080 |

**Nhận xét:**
- Giai đoạn **LLM chiếm 100% thời gian** (4469.5 ms trên tổng 4469.6 ms) hoàn toàn khớp với kỳ vọng thực tế vì bước retrieve từ in-memory keyword chỉ tốn <0.1 ms, trong khi LLM phải chạy cả prefill (110–150 tokens) lẫn autoregressive decode (80–120 tokens) trên CPU.
- **Nếu cần giảm độ trễ 2×:** Tôi sẽ tập trung tối ưu stage **LLM** bằng cách:
  1. Sử dụng **Prompt Caching** (cố định prefix ngữ cảnh để tái sử dụng KV cache, triệt tiêu gần như toàn bộ chi phí prefill).
  2. Bật GPU offload qua CUDA compilation (`make build-llama`) để tăng tốc decode tokens/s.
  3. Giới hạn `max_tokens` ngắn gọn hơn khi sinh câu trả lời RAG.
