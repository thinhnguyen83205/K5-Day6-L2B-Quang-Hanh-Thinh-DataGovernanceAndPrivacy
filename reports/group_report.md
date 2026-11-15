# Group Report — Day 06 Data Lab (Field-to-Release PoC)

> Điền đầy đủ, cụ thể, không viết chung chung. Mọi con số phải lấy từ file do script sinh ra
> (`reports/metrics_*.json`, `reports/table1_baseline_vs_poc.md`) — không gõ tay, không làm tròn mất đơn vị.
> 3–5 trang (hoặc README + report.md tương đương).

## 00 · Header

| Mục | Giá trị |
|---|---|
| Group ID | Thinh-Hanh-Quang |
| Challenge | T02 · Raw QC Triage |
| Thành viên (2) + vai trò | 1. Nguyễn Tiến Thịnh (MSSV: 2A202603354) — đặc trưng, QA/check review và báo cáo · 2. Nguyễn Đức Hạnh (MSSV: 2A202603344) — baseline, scoring và phân tích lỗi · Phần việc tích hợp pipeline và đánh giá end-to-end được cả hai phối hợp thực hiện |
| Git URL | ____ |
| Tag `demo-180m` (commit) | ____ |
| Tag `final-24h` (commit) | ____ |
| Deadline (giờ VN) | ____ |

## 01 · Problem statement

- Ai bị ảnh hưởng / ai chịu hậu quả khi hệ thống sai? ____
- Điểm nghẽn cụ thể: ____
- Đầu vào → đầu ra: ____
- Tiêu chí GO / HOLD / REWORK (chốt trước khi chạy eval): ____
- Giả định của nhóm (cần DPO/legal/domain owner xác nhận): ____

## 02 · Data & provenance

| Split | N | Nguồn / link | License | Synthetic? |
|---|---|---|---|---|
| dev | | | | |
| eval / heldout | | | | |

- Schema & class mapping: ____
- Seed: 42 · Cách chia split (theo scene/group/segment?): ____
- `manifest.csv` verified (sha256): có / không — lệnh: `python -m tools.verify_manifests`
- Dữ liệu thật dùng (nếu có) và cách xây GT cho nó: ____

## 03 · Research & design

- References kỹ thuật: dùng Laplacian variance làm baseline blur, kết hợp các percentile đặc trưng theo thiết kế T02; URL, ngày xem và lý do được liệt kê tại mục 11.
- Sơ đồ data flow (input → process → rank/QC → review → decision → report): ____
- Thành phần offline (model/weights + checksum + license nếu có): ____
- Trade-offs đã cân nhắc: ____

## 04 · Baseline vs PoC

- Baseline (mô tả, version/config, commit): ____
- PoC (mô tả, version/config, commit): ____
- Cùng budget / cùng heldout / cùng seed: xác nhận ____

**Bảng 1 · Baseline vs PoC & hard slices** — dán output của
`python -m src.compare reports/metrics_baseline.json reports/metrics_poc.json`:

| Metric (unit) | Baseline | PoC | Δ & chú thích |
|---|---|---|---|
| Primary: ______ | ____ | ____ | ____; higher/lower? |
| Hard-slice: ______ | ____ | ____ | ____; N=__ |
| Review@B / miss / FP | ____ | ____ | ____; B=__ |
| Runtime / 100 items | ____ | ____ | ____; CPU=__ |

## 05 · Evaluation protocol

- Công thức primary metric, đơn vị đánh giá, TP/FP/FN, denominator: ____ (xem `reports/metric_contract.md`)
- Threshold / budget B / K và lúc chốt: ____
- Độ bất định (CI/bootstrap/variance) khi N nhỏ: ____
- Lệnh evaluate chính xác: `python -m src.evaluate --track t__ --pred ... --out ...`

## 06 · Result slices

| Slice | N | Baseline | PoC | Ghi chú |
|---|---|---|---|---|
| | | | | |
| | | | | |

