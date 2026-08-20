# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Đỗ Quang Huy (2A202601896)
**Cohort:** A20-K3
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (build 26200), AMD64
- **CPU:** 11th Gen Intel(R) Core(TM) i7-1165G7 @ 2.80GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** not reported by `make probe` on this llama.cpp build (CPU backend loaded is `ggml-cpu-icelake.dll`, i.e. it auto-selected the Ice Lake code path — implies AVX2/AVX-512 support)
- **RAM:** 7.7 GB
- **Accelerator:** NVIDIA GeForce MX330 (2048 MiB VRAM, CUDA, compute capability 6.1) — Vulkan device also present but CUDA was picked
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cuda-12.4-x64.zip` + `cudart-llama-bin-win-cuda-12.4-x64.zip` (CUDA runtime DLLs)
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare) (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (local, Windows)

**Setup story** (≤ 80 chữ): `make probe` tự chọn Qwen3.5 0.8B vì RAM 7.7GB dưới
ngưỡng 8GB của Gemma 4 mặc định — không cần can thiệp. Riêng `.\lab.ps1` ban đầu
crash với lỗi parser (`'<' operator is reserved`): file chứa dấu em-dash UTF-8
không BOM, Windows PowerShell 5.1 đọc bằng code page hệ thống (cp1252) nên hiểu
sai một byte thành dấu ngoặc kép, làm gãy chuỗi ở dòng sau. Đã sửa bằng cách
thay các dấu em-dash bằng dấu gạch ngang ASCII trong `lab.ps1`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M (primary) | 0.50 | 2717 | 308 / 329 | 34.1 / 41.5 | 2423 / 2942 / 2942 | 29.3 |
| UD-Q2_K_XL (compare) | 0.39 | 3117 | 342 / 408 | 49.4 / 62.2 | 3467 / 4143 / 4143 | 20.3 |

**Quan sát** (≤ 60 chữ): Ngược kỳ vọng — 2-bit **chậm hơn** 4-bit 1.44x (20.3 vs
29.3 tok/s), vì cả hai đều offload GPU (MX330 yếu, 2GB VRAM) nên bị compute-bound,
không phải bandwidth-bound; dequant của Q2_K tốn compute hơn. (Lần đo đầu, cold
cache, chênh còn rõ hơn: 1.88x — cùng kết luận, khác trạng thái cache.) Đã hỏi
cùng câu qua cả hai server: Q4_K_M trả lời gọn (`stop`, 80 token); Q2_K_XL bị lặp và cắt
giữa chừng (`length`, dính cả tag `</think>` rò rỉ). Không đáng dùng trên máy này.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.61 | 15000 | 21000 | 22000 | 8.6 | 0.0% |
| 50 | 0.60 | 25000 | 53000 | 53000 | 16.6 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.98×
- **P95 tăng:** 2.52×
- **Effective concurrency ở 50 users:** 16.6 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.82 / 4 slots

**Saturation reading** (≤ 80 chữ): Server đã bão hoà **từ 10 users**, không phải
đâu đó giữa 10-50 — RPS gần như đứng yên (0.61→0.60) dù offered load gấp 5 lần.
Bằng chứng mạnh nhất: `n_busy_slots_per_decode` giữ ở 3.7-3.82/4 suốt cửa sổ 60s,
và `requests_deferred` = 33-46 liên tục — cả 4 slot bận gần như 100% thời gian,
hàng chờ luôn có. P95 tăng 2.52× trong khi throughput chỉ 0.98× là queue time
thuần, không phải compute (compute/token không đổi). Knob đổi trước: `--parallel`
(tăng số slot) — đúng thứ nghẽn cổ chai đo được, chứ không phải ctx-size hay quant.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | — | stub (localhost only, không k8s/Compose) |
| N17 Data pipeline | — | stub (`TOY_DOCS` list trong process, không Airflow) |
| N18 Lakehouse | — | stub (list Python thuần, không cả SQLite) |
| N19 Vector + features | — | stub (keyword-overlap fallback, `embed_url` không set nên `embed()` trả `None`) |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 8364.8 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Đúng kỳ vọng, thậm chí cực đoan hơn — vì `embed_url`
không set nên embed=0 tuyệt đối (không gọi mạng), keyword-overlap trên 6 doc
toy chỉ tốn 0.1-0.2ms, nên llm gần như là chi phí thật duy nhất. Muốn giảm 2×:
tấn công decode trước (không phải retrieval) — hạ `max_tokens` (query 3 tốn
8.6s decode/11.6s total) và dùng `-t 1` (từ §5, decode ở đây cũng GPU-bound).

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** hạ thread count từ mặc định theo physical-core (`-t 4`) xuống `-t 1`

```
before:  10.0 tok/s   (-t 4, physical-core default)
after:   22.5 tok/s   (-t 1)
speedup: 2.24×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Kết quả **không** khớp deck: đường cong đáng lẽ leo lên đến physical core count
(4) rồi mới chững/giảm, nhưng ở đây `-t 1` đã là đỉnh và throughput giảm đơn
điệu suốt tới `-t 8` (22.5 → 12.3 → 10.0 → 6.6 tok/s), chỉ nhích nhẹ lại ở
`-t 16` (7.8, nằm trong nhiễu vì `--reps 2`). Lý do: `ngl=99` — toàn bộ layer
của model 0.8B này được offload lên GPU (MX330). Nghĩa là phép nhân ma trận
thật sự của decode chạy trên CUDA, không phải trên CPU, nên `-t` ở đây không
còn sweep "số core làm việc bandwidth-bound" như deck giả định cho trường hợp
CPU-only — nó sweep kích thước threadpool phía CPU chỉ để làm sampling, quản
lý KV-cache bookkeeping và dispatch công việc sang hàng đợi GPU giữa các bước
token.

Với chỉ một GPU stream làm việc thật, thêm thread CPU chỉ tạo thêm overhead
điều phối (đánh thức thread, tranh chấp mutex/condvar quanh decode loop dùng
chung, và context-switch trên CPU 4 core/8 luồng cũng đang phải nuôi hàng đợi
GPU) mà không thêm compute hữu ích nào — nên throughput giảm đơn điệu theo số
thread. Nói ngắn gọn về cơ chế: **khi `ngl=99`, decode bị GPU-bound chứ không
còn CPU-bandwidth-bound, nên knob thread-count của CPU không còn là đòn bẩy mà
heuristic "physical core count" của deck giả định.** Thay đổi quan trọng nhất
trên máy này thực ra đã xảy ra ngầm khi bật GPU offload; với điều đó, việc còn
lại là giữ threadpool điều phối gọn nhất có thể — `-t 1`, cho 2.24× so với mặc
định physical-core `-t 4`.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

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
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
