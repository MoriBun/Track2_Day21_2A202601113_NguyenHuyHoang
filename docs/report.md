# Báo cáo Lab MLOps: CI/CD cho AI Systems

**Sinh viên:** Nguyễn Huy Hoàng  
**Repository:** `MoriBun/Track2_Day21_2A202601113_NguyenHuyHoang`

## 1. Mục tiêu và kết quả

Lab xây dựng quy trình MLOps gồm theo dõi thí nghiệm bằng MLflow, quản lý dữ
liệu bằng DVC trên AWS S3, huấn luyện/tự động kiểm thử bằng GitHub Actions và
triển khai API suy luận trên AWS EC2.

Pipeline Bước 2 đã chạy thành công với bốn jobs: **Unit Test → Train → Eval →
Deploy**. Model được tải lên S3 tại `models/latest/model.pkl`; API FastAPI đã
trả về thành công tại các endpoint `/health` và `/predict`.

## 2. Cấu hình mô hình được chọn

Mô hình sử dụng `RandomForestClassifier` với dữ liệu `train_phase1.csv`.

| Hyperparameter | Giá trị |
|---|---:|
| `n_estimators` | 100 |
| `max_depth` | `null` (không giới hạn) |
| `min_samples_split` | 2 |
| `max_features` | 1.0 |
| `random_state` | 0 |

Kết quả trên tập `eval.csv`:

| Metric | Giá trị |
|---|---:|
| Accuracy | 0.682 |
| F1 weighted | 0.6815 |

Ngưỡng Eval gate được đặt là `0.68` theo mức được giảng viên cho phép. Cấu
hình trên đạt ngưỡng khi chỉ sử dụng `train_phase1.csv`, không dùng dữ liệu
`train_phase2.csv` của Bước 3.

## 3. Khó khăn và cách xử lý

- `mlflow==2.13.0` yêu cầu `pkg_resources`, nhưng phiên bản `setuptools` mới
  đã loại bỏ module này. Đã khóa dependency `setuptools<81`.
- EC2 ban đầu chạy Python 3.14, không tương thích với `scikit-learn==1.4.2`.
  Đã thay bằng Ubuntu 22.04 LTS với Python 3.10.
- Accuracy ban đầu thấp hơn gate. Đã tinh chỉnh RandomForest trên đúng dữ liệu
  Bước 2 và điều chỉnh ngưỡng Eval thành 0.68 theo hướng dẫn của giảng viên.
- GitHub Actions trên repository fork cần được bật thủ công trước lần chạy đầu.

## 4. Minh chứng

Các ảnh dưới đây được lưu trong [`docs/evidence/`](evidence/):

1. `01-mlflow-runs.png`: MLflow UI có ít nhất ba runs.
2. `02-github-actions-step2.png`: bốn jobs GitHub Actions màu xanh.
3. `03-api-health-predict.png`: kết quả `/health` và `/predict`.
4. `04-s3-dvc-and-model.png`: dữ liệu DVC và model trên AWS S3.

> Không đưa access key, secret key, private SSH key hoặc file `.pem` vào ảnh
> hay repository.
