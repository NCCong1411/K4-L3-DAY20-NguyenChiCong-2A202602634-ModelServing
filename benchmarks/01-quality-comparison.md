# Q4 vs Q2 quality check

Host: Windows AMD64, Ryzen 5 7535HS, six physical cores, CPU inference (`ngl=0`).
Both servers used the same Gemma 4 E2B model family, five prompts, `temperature=0`,
and `max_tokens=128`.

| Test | Expected | UD-Q4_K_XL | UD-Q2_K_XL |
|:--|:--|:--|:--|
| `17 * 23` | `391` | Correct | Correct |
| Sort `[5,1,4,2,3]` | `[1,2,3,4,5]` | Correct values, wrapped in a JSON code fence | Incorrect: `[5,1,2,3,4]` |
| Invoice extraction | `{"customer":"Nguyen","total_vnd":95000}` | Correct schema but arithmetic result `85000` | Incorrect type and no computed total |
| TTFT vs TPOT | First-token latency vs per-output-token latency | Two bullets, but expanded both acronyms incorrectly | Two bullets, both definitions incorrect |
| 8 requests, 4 slots, 2 s each | `4` | Reasoning started but violated output-only instruction and hit token limit | Reasoning started but violated output-only instruction and hit token limit |

The invoice expected value is 95,000 VND: `3 * 25,000 + 2 * 10,000`. Neither quant
was perfect, but Q4 preserved substantially more task structure. Q2 introduced clear
failures on sorting and structured extraction while also benchmarking slower on this
CPU runtime.
