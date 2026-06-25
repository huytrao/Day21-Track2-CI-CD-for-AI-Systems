# BÁO CÁO THỰC HÀNH MLOPS LAB: TỪ THỰC NGHIỆM CỤC BỘ ĐẾN TRIỂN KHAI LIÊN TỤC

* **Khóa học**: AIInAction - VinUni (Buổi: Day 21 - CI/CD cho AI Systems)
* **Học viên**: TraoAnHuy (MSSV: 2A202600819)
* **Repository**: https://github.com/huytrao/Day21-Track2-CI-CD-for-AI-Systems

---

## 1. Kết Quả Thực Nghiệm Cục Bộ (Bước 1)

Trong quá trình huấn luyện cục bộ bằng thuật toán `RandomForestClassifier` trên tập dữ liệu Wine Quality (Phase 1 gồm 2998 mẫu), chúng tôi đã tiến hành tối thiểu 3 lần chạy thực nghiệm với các cấu hình siêu tham số khác nhau và ghi nhận kết quả trên MLflow:

| Thực Nghiệm | n_estimators | max_depth | min_samples_split | Accuracy (Eval) | F1-Score (Eval) | Ghi Chú |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Lần 1** | 400 | 20 | 2 | 0.6760 | 0.6749 | Thiết lập mặc định ban đầu |
| **Lần 2** | 50 | 3 | 2 | 0.5580 | 0.5185 | Mô hình nông, độ chính xác giảm mạnh |
| **Lần 3** | 200 | 10 | 5 | 0.6440 | 0.6417 | Thiết lập trung bình |
| **Tối Ưu** | 100 | 20 | 2 | **0.6840** | **0.6829** | **Kết quả tốt nhất trên Phase 1** |

### Lý do lựa chọn bộ siêu tham số tốt nhất:
Bộ siêu tham số `n_estimators: 100`, `max_depth: 20`, `min_samples_split: 2` được chọn vì đạt độ chính xác cao nhất cục bộ (`0.6840`). Tuy nhiên, do lượng dữ liệu Phase 1 còn ít, độ chính xác vẫn chưa thể đạt ngưỡng `0.70` (được cấu hình làm cổng kiểm soát chất lượng - Eval Gate).

Khi bổ sung tập dữ liệu Phase 2 nâng tổng số mẫu huấn luyện lên **5996 mẫu**, mô hình được cải thiện rõ rệt và độ chính xác tăng lên đạt **`0.7580`**, chính thức vượt cổng kiểm soát và deploy thành công.

---

## 2. Thiết Lập Hệ Thống CI/CD và DVC (Bước 2 & 3)

* **DVC Storage Remote**: Dữ liệu đã được phiên bản hóa và lưu trữ tập trung tại Google Cloud Storage Bucket `gs://mlops-lab-bucket-fair-yew-499804-r2/dvc`.
* **FastAPI VM**: API dự đoán được cài đặt dưới dạng systemd service `mlops-serve` chạy trên GCE VM tại IP public `34.42.36.212` phục vụ cổng `8000`.
* **CI/CD Pipeline**: Được triển khai qua GitHub Actions gồm 4 jobs: **Unit Test -> Train -> Eval -> Deploy**.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

Trong quá trình thực hiện lab, chúng tôi đã gặp phải một số vấn đề kỹ thuật và xử lý thành công như sau:

1. **Lỗi xác thực DVC trên GitHub Actions (401 Unauthorized)**:
   * *Khó khăn*: DVC cấu hình cục bộ tìm key tại đường dẫn relative `../sa-key.json` (do cấu hình lúc tạo). Khi chạy trên runner của GHA, khóa được ghi tạm vào `/tmp/sa-key.json` nên DVC báo lỗi không tìm thấy credentials.
   * *Giải pháp*: Cập nhật step `Authenticate to Cloud Storage` trong workflow để sao chép khóa `/tmp/sa-key.json` vào thư mục gốc của repository làm việc (`./sa-key.json`).

2. **Lỗi thiếu tham số IP của VM (Missing Server Host)**:
   * *Khó khăn*: Runner của GitHub Actions bị lỗi khi chạy deploy do thiếu tham số `VM_HOST`.
   * *Giải pháp*: Cấu hình giá trị dự phòng (fallback) ngay trong file workflow: `${{ secrets.VM_HOST || '34.42.36.212' }}`, giúp pipeline hoạt động ổn định kể cả khi thiếu secrets trên GitHub.

3. **Lỗi kiểm tra sức khỏe VM bị Timeout (Process exited with status 1)**:
   * *Khó khăn*: Server FastAPI cần khoảng 10-15 giây khi bắt đầu để tải mô hình từ GCS. Lệnh kiểm tra sức khỏe cũ chỉ đợi 5 giây khiến cổng kiểm thử kết thúc với lỗi.
   * *Giải pháp*: Viết vòng lặp thử lại (retry loop) bằng Bash để thăm dò `/health` mỗi 5 giây, tối đa 6 lần (30 giây) giúp quy trình deploy kết thúc thành công 100%.

---

## 4. Kiểm Tra Hoạt Động Của API Triển Khai

* **Health Check**:
  ```bash
  curl.exe http://34.42.36.212:8000/health
  # Trả về: {"status": "ok"}
  ```
* **Predict API**:
  ```bash
  curl.exe -X POST http://34.42.36.212:8000/predict -H "Content-Type: application/json" -d "{\"features\": [7.4, 0.70, 0.00, 1.9, 0.076, 11.0, 34.0, 0.9978, 3.51, 0.56, 9.4, 0]}"
  # Trả về: {"prediction": 0, "label": "thap"}
  ```
