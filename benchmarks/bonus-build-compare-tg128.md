# Bonus B1 - Prebuilt vs source build

Host `Linux-x86_64` · CPU `11th Gen Intel(R) Core(TM) i7-1165G7 @ 2.80GHz`
Vector extensions detected: AVX-512, AVX2
llama.cpp `b10488` both sides · `threads=4` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `tg128`, 3 repetitions

| Binary | Built for | tg128 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 31.2 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 29.5 | 0.94x |

On this machine, the prebuilt binary is **1.06x faster**.

before: 31.2 tok/s (prebuilt release)
after:  29.5 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 0.94x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.



## Your explanation

Kết quả thực tế trên CPU Intel Core i7-1165G7: Bản **prebuilt release đạt 31.2 tok/s**, nhanh hơn **1.06× (0.94× relative)** so với bản tự build `-DGGML_NATIVE=ON` (29.5 tok/s).

**Cơ chế giải thích cho kết quả bất ngờ này:**
1. **Memory-Bandwidth Bound Decode:** Tác vụ sinh token (tg128) bị thắt nút cổ chai chủ yếu bởi băng thông RAM DDR4 chứ không phải số lượng phép tính số học (ALU instructions). Vì vậy, việc compiler tự động vector hóa rộng hơn không thể vượt qua giới hạn vật lý của memory bus.
2. **AVX-512 Thermal Throttling / Downclocking:** i7-1165G7 là vi xử lý di động tiết kiệm điện (TDP 12W–28W). Khi cờ `-DGGML_NATIVE=ON` bật toàn bộ tập lệnh AVX-512 F/CD/BW/DQ/VL/VNNI, các đơn vị thực thi 512-bit vector tiêu thụ điện năng và tỏa nhiệt rất lớn, kích hoạt cơ chế hạ xung nhịp tự động (AVX-512 frequency offset downclocking) của CPU để giữ an toàn nhiệt, làm giảm nhẹ xung nhịp sustained clock của các nhân.
3. **Runtime Dispatch vs Compiler Auto-vectorization:** Bản prebuilt release của upstream llama.cpp sử dụng cơ chế dynamic dispatch runtime tới các hand-tuned assembly microkernels (tối ưu hóa thủ công cho AVX2/AVX-512) mà không làm ép xung nhiệt toàn bộ binary, mang lại hiệu suất thực tế cao hơn và ổn định hơn.
