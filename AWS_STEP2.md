# Bước 2 trên AWS

Repo này được cấu hình cho AWS S3 và EC2. Không đưa access key hoặc private key
vào Git; chúng chỉ được dùng trong AWS CLI, GitHub Secrets hoặc EC2 IAM Role.

## 1. Chuẩn bị AWS CLI và S3 bucket

Đăng nhập AWS CLI bằng IAM Identity Center hoặc access key của tài khoản cá nhân,
sau đó chạy trong PowerShell:

```powershell
$AWS_REGION = "ap-southeast-1"
$BUCKET = "ten-bucket-duy-nhat-cua-ban"

aws sts get-caller-identity
aws s3api create-bucket --bucket $BUCKET --region $AWS_REGION --create-bucket-configuration LocationConstraint=$AWS_REGION
```

Tài khoản hoặc IAM user dùng để chạy DVC/GitHub Actions cần quyền tối thiểu:

- `s3:ListBucket` cho `arn:aws:s3:::TEN_BUCKET`
- `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` cho `arn:aws:s3:::TEN_BUCKET/*`

## 2. DVC remote trên S3

Kích hoạt `.venv`, bảo đảm AWS CLI đã xác thực, rồi chạy:

```powershell
dvc init
dvc remote add -d myremote "s3://$BUCKET/dvc"

dvc add data/train_phase1.csv
dvc add data/eval.csv
dvc add data/train_phase2.csv
dvc push
```

Kiểm tra dữ liệu đã lên S3:

```powershell
aws s3 ls "s3://$BUCKET/dvc/" --recursive
```

## 3. EC2 cho inference API

Tạo EC2 Ubuntu 22.04 (tối thiểu `t3.micro`) trong AWS Console. Security Group cần:

- TCP 22: chỉ IP công khai của bạn (để SSH)
- TCP 8000: IP của bạn để test API; chỉ mở rộng hơn nếu bài lab yêu cầu

Gắn một IAM Role vào EC2 có quyền `s3:GetObject` trên
`arn:aws:s3:::TEN_BUCKET/models/latest/model.pkl`. Dùng IAM Role tránh phải copy
AWS access key lên VM.

SSH vào instance bằng user `ubuntu`, rồi chạy:

```bash
sudo apt update
sudo apt install -y python3-venv
python3 -m venv ~/mlops-venv
~/mlops-venv/bin/pip install fastapi==0.111.0 uvicorn==0.29.0 scikit-learn==1.4.2 joblib==1.4.2 boto3==1.34.107
mkdir -p ~/src ~/models
```

Copy [src/serve.py](src/serve.py) từ máy local lên `~/src/serve.py` trên EC2.
Sau đó, tạo service trên EC2 (thay `TEN_BUCKET_CUA_BAN`):

```bash
sudo tee /etc/systemd/system/mlops-serve.service > /dev/null <<EOF
[Unit]
Description=MLOps Model Inference Server
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu
Environment="S3_BUCKET=TEN_BUCKET_CUA_BAN"
ExecStart=/home/ubuntu/mlops-venv/bin/python /home/ubuntu/src/serve.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable mlops-serve
```

Không khởi động service trước pipeline đầu tiên: model chưa tồn tại trên S3.

## 4. SSH key cho GitHub Actions

Trên máy local:

```powershell
ssh-keygen -t ed25519 -f "$HOME\.ssh\mlops_deploy" -N "" -C "github-actions-deploy"
Get-Content "$HOME\.ssh\mlops_deploy.pub"
```

Thêm public key vừa in vào cuối file `~/.ssh/authorized_keys` của user `ubuntu` trên EC2.

## 5. GitHub Actions Secrets

Tại **GitHub repo → Settings → Secrets and variables → Actions**, tạo các secrets:

| Secret | Giá trị |
|---|---|
| `CLOUD_CREDENTIALS` | `{"aws_access_key_id":"...","aws_secret_access_key":"..."}` của IAM user có quyền S3 |
| `AWS_REGION` | Ví dụ `ap-southeast-1` |
| `CLOUD_BUCKET` | Chỉ tên bucket, không có `s3://` |
| `VM_HOST` | Public IPv4/DNS của EC2 |
| `VM_USER` | `ubuntu` |
| `VM_SSH_KEY` | Toàn bộ nội dung private key `~/.ssh/mlops_deploy` |

## 6. Chạy pipeline và kiểm tra

Commit các file DVC pointer, cấu hình DVC và code, rồi push lên nhánh `main`:

```powershell
git add .dvc .gitignore data/*.dvc src tests .github requirements.txt params.yaml
git commit -m "feat: add AWS S3 CI/CD pipeline"
git push origin main
```

Sau khi cả bốn jobs xanh, kiểm tra API:

```powershell
$VM_IP = "PUBLIC_IPV4_CUA_EC2"
curl "http://$VM_IP`:8000/health"
curl -X POST "http://$VM_IP`:8000/predict" -H "Content-Type: application/json" -d '{"features":[7.4,0.70,0.00,1.9,0.076,11.0,34.0,0.9978,3.51,0.56,9.4,0]}'
```

Kỳ vọng `/health` trả `{"status":"ok"}` và `/predict` trả `prediction` cùng `label` hợp lệ.

Khi service không khởi động, xem log trên EC2:

```bash
sudo journalctl -u mlops-serve -n 50 --no-pager
```
