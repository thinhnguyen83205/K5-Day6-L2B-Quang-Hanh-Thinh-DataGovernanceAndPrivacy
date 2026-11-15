# Group Report — Day 06 Data Lab (Field-to-Release PoC)

> Mọi số liệu eval chính thức phải lấy từ file script sinh ra (`reports/metrics_*.json`,
> `reports/table1_baseline_vs_poc.md`). Số self-test và runtime được ghi riêng, có nguồn rõ ràng.
> 3–5 trang (hoặc README + report.md tương đương).

## 00 · Header

| Mục | Giá trị |
|---|---|
| Group ID | Thinh-Hanh-Quang |
| Challenge | T02 · Raw QC Triage |
| Thành viên (2) + vai trò | 1. Nguyễn Tiến Thịnh (MSSV: 2A202603354) — trích đặc trưng, báo cáo · 2. Nguyễn Đức Hạnh (MSSV: 2A202603344) — baseline, scoring, self-test và phân tích lỗi · Tích hợp T02 vào `src/pipeline.py` còn chờ hoàn tất |
| Git URL | https://github.com/thinhnguyen83205/K5-Day6-L2B-Quang-Hanh-Thinh-DataGovernanceAndPrivacy |
| Tag `demo-180m` (commit) | Chưa tạo (không có tag trong repository) |
| Tag `final-24h` (commit) | Chưa tạo (không có tag trong repository) |
| Deadline (giờ VN) | Chưa có thông tin xác nhận |

## 01 · Problem statement

- Ai bị ảnh hưởng / ai chịu hậu quả khi hệ thống sai? Nhân viên QC/reviewer chịu thêm việc khi ảnh tốt bị gắn cờ; nếu ảnh lỗi bị bỏ sót, dữ liệu chất lượng thấp có thể đi tiếp vào bước sử dụng sau.
- Điểm nghẽn cụ thể: reviewer phải xem thủ công nhiều ảnh trong khi thời gian/budget hữu hạn; cần xếp ảnh nghi lỗi lên trước, không tự động thay thế quyết định của người review.
- Đầu vào → đầu ra: ảnh cùng `images.csv` (`image_id`, `path`, `camera_id`, `light`, `split`) → điểm nghi lỗi và thứ hạng → queue/contact sheet top-20 → người review xác nhận KEEP/RECAPTURE → quyết định HOLD/GO/REWORK có lý do và owner.
- Tiêu chí GO / HOLD / REWORK: hiện chưa có ngưỡng GO định lượng được xác nhận. Giữ HOLD khi chưa có reviewer hoặc chưa có kết quả eval thật; chỉ xem xét GO sau khi reviewer/domain owner duyệt và tiêu chí được chốt trước eval; REWORK khi lỗi ảnh cần chụp lại hoặc pipeline/provenance không đạt.
- Giả định của nhóm (cần DPO/legal/domain owner xác nhận): mọi ảnh trong fixture là dữ liệu synthetic, không có PII thật; các ngưỡng chỉ phục vụ hỗ trợ QC, không phải quyết định tự động cuối cùng.

## 02 · Data & provenance

| Split | N | Nguồn / link | License | Synthetic? |
|---|---|---|---|---|
| dev | 56 ảnh / 14 cảnh | `fixtures/t02_raw_qc/` — fixture sinh offline, seed 42 | CC0-1.0 (synthetic) | Có |
| eval / heldout | 40 ảnh / 10 cảnh | `fixtures/t02_raw_qc/` — heldout theo cảnh; nhãn GT eval chỉ có ở gói giảng viên | CC0-1.0 (synthetic) | Có |

- Schema: `images.csv` gồm `image_id,path,camera_id,light,split`; nhãn evaluator gồm `defect_type ∈ {none, blur, glare, dark, compression}`, `severity ∈ {0,1,2}`, `is_defect`. `gt/labels.csv` không có trong checkout học viên.
- Seed: 42 · Chia theo cảnh gốc, không tách ảnh cùng cảnh qua dev/eval; 14 cảnh dev và 10 cảnh eval.
- `manifest.csv` của T02 được xác minh SHA-256: có (`OK` cho `fixtures/t02_raw_qc/manifest.csv` khi chạy `python -m tools.verify_manifests`). Lệnh toàn repo hiện báo lỗi hash ở 13 fixture khác trên working copy Windows; không dùng các kết quả đó làm bằng chứng T02.
- Dữ liệu thật dùng: không có; không tự tạo nhãn eval. Do thiếu gói GT giảng viên, chưa thể chạy metric eval chính thức.

