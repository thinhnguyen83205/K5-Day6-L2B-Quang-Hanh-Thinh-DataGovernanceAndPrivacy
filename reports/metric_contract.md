# Metric contract — T02 Raw QC Triage

> Contract này khai báo protocol trước khi xem nhãn/kết quả eval. Không dùng split
> `eval` để gán nhãn thủ công, chọn ngưỡng, thay trọng số hoặc điều chỉnh thuật toán.
> Đây là định nghĩa metric và protocol, không phải báo cáo kết quả; chưa có số liệu
> eval thật trong repo học viên.

## 1. Phạm vi và dữ liệu

| Mục | Khai báo |
|---|---|
| Track | T02 — Raw QC Triage |
| Input mặc định | `fixtures/t02_raw_qc/images.csv` và ảnh trong `fixtures/t02_raw_qc/images/` |
| Candidate pool | Manifest có 96 ảnh: 56 `dev` thuộc 14 scene và 40 `eval` thuộc 10 scene |
| Eval population | 40 ảnh có `split=eval` trong ground truth của giảng viên |
| Prediction file | `ranked_review.csv`; phải có `image_id, score`; `pred_defect_type` là cột tùy chọn |
| Ghép prediction với GT | Ghép chính xác theo `image_id`; chỉ tính các ID thuộc eval GT. Mỗi ảnh cần đúng một prediction |
| Ground truth | `defect_type`, `severity`, `is_defect`; chỉ evaluator của giảng viên được đọc `gt/` |
| Seed | 42 cho pipeline/baseline. Mọi lần chạy so sánh phải dùng cùng seed và cùng input pool |

Fixture là dữ liệu synthetic/offline. Nếu nhóm bổ sung dữ liệu thực, phải khai báo
nguồn, license/quyền sử dụng, split và quy tắc gán nhãn trong báo cáo; dữ liệu đó
không thay thế eval fixture trừ khi giảng viên phê duyệt protocol riêng.

## 2. Metric chính và cách tính

**Primary metric: `Defect Recall@B` — càng cao càng tốt.**

- Đơn vị: ảnh (`image_id`).
- `B = 20`; evaluator xếp hạng theo `score` giảm dần và lấy tối đa 20 prediction
  thuộc eval population.
- Tử số: số ảnh lỗi thật (`is_defect = 1`) nằm trong top-B.
- Mẫu số: tổng số ảnh lỗi thật trong 40 ảnh eval.
- Công thức: `Defect Recall@B = defects_in_top_B / defects_in_eval`.
- Nếu eval không có ảnh lỗi, metric không xác định; không thay bằng 0.

Metric chính thức thứ hai: **`Precision@B` — càng cao càng tốt.**

- Tử số: số ảnh lỗi thật trong top-B.
- Mẫu số: `B = 20`.
- Công thức: `Precision@B = defects_in_top_B / 20`.
- Ở pool này, Precision@20 tối đa là 1.0; giá trị tối đa thực tế còn phụ thuộc
  số ảnh lỗi trong eval.

Không diễn giải self-test synthetic là điểm eval thật. Mọi kết quả chính thức phải
ghi kèm numerator, denominator, `N=40`, `B=20` và file JSON do evaluator sinh.

## 3. Baseline, PoC và protocol so sánh

| Mục | Khai báo |
|---|---|
| Baseline | Điểm `score = -laplacian_variance` trên toàn ảnh; score cao được ưu tiên review |
| PoC | Kết hợp blur, dark, glare cục bộ và blockiness; chuẩn hóa robust theo camera/ánh sáng |
| Ngân sách hai phương pháp | Cùng `B=20` trên cùng eval population |
| Pool và split | Cùng manifest T02; chỉ dùng các prediction khớp với 40 eval IDs |
| Seed | 42 cho cả baseline và PoC |
| Tie-break | Giữ thứ tự ổn định khi score bằng nhau; không đổi quy tắc sau khi xem eval |
| Tuning | Chỉ được điều chỉnh trên dev có nhãn; ghi lại thay đổi và chốt trước khi chạy eval |

### Tham số PoC hiện khai báo trong code

Các giá trị dưới đây phản ánh default hiện tại; nhóm cần xác nhận và đóng băng
trước khi chạy đánh giá chính thức. Nếu điều chỉnh dựa trên dev, ghi rõ lý do,
giá trị mới và ngày chốt ở cuối file.

| Tham số | Giá trị hiện tại |
|---|---:|
| Lưới đặc trưng | 4 × 4 ô |
| Blur percentile | 75 |
| Ngưỡng glare | pixel > 245 |
| Đặc trưng dark | percentile độ sáng thứ 95 |
| Block JPEG | 8 pixel |
| Nhóm chuẩn hóa | `(camera_id, light)` → `light` → toàn bộ |
| Số ảnh tối thiểu mỗi nhóm | 6 |
| Scale floor | blur 0.10; dark 2.0; glare 0.02; compression 0.05 |
| Trọng số | blur/dark/glare/compression đều 1.0 |
| Ngưỡng dự đoán `none` | 1.5 |
| Điểm xếp hạng | `max(0, z_blur, z_dark, z_glare, z_compression)` |

