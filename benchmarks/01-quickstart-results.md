# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=6` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 7167 | 1800 / 2018 | 56.7 / 58.5 | 5396 / 5587 / 5587 | 17.6 |
| UD-Q2_K_XL | 2.24 | 6765 | 2143 / 2245 | 59.0 / 60.7 | 5824 / 5957 / 5957 | 17.0 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.04x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## My observation

Q2 is 0.73 GB (24.6%) smaller and loads 5.6% faster, but on this CPU-only run it
decodes 3.4% slower, raises median TPOT by 4.1%, median TTFT by 19.1%, and median E2E
latency by 7.9%. The reduced memory traffic did not repay the heavier dequantization
cost at `ngl=0`.

The separate five-prompt quality check also favours Q4. Q4 got arithmetic and sorting
right and preserved the invoice schema, while Q2 mis-sorted the array and failed the
structured extraction. I would deploy Q4 here: Q2 saves disk and a little load time,
but is slower during inference and less reliable on these prompts.
