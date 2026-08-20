# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Phạm Tân Gia Quốc
**Cohort:** A20-K1
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 Home Single Language
- **CPU:** AMD Ryzen 7 6800H with Radeon Graphics
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 15.2 GB
- **Accelerator:** Vulkan (NVIDIA GeForce RTX 3050 Laptop GPU, 4GB VRAM)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): Lab chạy trực tiếp trên laptop sau khi chạy bootstrap.ps1. Không có bước nào fail. llama-bench gặp vấn đề với đường dẫn Unicode có tiếng Việt nên đã copy model ra thư mục ngắn hơn (C:/Users/Admin/llama_lab).

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 19190 | 512 / 1789 | 22.7 / 23.0 | 1942 / 3176 / 3176 | 44.1 |
| UD-Q2_K_XL | 2.24 | 9797 | 586 / 2661 | 39.6 / 51.8 | 2813 / 5459 / 5459 | 25.2 |

**Quan sát** (≤ 60 chữ): Q2 decode **1.75x chậm hơn** Q4 (25.2 vs 44.1 tok/s), dù nhỏ hơn 0.73 GB. Điều này cho thấy decode bị giới hạn bởi dequantization overhead chứ không phải memory bandwidth. Trên GPU Vulkan, việc giải nén nhiều hơn từ Q2 không bù được byte tiết kiệm được. **Không đáng dùng Q2** trên máy này.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.07 | 5100 | 24000 | 30000 | 8.7 | 0.0% |
| 50 | 1.48 | 28000 | 32000 | 34000 | 35.3 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** **1.39×** (28% of linear)
- **P95 tăng:** **1.33×**
- **Effective concurrency ở 50 users:** **35.3** so với `--parallel` = **4** slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): **3.93** / **4** slots

**Saturation reading** (≤ 80 chữ): Server bão hoà rõ ràng: throughput chỉ tăng 1.39× cho 5× load, và effective concurrency (35.3) cao hơn nhiều so với 4 slots. Peak busy_slots = 3.93/4 chứng minh continuous batching đang hoạt động — các request xen kẽ nhau trong 4 decode slots. Để tăng goodput, tôi sẽ tăng `--parallel` lên 8 trước vì bottleneck là số slot, không phải compute.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | stub | stub |
| N17 Data pipeline | stub | stub |
| N18 Lakehouse | stub | stub |
| N19 Vector + features | stub | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: **0.0 ms** (keyword fallback, không có embedding model)
- retrieve: **0.0 ms** (keyword fallback đồng bộ)
- llm: **19256.1 ms** (100% of total)
- **stage chiếm nhiều nhất:** **llm** (**100%** của total)

**Reflection** (≤ 60 chữ): LLM hoàn toàn dominate (100%). Embed/retrieve dùng stub nên 0ms. Nếu cần giảm latency 2×, tôi sẽ giảm max_tokens hoặc tăng batch size — LLM decode là bottleneck duy nhất.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Tăng threads từ 1 lên 32

```
before:  58.2 tok/s (1 thread)
after:   58.7 tok/s (32 threads)
speedup: 1.01×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Trên máy này dùng Vulkan backend với RTX 3050, inference chạy chủ yếu trên GPU chứ không phải CPU. Vì vậy số lượng CPU threads ảnh hưởng rất ít đến throughput — GPU compute units là bottleneck, không phải CPU scheduling.

Gain chỉ 1.01× cho thấy: (1) Vulkan compute đã parallelize tốt với 1 thread, (2) CPU overhead cho thread coordination không đáng kể trên GPU workload. Điều này ngược với expectation từ deck về CPU-bound decode — trên máy có GPU mạnh, **GPU offload mới là bottleneck**.

Để cải thiện thực sự, cần tăng `--parallel` slots (từ 4 lên 8+) hoặc dùng `--gpu-offload` đúng cách với CUDA thay vì Vulkan.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

_(để trống vì không làm bonus track)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

Q2 chậm hơn Q4 về decode speed (1.75×) là ngược intuition thông thường. Điều này xảy ra vì trên Vulkan/GPU, dequantization overhead của Q2 nhiều hơn bytes tiết kiệm được — lesson: quantization không phải lúc nào cũng nhanh hơn.

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
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