## 03 · Research & design

- References kỹ thuật: dùng Laplacian variance làm baseline blur, kết hợp các percentile đặc trưng theo thiết kế T02; URL, ngày xem và lý do được liệt kê tại mục 11.
- Sơ đồ data flow: `images.csv + ảnh + manifest` → kiểm hash → trích blur/dark/glare/blockiness → robust z-score theo `(camera_id, light)` (lùi về `light`, rồi toàn bộ khi nhóm nhỏ) → xếp score giảm dần/top-20 → `review_queue.csv` + contact sheet → reviewer/decision log → HOLD chờ duyệt.
- Thành phần offline: không dùng model/weights pretrained; đặc trưng ảnh được tính cục bộ bằng NumPy/SciPy/Pillow, không cần mạng. Fixture T02 khai báo synthetic và CC0-1.0.
- Trade-offs: baseline Laplacian đơn giản, dễ giải thích nhưng không hiệu chỉnh khác biệt camera; PoC kết hợp bốn tín hiệu và chuẩn hóa theo nhóm để giảm ảnh hưởng camera/ánh sáng, đổi lại có nhiều tham số heuristic và cần kiểm chứng trên dữ liệu thật trước khi khái quát hóa.

## 04 · Baseline vs PoC

- Baseline: `score = -laplacian_variance` toàn ảnh, không chuẩn hóa nhóm; `src/t02_baseline.py`, seed 42, budget 20; code ở commit `50c3546`.
- PoC: blur percentile theo lưới 4×4, p95 độ sáng, glare lớn nhất theo ô và blockiness; robust z-score theo nhóm camera/ánh sáng; score là thành phần lỗi lớn nhất. `src/t02_features.py` (`2bb4015`) và `src/t02_scoring.py`/`src/t02_poc.py` (`50c3546`), seed 42, budget 20.
- Cùng budget / heldout / seed: hai lệnh chạy dùng cùng fixture T02, cùng `split=eval` trong protocol, seed 42 và B=20. Tuy nhiên chưa có GT eval trong checkout, nên chưa xác nhận được so sánh metric trên heldout.

**Bảng 1 · Baseline vs PoC & hard slices** — điền bằng output của
`python -m src.compare reports/metrics_baseline.json reports/metrics_poc.json` sau khi có GT. Hiện chưa có file metrics eval:

| Metric (unit) | Baseline | PoC | Δ & chú thích |
|---|---|---|---|
| Defect Recall@B (eval, N=40) | Chưa chạy: thiếu GT eval | Chưa chạy: thiếu GT eval | Chưa có kết quả chính thức; higher is better; B=20 |
| False alarm theo `light=night` | Chưa chạy: thiếu GT eval | Chưa chạy: thiếu GT eval | Chưa có kết quả chính thức; lower is better |
| Review@B / miss / FP | Chưa tính được nếu thiếu nhãn | Chưa tính được nếu thiếu nhãn | Chưa có kết quả chính thức; B=20 |
| Runtime trên 96 ảnh fixture | 0.2388 s | 0.4014 s | Từ `out/baseline/run_info.json` và `out/poc/run_info.json`; một lần chạy, Python 3.13.15, Windows 11 |

Kết quả self-test synthetic được báo riêng tại mục 06, không thay thế bảng eval chính thức.

## 05 · Evaluation protocol

- Primary: `Defect Recall@20 = TP trong top-20 / tổng ảnh lỗi trong 40 ảnh eval`; `Precision@20 = TP / 20`. FP là ảnh sạch trong top-20; FN là ảnh lỗi ngoài top-20. Metric contract được chốt 14/11/2026 15:00 UTC+07:00, trước khi xem eval thật.
- Budget B=20 theo `configs/tracks/t02.yaml`; `none_threshold=1.5` theo default PoC. Không chỉnh theo eval; eval thật chưa chạy.
- CI cho eval thật: chưa tính vì thiếu GT. Contract đề xuất bootstrap theo 10 cảnh gốc, 95%, 10.000 lần; độ lệch chuẩn self-test ở mục 06 không phải CI eval.
- Lệnh evaluate sau khi có overlay GT của giảng viên: `.\.venv\Scripts\python.exe -m src.evaluate --track t02 --pred out\poc\ranked_review.csv --run-name poc --out reports\metrics_poc.json --slices-out reports\slices_poc.csv` (lệnh tương tự với `out\baseline\ranked_review.csv` và `baseline`).

