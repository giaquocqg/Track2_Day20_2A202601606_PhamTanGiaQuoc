# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 57.4 | 99% |
| 4 | 57.0 | 98% |
| 8 | 56.9 | 98% |
| 16 | 56.4 | 97% |
| 32 | 58.2 | 100% |

**Best**: `-t 32` at 58.2 tok/s
**Slowest tested**: `-t 16` at 56.4 tok/s (1.03x spread)
**Against the physical-core default** (`-t 8`, 56.9 tok/s): 1.02x

Use this in your run:

```bash
LAB_N_THREADS=32 make bench
```

## Your explanation

**Curve gần như flat** (57-58 tok/s across all thread counts). Peak tại 32 threads nhưng gain chỉ 1.01× so với 1 thread. Nguyên nhân: máy này dùng Vulkan backend với RTX 3050 GPU, inference chạy trên GPU chứ không phải CPU. CPU threads chỉ lo coordination overhead, không phải compute bottleneck. Kết luận: trên GPU-accelerated inference, thread count ít ảnh hưởng — GPU compute才是bottleneck.