- Failure case 1 (ảnh/metadata + root cause): ____
- Failure case 2: ____
- Latency / compute (máy, CPU, ms/item): ____

## 07 · Human & decision

- Review queue (file, B, cách ưu tiên): ____
- Cách override + audit log (`out/audit_log.jsonl`): ____
- Decision policy + `reason_code` + owner: ____
- Khi nào HOLD/REWORK thay vì tự động release: ____

## 08 · Reproduce & tests

```bash
make setup          # hoặc: python -m venv .venv && pip install -r requirements.txt
python -m src.main --input fixtures/<track> --config configs/lab.yaml --output out/ --offline --seed 42
python -m src.evaluate --track t__ --pred out/<file> --out reports/metrics_poc.json
pytest -q
```

- Output của `pytest -q`: ____
- Máy / CPU / OS / Python: ____ · git sha: ____

## 09 · Limitations & ethics

- Rủi ro pháp lý / PII / license: ____
- Điều chưa biết; false alarms; nếu scale 10× thì sao: ____
- Không tuyên bố "đạt GDPR / luật Việt Nam" từ checklist tự động; giả định cần DPO/legal xác nhận: ____
- Kế hoạch cải tiến: ____

## 10 · Team contribution

**Bảng 2 · Individual contribution evidence** (đối chiếu với `git shortlog -sne --all` và `reports/team_contributions.csv`)

| Học viên | Git author/email | Module / vai trò | Commit SHA / diff chứng minh | Trình bày? |
|---|---|---|---|---|
| Nguyễn Tiến Thịnh — MSSV 2A202603354 | [điền] | Trích đặc trưng; QA/check review queue, contact sheet và decision log; tổng hợp, viết báo cáo nhóm | `src/t02_features.py`; `tests/test_t02_scoring.py`; `out/poc/review_queue.csv`; `out/poc/contact_sheet.html`; `reports/group_report.md`; [SHA sau khi commit] | [điền] |
| Nguyễn Đức Hạnh — MSSV 2A202603344 | [điền] | Baseline Laplacian; chuẩn hóa theo nhóm; scoring, phân loại lỗi và phân tích false alarm | `src/t02_scoring.py`; `src/t02_baseline.py`; `src/t02_poc.py`; `tools/t02_eval_dev.py`; `tools/t02_selftest.py`; [SHA sau khi commit] | [điền] |

> Phần việc tích hợp `score_images(...)` vào pipeline và chạy E2E do Nguyễn Tiến Thịnh và Nguyễn Đức Hạnh phối hợp thực hiện; nhóm không có thành viên thứ ba.

## 11 · References

| URL | Ngày xem | 1–2 dòng ý nghĩa (không chép nguyên văn) |
|---|---|---|
| https://docs.opencv.org/4.x/d5/db5/tutorial_laplace_operator.html | 2026-11-15 | Giải thích toán tử Laplace; dùng làm cơ sở cho baseline sharpness bằng phương sai Laplacian, không xem đây là nhãn blur chuẩn. |
| https://numpy.org/doc/stable/reference/generated/numpy.percentile.html | 2026-11-15 | Tài liệu hàm percentile được dùng để tính các đặc trưng percentile trong T02 (ví dụ blur percentile và p95 độ sáng). |

---

**Bảng 3 · Checklist chốt khi nộp**

- [v] Link repo và tag cuối cùng trong README; input mẫu không chứa PII thật.
- [v] Đúng 1 lệnh run, 1 lệnh evaluate, 1 lệnh tests; không cần download khi chạy.
- [v] Có baseline/PoC metrics, ≥1 failure case, reviewer evidence, decision log.
- [v] Có license/source provenance, data splits, seed, offline environment, GT/eval N.
- [v] Mỗi thành viên có ≥1 commit thực chất và được nêu minh bạch trong bảng trên.
