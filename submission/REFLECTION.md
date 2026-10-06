# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.

**Họ Tên:** Nguyễn Vũ Huy
**MSSV:** 2A202602662
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe` (`hardware.json`, screenshot `01-hardware-probe.png`).

- **OS:** Windows 11 Home (10.0.26200), AMD64
- **CPU:** Intel Core Ultra 7 155H (Meteor Lake — 6 P-core + 8 E-core + 2 LP E-core)
- **Cores:** 16 physical / 22 logical
- **CPU extensions:** AVX2, AVX-VNNI (không có AVX-512)
- **RAM:** 31.4 GB (LPDDR5x, dùng chung với iGPU)
- **Accelerator:** Intel Arc Graphics (iGPU) qua Vulkan — `llama-server --list-devices` thấy `Vulkan0: Intel(R) Arc(TM) Graphics (18311 MiB)`
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare)

**Chạy ở đâu:** laptop của tôi (không dùng Colab/Kaggle).

**Setup story:** `.\lab.ps1` không chạy được trên Windows PowerShell 5.1: file là UTF-8
không BOM, nên dấu "—" làm vỡ parser. Vì vậy tôi gọi thẳng các lệnh Python mà `lab.ps1`
map tới (`.venv\Scripts\python labs\...`, `python -m locust ...`), với `PYTHONUTF8=1`. Vulkan
build tự offload toàn bộ lên Arc (`ngl=99`). Riêng `make tune` tôi chạy với
`LAB_N_GPU_LAYERS=0` để đo đúng tác động của số thread CPU.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Từ `benchmarks/01-quickstart-results.md` (`threads=16`, `ngl=99`, `ctx=2048`, `max_tokens=64`).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 15741 | 455 / 2608 | 40.9 / 41.4 | 3034 / 5175 / 5175 | 24.4 |
| UD-Q2_K_XL | 2.24 | 9721 | 1199 / 13794 | 451.2 / 457.9 | 29641 / 37585 / 37585 | 2.2 |

**Quan sát:** 2-bit **không** nhanh hơn mà **chậm hơn 11×** trên iGPU (2.2 so với 24.4
tok/s), dù nhỏ hơn 0.73 GB. `llama-bench` cho thấy trên CPU hai bản ngang nhau (16.8 so với
16.5 tok/s). Vậy lỗi nằm ở kernel Vulkan Q2_K trên Arc, không phải ở số bit. Tôi hỏi cùng các
câu (cộng giờ, số nguyên tố, giải thích bandwidth) trên cả hai bản: đáp án giống nhau, bản Q2
dài dòng hơn. **Không đáng dùng.**

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`--parallel 4`, `ngl=99`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.55 | 12000 | 33000 | 34000 | 8.6 | 0 (0.0%) |
| 50 | 0.68 | 34000 | 56000 | 58000 | 21.4 | 0 (0.0%) |

*(Số request lấy từ `locust-*_stats.csv`: 32 và 40. Bảng summary trên màn hình locust ghi 33
và 41 vì có 1 request hoàn tất đúng lúc locust shutdown, sau khi CSV đã được ghi.)*

- **Offered load tăng 5×, throughput thực tăng:** 1.23×
- **P95 tăng:** 1.70×
- **Effective concurrency ở 50 users:** 21.4 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.93 / 4 slots (kèm `requests_deferred` = 46)

**Saturation reading:** server đã bão hòa từ **10 users**: effective concurrency 8.6 so với
4 slot. Lên 50 users RPS chỉ tăng 1.23×, còn P50 tăng gần 3× (12 → 34 s). Phần tăng thêm là
**queue time**: thời gian phục vụ không đổi (request ngắn nhất 5.6 s rồi 6.3 s, ≈ 48 step ×
~92 ms), và `requests_deferred` ≈ 46 cho thấy hàng đợi trực tiếp. Muốn nâng goodput@SLO,
tôi đổi `--parallel 8` cùng `--ctx-size 4096` **trước**. Lý do: 4 sequence/step chỉ làm step
chậm 2.2×, nên batching còn dư địa.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline` (`benchmarks/03-integration-results.md`).

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | không có, mọi thứ chạy local trên laptop | stub |
| N17 Data pipeline | 6 `TOY_DOCS` hard-code trong `pipeline.py` | stub |
| N18 Lakehouse | không có tầng lưu trữ, docs nằm trong memory | stub |
| N19 Vector + features | `retrieve()` keyword-overlap fallback, không có embedding/vector index | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 3813.2 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection:** LLM chiếm 100% là đúng kỳ vọng. Điều bất ngờ là khoảng **2 s mỗi query**
không nằm trong model: `localhost` trên Windows thử `::1` trước, còn server chỉ bind
127.0.0.1 (`/health`: 2067 ms qua localhost, 3 ms qua 127.0.0.1). Chạy lại với
`--base-url http://127.0.0.1:8080`, llm còn **1340 ms**. Muốn giảm 2×: sửa host / keep-alive
trước, sau đó prefix caching và giảm `max_tokens`.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> Nguồn: `benchmarks/01-tuning-tg128.md` (`make tune` với `LAB_N_GPU_LAYERS=0`, decode tg128 trên CPU).

**Change:** hạ `-t` từ 16 (mặc định = số core vật lý) xuống 8 cho decode trên CPU

```
before:  13.5 tok/s   (-t 16, ngl=0)
after:   15.8 tok/s   (-t 8,  ngl=0)
speedup: 1.17×
```

**Tại sao nó work:**