## 06 · Result slices

| Slice | N | Baseline | PoC | Ghi chú |
|---|---|---|---|---|
| Self-test synthetic — Recall@B | 120 ảnh/seed × 10 seed; B=60 | 0.733 ± 0.061 | 0.989 ± 0.015 | Trung bình ± độ lệch chuẩn qua seed 101–110; không phải eval thật |
| Self-test synthetic — Precision@B | 120 ảnh/seed × 10 seed; B=60 | 0.582 ± 0.052 | 0.793 ± 0.114 | Trung bình ± độ lệch chuẩn qua seed 101–110 |
| Self-test synthetic — false alarm `camera=cam_B` | Số ảnh sạch thay đổi theo seed; B=60 | 1.000 ± 0.000 | 0.168 ± 0.118 | Hard negative camera mềm; không đại diện cho eval thật |

- Failure case 1 (synthetic slice, không phải eval): baseline có false-alarm rate `cam_B=1.000 ± 0.000`; self-test cố ý làm ảnh cam_B mềm dù ảnh sạch. Nguyên nhân phù hợp với baseline dùng một điểm Laplacian toàn ảnh, không chuẩn hóa theo camera. PoC giảm chỉ số xuống `0.168 ± 0.118`, nhưng chưa bằng 0.
- Failure case 2: chưa có case ảnh cụ thể được reviewer xác nhận/lưu cùng root cause; cần bổ sung sau khi kiểm tra queue. Không suy diễn nhãn thật từ score.
- Latency / compute: baseline 0.2388 s/96 ảnh, PoC 0.4014 s/96 ảnh; máy theo `run_info.json`: Python 3.13.15, Windows 11, Intel64 Family 6 Model 183 Stepping 1. Đây là một lần chạy cục bộ, chưa phải benchmark lặp.

## 07 · Human & decision

- Review queue: `out/poc/review_queue.csv`, B=20, sắp theo score giảm dần; `out/poc/contact_sheet.html` có ảnh và thông tin xếp hạng. Các trường reviewer/decision/override trong queue hiện còn trống; đây là output local, không được commit.
- Override/audit: chưa có reviewer override và chưa có `out/poc/audit_log.jsonl`; chưa thể khẳng định đã có audit evidence. Hạ tầng `src/common/review.py` hỗ trợ lưu quyết định và append-only audit khi được gọi.
- Decision hiện tại: `out/poc/decision.json` ghi `HOLD`, `reason_code=PENDING_HUMAN_REVIEW`, `n_items=96`, `n_flagged=49`, `budget=20`. Owner vẫn là placeholder `TODO: tên người ký duyệt (role)`, cần nhóm chỉ định trước khi nộp.
- Giữ HOLD khi queue chưa được người có trách nhiệm duyệt, owner chưa xác định, hoặc chưa có metric eval/provenance hợp lệ. REWORK/RECAPTURE cho ảnh reviewer xác nhận lỗi; không tự động release từ heuristic.

## 08 · Reproduce & tests

```powershell
.\.venv\Scripts\python.exe -m src.main --track t02 --pipeline src.t02_baseline --output out\baseline --run-name baseline --offline --seed 42
.\.venv\Scripts\python.exe -m src.main --track t02 --pipeline src.t02_poc --output out\poc --run-name poc --offline --seed 42
.\.venv\Scripts\python.exe -m pytest -q tests\test_t02_scoring.py
```