Ảnh không đọc được được gắn `score=+inf` và `pred_defect_type=unreadable` trong
PoC để được đưa lên review; phải báo riêng số ảnh không đọc được nếu có.

## 4. Metric phụ, lỗi và slices

| Metric / slice | Định nghĩa | Hướng tốt |
|---|---|---|
| False alarm in top-B theo `light` | Ảnh sạch trong top-B thuộc slice / tổng ảnh sạch của slice đó | Thấp |
| Recall trong top-B theo `light` | Ảnh lỗi của slice nằm trong top-B / tổng ảnh lỗi của slice | Cao |
| Recall theo `defect_type` | Ảnh lỗi loại đó trong top-B / tổng ảnh lỗi loại đó trong eval | Cao |
| Defect-type accuracy | Dự đoán đúng loại trên các ảnh lỗi GT; chỉ tính nếu có `pred_defect_type` | Cao |
| Runtime | Thời gian xử lý và ms/ảnh; báo máy, Python và số ảnh đo | Thấp hơn, nhưng không đổi chất lượng để tối ưu runtime nếu chưa nêu rõ |

Slices bắt buộc báo cáo:

1. `light=day`, `light=dusk`, `light=night` (false alarm và recall).
2. `defect_type=blur`, `dark`, `glare`, `compression` (recall từng loại).

Evaluator chuẩn hiện có slices theo ánh sáng và loại lỗi. Nếu báo cáo thêm slice
theo camera, phải ghi đây là phân tích bổ sung, mô tả công thức và không gọi đó là
metric chuẩn do evaluator tự sinh.

Các lỗi cần nêu: ảnh lỗi bị bỏ khỏi top-B (miss/FN), ảnh sạch lọt top-B (false
alarm/FP), ảnh không đọc được, và các failure case theo điều kiện ánh sáng/camera.

## 5. Độ bất định và báo cáo số liệu

- Báo numerator/denominator bên cạnh mọi tỷ lệ; không chỉ ghi phần trăm đã làm tròn.
- Eval có 40 ảnh theo 10 scene, nên các ảnh cùng scene có thể phụ thuộc nhau.
- Nếu nhóm tính khoảng tin cậy, ưu tiên bootstrap theo **scene gốc** (resample
  scene, giữ nguyên các ảnh trong scene), 95%, 10.000 lần lặp; dùng cùng mẫu
  bootstrap cho baseline và PoC khi so sánh.
- Nếu chỉ có thể tính Wilson interval theo ảnh, ghi rõ đây là xấp xỉ coi ảnh độc
  lập và có thể đánh giá thấp bất định do gom cụm theo scene.
- Nếu không tính được CI, ghi “chưa tính”; không tự suy diễn CI từ độ lệch chuẩn
  của self-test.

## 6. Lệnh tái lập

Chạy baseline và PoC trên cùng fixture:

```powershell
.\.venv\Scripts\python.exe -m src.main --track t02 --pipeline src.t02_baseline --output out\baseline --run-name baseline --offline --seed 42
.\.venv\Scripts\python.exe -m src.main --track t02 --pipeline src.t02_poc --output out\poc --run-name poc --offline --seed 42
```

Sau khi giảng viên cung cấp ground truth, chạy evaluator và lưu kết quả:

```powershell
.\.venv\Scripts\python.exe -m src.evaluate --track t02 --pred out\baseline\ranked_review.csv --run-name baseline --out reports\metrics_baseline.json --slices-out reports\slices_baseline.csv
.\.venv\Scripts\python.exe -m src.evaluate --track t02 --pred out\poc\ranked_review.csv --run-name poc --out reports\metrics_poc.json --slices-out reports\slices_poc.csv
.\.venv\Scripts\python.exe -m src.compare reports\metrics_baseline.json reports\metrics_poc.json --out reports\table1_baseline_vs_poc.md
```

Các lệnh evaluator cần `fixtures/t02_raw_qc/gt/labels.csv`, vốn không có trong repo
học viên. Không tạo nhãn eval giả để làm cho lệnh chạy.

## 7. Chốt protocol

| Mục | Giá trị |
|---|---|
| Nhóm | Thinh-Hanh-Quang |
| Người chịu trách nhiệm chốt (tên/vai trò) | Nguyễn Tiến Thinh |
| Tham số mặc định được chấp nhận hoặc thay đổi | Chấp nhận các default hiện tại trong code; không thay đổi |
| Ngày giờ chốt, trước khi chạy eval | 14/11/2026 15:00 (UTC+07:00), theo xác nhận của nhóm |
| Trạng thái eval khi chốt | Chỉ xem self-test synthetic; chưa xem nhãn/kết quả eval thật |
| Commit SHA chứa contract đã chốt | Chưa commit — điền sau khi commit |

> Nếu baseline bằng 0 hoặc metric không xác định, ghi đúng giá trị và lý do; không
> báo phần trăm cải thiện khi phép chia không có ý nghĩa. Không sửa contract sau
> khi đã xem kết quả eval; nếu phát hiện lỗi protocol, ghi nhận và xin giảng viên
> xác nhận trước khi chạy lại.
