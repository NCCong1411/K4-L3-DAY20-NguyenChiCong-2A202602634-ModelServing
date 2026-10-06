# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** _Nguyễn Chí Công_
**MSSV:** _2A202602634_
**Cohort:** _A20-K4_
**Ngày submit:** _2026-10-06_

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** _Windows (NT release 10), AMD64_
- **CPU:** _AMD Ryzen 5 7535HS with Radeon Graphics_
- **Cores:** _6 physical / 12 logical_
- **CPU extensions:** _AVX2_
- **RAM:** _15.2 GB_
- **Accelerator:** _NVIDIA GeForce RTX 2050 4096 MiB; Vulkan present; inference run on CPU_
- **llama.cpp asset đã tải:** _llama-b10488-bin-win-cpu-x64.zip_
- **Model đã dùng:** _Gemma 4 E2B_ (`LAB_MODEL=`_`gemma4-e2b`_)
- **Quantization:** _UD-Q4_K_XL_ + _UD-Q2_K_XL_ (từ `models/active.json`)

**Chạy ở đâu:** _laptop của tôi_
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Setup tự chọn CUDA, nhưng file CUDA runtime DLL tải về bị hỏng ZIP dù archive chính đã
extract. Tôi chuyển sang CPU prebuilt cùng build `b10488` để giữ lab ổn định và khai
báo `ngl=0`. PowerShell 5.1 cũng đọc sai UTF-8 không BOM, nên runner được đổi chuỗi help
sang ASCII và ép Python UTF-8. Sau đó cả hai model quant tải và benchmark thành công.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 7167 | 1800 / 2018 | 56.7 / 58.5 | 5396 / 5587 / 5587 | 17.6 |
| UD-Q2_K_XL | 2.24 | 6765 | 2143 / 2245 | 59.0 / 60.7 | 5824 / 5957 / 5957 | 17.0 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 nhỏ hơn 24.6% và load nhanh hơn 5.6%, nhưng decode chậm hơn 3.4%, TTFT P50 tăng
19.1% và E2E P50 tăng 7.9%. Với năm prompt giống nhau, Q2 còn sai sort và structured
extraction rõ hơn Q4. Vì vậy Q2 không đáng dùng trên cấu hình CPU này; tôi chọn Q4.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.33 | 22000 | 37000 | 37000 | 7.3 | 0.0% |
| 50 | 0.57 | 32000 | 53000 | 57000 | 17.8 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** _1.71×_
- **P95 tăng:** _1.43×_
- **Effective concurrency ở 50 users:** _17.8_ so với `--parallel` = _4_ slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): _3.90_ / _4_ slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hoà rõ ở 50 users: tải tăng 5× nhưng RPS chỉ tăng 1.71×, P95 lên 53 s;
mẫu metrics u50 đạt 3.90/4 busy slots và 46 request bị deferred. Effective concurrency
17.8 vượt bốn slot nên phần tăng thêm là queue time. Với SLO P95 45 s,
tôi ưu tiên admission control/cap concurrency; thêm slot chỉ làm các decode stream
tranh cùng memory bandwidth.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost only — stub |
| N17 Data pipeline | in-memory document list — stub |
| N18 Lakehouse | no Delta/Iceberg table — stub |
| N19 Vector + features | keyword-overlap retrieval — stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: _0.0 ms_
- retrieve: _0.1 ms_
- llm: _7408.4 ms_
- **stage chiếm nhiều nhất:** _llm_ (_100%_ của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM là bottleneck đúng như kỳ vọng vì embed được bỏ qua và retrieval chỉ là phép so
khớp trong bộ nhớ. Muốn giảm latency 2×, tôi sẽ tối ưu inference: giữ sáu thread vật
lý, giảm output token và thử runtime/backend nhanh hơn; tối ưu retrieval 0.1 ms gần
như không ảnh hưởng total.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** _giảm `-t` từ 12 logical threads xuống 6 physical threads_

```
before:  14.9 tok/s (`-t 12`)
after:   18.6 tok/s (`-t 6`)
speedup: 1.25×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Điểm knee nằm đúng ở sáu physical cores. Từ một đến sáu thread, decode tăng từ 5.5
lên 18.6 tok/s vì có thêm execution capacity. Nhưng 12 SMT threads chỉ đạt 14.9 tok/s
và 24 threads còn 12.3 tok/s.

Decode phải đọc lại trọng số cho mỗi token và bị giới hạn bởi memory bandwidth/cache.
SMT không tạo thêm memory channel hay physical core; các thread trên sáu tranh cùng
bandwidth, cache và scheduling slots. Oversubscription ở 24 threads còn thêm context
switch/coordination cost, nên nhiều thread hơn làm chậm thay vì tăng throughput.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _B1 build-from-source và compare-builds_

**Numbers:**

```
before:  18.2 tok/s (prebuilt release)
after:   17.3 tok/s (source build, `-DGGML_NATIVE=ON`)
speedup: 0.95× (native chậm hơn 5%; prebuilt nhanh hơn 1.05×)
```

**Điều này nói lên gì mà deck chưa nói:**

CMake xác nhận native build dùng AVX2/FMA/F16C, nhưng prebuilt có runtime CPU dispatch
nên cũng có thể chọn kernel AVX2 phù hợp. Decode `tg128` chủ yếu bị giới hạn bởi memory
bandwidth/cache thay vì thiếu vector instructions; vì thế native codegen không bảo đảm
nhanh hơn. Trên máy này, kết quả cho thấy giữ prebuilt vừa đơn giản vừa nhanh hơn khoảng
5%, thay vì giả định compile native luôn tạo speedup.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

Q2 nhỏ hơn và load nhanh hơn nhưng decode lại chậm hơn Q4; tương tự, native build có
AVX2 vẫn thua prebuilt. Cả hai kết quả nhắc tôi phải đo trên workload thật thay vì suy
ra hiệu năng chỉ từ số bit hoặc compiler flag.

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Tôi dùng OpenAI Codex để đọc hướng dẫn, chẩn đoán lỗi PowerShell/UTF-8 và runtime ZIP,
điều phối các lệnh benchmark, và hỗ trợ cấu trúc báo cáo từ số liệu do chính máy tôi
sinh ra. Tôi đã kiểm tra lại số liệu, kết quả model và các giải thích trước khi nộp.
