# Bonus - Context-length sweep (prefill cost)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=4` `ngl=0` · RAM 7.8 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 181.3 | 1411.6 | 1.00x |
| 1024 | 158.5 | 6461.4 | 1.14x |
| 2048 | 121.2 | 16892.1 | 1.50x |
| 4096 | 124.1 | 32997.7 | 1.46x |

At 4096 tokens, prefill costs **32998 ms** --
1.46x what linear scaling from the smallest point would predict. That excess
is attention's O(N^2) term becoming visible, and every millisecond of it lands in TTFT
before the user sees a single token.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Your finding

Chi phí Prefill (TTFT) bùng nổ phi tuyến tính bắt đầu rõ nét từ ngưỡng **1024 đến 2048 tokens**, nơi tốc độ prefill giảm từ 181.3 xuống 121.2 tok/s và độ trễ tăng vọt lên **1.50× so với tỷ lệ tuyến tính** (16.9 giây cho 2048 tokens và gần 33 giây cho 4096 tokens).

**Ý nghĩa thiết kế cho RAG Pipeline:**
1. **Prefill Bottleneck:** Trong khi người dùng thường lo ngại về tốc độ decode, thực tế trên CPU cho thấy việc nhồi nhét quá nhiều tài liệu ngữ cảnh vào prompt (ví dụ 4-6 chunks văn bản dài) sẽ phá hủy hoàn toàn trải nghiệm người dùng vì họ phải chờ 15–33 giây TTFT trước khi token đầu tiên xuất hiện.
2. **Context Budget:** Với hệ thống phục vụ trên CPU, ngân sách ngữ cảnh cho RAG pipeline bắt buộc phải giới hạn ở mức **≤ 512–1024 tokens** (tương đương 2–3 chunks ngắn chất lượng cao), kết hợp với Reranker sắc bén thay vì nhồi nhét thô thiển toàn bộ context window.
3. **Giải pháp kiến trúc:** Cần triển khai **Prompt Caching** (cố định tiền tố hệ thống) hoặc tách cụm tính toán Disaggregated Prefill/Decode để cô lập chi phí $O(N^2)$ prefill khỏi các luồng decode tương tác.
