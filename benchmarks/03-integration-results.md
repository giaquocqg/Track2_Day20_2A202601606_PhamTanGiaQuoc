# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 30200.5 | 30200.6 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 24446.6 | 24446.7 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 3121.2 | 3121.4 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **19256.1** · total **19256.2**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

**N16 Cloud/IaC:** stub | **N17 Data pipeline:** stub | **N18 Lakehouse:** stub | **N19 Vector+features:** stub | **N20 Serving:** real

**Dominant stage:** llm (100%) — hoàn toàn như kỳ vọng vì chỉ có llama-server là real. Embed/retrieve dùng keyword fallback nên 0ms. Để giảm latency 2×: tấn công llm bằng cách giảm max_tokens hoặc tăng batch size — LLM decode là bottleneck duy nhất.