Decode là **memory-bandwidth-bound**: mỗi token phải đọc lại toàn bộ trọng số đang active
(~1.3 GB ở Q4) từ DRAM, trong khi phép tính trên mỗi byte rất ít. Vì vậy số thread chỉ
giúp đến khi các core đã kéo hết băng thông mà chúng có thể kéo. Đường cong của tôi cho thấy
đúng như vậy: 1 thread 5.6 tok/s, 4 thread 14.7 (≈ 90% đỉnh), 8–12 thread đi ngang quanh
16.5. Sau điểm đó, thêm thread không còn việc để làm, chỉ thêm việc chờ.

Điều **khác** với deck là đỉnh **không** nằm ở 16 core vật lý, và 16 thread còn **chậm hơn**
8. Nguyên nhân là CPU lai của Meteor Lake: 16 "core" gồm 6 P-core, 8 E-core và 2 LP E-core
nằm ở SoC tile. llama.cpp chia mỗi mat-mul **đều** cho `-t` thread và có barrier sau mỗi op,
nên mỗi step đi theo tốc độ của thread **chậm nhất**. Với `-t 16`, một số thread rơi vào
E-core/LP E-core, chậm hơn và xa L3 của P-core, thành straggler. Thêm việc Windows scheduler
di chuyển thread, kết quả dao động rất mạnh: lần chạy lại cho 13.46 ± 11.64 tok/s ở 16
thread, so với 16.47 ± 0.92 ở 8 thread. Lên 22 thread (HT dùng chung execution unit) và 44
thread (oversubscribe, spin trên barrier), tốc độ rơi về 9.6 rồi 5.5, ngang 1 thread. Ngoài
ra, ~16.5 tok/s × ~1.3 GB chỉ là ~20 GB/s, thấp hơn nhiều so với peak lý thuyết của
LPDDR5x. "Bandwidth-bound" ở đây nghĩa là bị chặn bởi băng thông mà các core CPU *thực sự*
kéo được, cộng chi phí đồng bộ. Đó cũng là lý do iGPU Arc trên **cùng** DRAM đạt 24–27 tok/s:
GPU giữ được nhiều memory request đang bay hơn hẳn vài core CPU.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

**Đã làm:** B1 build-compare · B2 `sweep-gpu` · B3 · B4 challenge C7 · B5 C8 semantic cache

**Numbers** (B2 `sweep-gpu`, `benchmarks/bonus-gpu-offload-sweep.md`, tg128, `-t 8`):

```
before:  16.9 tok/s   (-ngl 0, CPU only)
after:   27.0 tok/s   (-ngl 99, toàn bộ lên Arc iGPU)
speedup: 1.60×
```

**Điều này nói lên gì mà deck chưa nói:**

- **B2/B3 — offload trên iGPU là "tất cả hoặc không gì".** Offload toàn bộ nhanh hơn 1.60×,
  nhưng mọi mức offload một phần (8–34 layer) đều ngang hoặc chậm hơn CPU-only. Đo lại
  -ngl 24–35 cũng cho kết quả tương tự. iGPU không có VRAM riêng và không có PCIe: cái mất ở
  đây không phải dung lượng hay băng thông host↔device. Mỗi token đi theo nhịp các layer
  còn lại trên CPU, cộng chi phí đồng bộ ở chỗ graph split, cộng việc CPU và GPU tranh cùng
  một DRAM. Mô hình "VRAM đầy thì offload một phần" của deck không áp dụng cho iGPU.
- **B1 — prebuilt *thắng* bản tự build.** Bản Windows prebuilt không phải "generic": nó ship
  14 biến thể `ggml-cpu-*.dll` và tự chọn `alderlake` (AVX2 + AVX-VNNI) lúc chạy. Decode thì
  hòa (17.1 so với 19.3 tok/s, nằm trong nhiễu). Prefill thì prebuilt Clang **nhanh hơn
  3.42×** (343.4 so với 100.3 tok/s) so với bản MSVC `GGML_NATIVE=ON`
  (`bonus-build-compare-*.md`).
- **B4 / C7 — vector width quan trọng tới AVX2, sau đó compiler mới là yếu tố quyết định.**
  Bản chỉ có SSE4.2 chậm hơn ~7× khi decode và ~15–20× khi prefill. Bật AVX-VNNI tường minh
  trên MSVC không giúp gì. Ngoài ra MSVC "native" âm thầm bỏ qua VNNI và BMI2
  (`bonus-c7-isa-survey.md`).
- **B5 / C8 — semantic cache với embedder yếu là nguy hiểm.** False hit "prefix caching"
  (sim 0.85) có điểm cao hơn một paraphrase thật (0.80), nên không threshold nào sửa được cả
  hai lỗi (`bonus-c8-semantic-cache.md`).

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Con số lớn nhất của cả lab không đến từ model mà từ DNS: `localhost` trên Windows tốn ~2 s
mỗi connection mới, làm pipeline RAG chậm gần 3×. Cái thứ hai là 2-bit chậm hơn 4-bit tới
11× trên iGPU: "ít bit hơn = nhanh hơn" chỉ đúng khi có kernel tốt cho format đó.

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
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Tôi dùng **Claude Code (Anthropic)** làm trợ lý trong suốt lab. Claude đã:
- chạy các lệnh lab trên chính laptop của tôi (probe, setup, bench, tune, serve, smoke, load
  test, metrics, load-report, pipeline, các bonus sweep/build);
- chụp screenshot cửa sổ console thật đang chạy các lệnh đó;
- chẩn đoán các kết quả bất thường bằng các lần đo bổ sung (llama-bench Q2/Q4 trên CPU và
  GPU, sweep thread chi tiết, `localhost` vs `127.0.0.1`);
- soạn nháp phần nhận xét trong `benchmarks/*.md` và file REFLECTION này.

Mọi con số đều được sinh ra từ các lần chạy trên máy đã khai báo trong `hardware.json`.
Không con số nào được sửa tay. Code của lab không bị sửa.
