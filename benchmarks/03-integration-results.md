# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 7681.6 | 7681.7 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 7360.9 | 7361.0 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 7182.8 | 7183.0 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **7408.4** · total **7408.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- N16 Cloud/IaC: stub — this run is localhost only.
- N17 Data pipeline: stub — documents are an in-memory list.
- N18 Lakehouse: stub — no Delta/Iceberg table is queried.
- N19 Vector + features: stub — retrieval uses keyword overlap, not a vector index.
- N20 Serving: real — all three answers come from the local llama-server endpoint.

The LLM accounts for essentially 100% of the 7.41-second mean, which is expected for
an in-memory toy retriever with no embedding call. To halve end-to-end latency I would
target LLM inference first: preserve the six-thread optimum, shorten generated output,
and test a faster runtime/backend. Optimising the 0.1 ms retrieval stage cannot produce
a meaningful end-to-end improvement.
