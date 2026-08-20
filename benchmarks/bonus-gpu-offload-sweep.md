# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `nvidia_cuda, vulkan` ·
llama.cpp `b10488` · `threads=8` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 17.5 | 1.00x | 25% |
| 8 | 22.9 | 1.31x | 32% |
| 16 | 31.2 | 1.78x | 44% |
| 24 | 45.1 | 2.57x | 63% |
| 32 | 55.8 | 3.18x | 78% |
| 99 | 71.6 | 4.08x | 100% |

Best: `-ngl 99` at 71.6 tok/s
-- 4.08x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Full offload (`-ngl 99`) đạt hiệu năng tối ưu nhất trên máy này với **71.6 tok/s**, mang lại mức tăng tốc **4.08x** so với CPU-only (`-ngl 0` đạt 17.5 tok/s). 

Khi tăng số layers offload từ 0 lên 8, 16, 24, 32, và 99 (toàn bộ model nằm trong VRAM 4GB của RTX 3050), tốc độ decode tăng gần như tuyến tính theo tỷ lệ layer được tính toán trực tiếp trên GPU. Model `UD-Q4_K_XL` chiếm khoảng 2.97 GB VRAM, hoàn toàn vừa vặn trong 4096 MiB VRAM của RTX 3050 Laptop GPU, do đó không xảy ra hiện tượng tràn VRAM (OOM) hay PCIe thrashing giữa Host và Device. Memory bandwidth của GDDR6 GPU (~192 GB/s) vượt trội so với DDR5 CPU (~38 GB/s), giải phóng hoàn toàn điểm nghẽn decode.