- `tests/test_t02_scoring.py`: 15 passed. T02 self-test: `.\.venv\Scripts\python.exe -m tools.t02_selftest --seed-start 101 --seeds 10` (trên PowerShell có thể đặt `$env:PYTHONUTF8='1'` để in tiếng Việt). Hai lệnh run trên trả status 0. Lệnh mặc định `--pipeline src.pipeline` hiện chưa triển khai T02 và trả status 3; dùng các module T02 độc lập như lệnh ở trên.
- Máy / CPU / OS / Python: Intel64 Family 6 Model 183 Stepping 1 · Windows 11 · Python 3.13.15 · seed 42 · run git SHA `09550b2`.

## 09 · Limitations & ethics

- Rủi ro pháp lý / PII / license: fixture T02 synthetic, manifest khai báo CC0-1.0 và không chứa PII thật. Chưa dùng dữ liệu thật; trước khi dùng cần xác minh quyền, retention, access control và quy trình xóa với owner/DPO.
- Điều chưa biết; false alarms; nếu scale 10× thì sao: self-test không chứng minh tổng quát hóa; cam_B vẫn có false alarms, còn recall/precision eval thật và failure cases reviewer chưa biết. Chưa benchmark scale 10×; cần đo thời gian/bộ nhớ trên pool lớn hơn.
- Không tuyên bố đạt GDPR hoặc luật Việt Nam; pháp chế/DPO/domain owner cần xác nhận căn cứ sử dụng ảnh và quy trình lưu trữ.
- Kế hoạch cải tiến: tích hợp T02 vào `src/pipeline.py`; cấu hình owner; review và lưu decision/audit; lấy GT overlay để chạy evaluator; lưu hai failure case có ảnh/metadata/root cause; benchmark và cập nhật metrics tự sinh.

## 10 · Team contribution

**Bảng 2 · Individual contribution evidence** (đối chiếu với `git shortlog -sne --all` và `reports/team_contributions.csv`)

| Học viên | Git author/email | Module / vai trò | Commit SHA / diff chứng minh | Trình bày? |
|---|---|---|---|---|
| Nguyễn Tiến Thịnh — MSSV 2A202603354 | `MrNguyen05 <thinhnguyen83205@gmail.com>` | Trích đặc trưng; chỉnh sửa báo cáo nhóm | `src/t02_features.py` (`2bb4015`); cập nhật report (`09550b2`) | Chưa xác nhận |
| Nguyễn Đức Hạnh — MSSV 2A202603344 | `duchanhdev <duchanhdev@gmail.com>` | Baseline Laplacian; scoring, self-test và phân tích lỗi | `src/t02_baseline.py`; `src/t02_scoring.py`; `src/t02_poc.py`; `tools/t02_selftest.py` (`50c3546`) | Chưa xác nhận |

> Nhóm có hai thành viên. Tích hợp T02 vào `src/pipeline.py` chưa hoàn tất (module mặc định trả status 3); không ghi nhận đây là phần việc đã hoàn thành.

## 11 · References

| URL | Ngày xem | 1–2 dòng ý nghĩa (không chép nguyên văn) |
|---|---|---|
| https://docs.opencv.org/4.x/d5/db5/tutorial_laplace_operator.html | 2026-11-15 | Giải thích toán tử Laplace; dùng làm cơ sở cho baseline sharpness bằng phương sai Laplacian, không xem đây là nhãn blur chuẩn. |
| https://numpy.org/doc/stable/reference/generated/numpy.percentile.html | 2026-11-15 | Tài liệu hàm percentile được dùng để tính các đặc trưng percentile trong T02 (ví dụ blur percentile và p95 độ sáng). |

---

**Bảng 3 · Checklist chốt khi nộp**

- [ ] Đã có link repo; chưa tạo tag `final-24h`. Input T02 là synthetic, không chứa PII thật.
- [ ] Có lệnh run độc lập cho baseline/PoC và lệnh test; eval chính thức chỉ chạy được khi giảng viên cung cấp GT. Pipeline mặc định `src.pipeline` chưa tích hợp T02.
- [ ] Có self-test synthetic và decision log HOLD; chưa có metrics eval thật, failure case ảnh được reviewer xác nhận, hoặc reviewer evidence.
- [ ] Có manifest/source provenance, split, seed và offline run; GT eval N=40 được khai báo trong contract nhưng nhãn không có trong checkout học viên.
- [x] Mỗi thành viên có commit code thực chất (`2bb4015`, `50c3546`) và được nêu trong bảng trên.
