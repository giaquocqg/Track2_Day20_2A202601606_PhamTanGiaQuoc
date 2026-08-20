# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 7941 | 475 / 593 | 15.4 / 15.7 | 1462 / 1584 / 1584 | 65.0 |
| UD-Q2_K_XL | 2.24 | 7507 | 468 / 496 | 18.7 / 21.3 | 1646 / 1799 / 1799 | 53.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.21x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

**Q2 KHÔNG đáng dùng trên máy này.** Mặc dù `UD-Q2_K_XL` (2.24 GB) nhỏ hơn 0.73 GB so với `UD-Q4_K_XL` (2.97 GB), tốc độ decode của Q2 lại chậm hơn (53.5 tok/s vs 65.0 tok/s, chậm hơn 1.21×). Điều này xảy ra do mô hình hoàn toàn nằm vừa trong 4 GB VRAM của RTX 3050 Laptop GPU, nên chi phí giải nén (dequantization) các block lượng tử hóa 2-bit phức tạp tốn nhiều chu kỳ tính toán hơn là lượng byte tiết kiệm được. Về chất lượng trả lời, `UD-Q4_K_XL` cho câu trả lời mạch lạc, chính xác và không bị suy giảm khả năng suy luận logic so với 2-bit. Kết luận: `UD-Q4_K_XL` là lựa chọn tối ưu vượt trội.
