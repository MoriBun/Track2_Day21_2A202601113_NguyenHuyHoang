# Thiết lập môi trường phát triển

Yêu cầu: Python 3.10 trở lên và `pip`.

```bash
python --version
```

Nếu máy Windows không nhận lệnh `python`, dùng `py` thay cho `python` trong các lệnh dưới đây.

## Windows PowerShell

Chạy tại thư mục gốc của repository:

```powershell
py -3.10 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python generate_data.py
```

`requirements.txt` đã khóa `setuptools<81` để tương thích với `mlflow==2.13.0`.
Nếu bạn đã cài dependencies trước khi có thay đổi này và gặp lỗi
`ModuleNotFoundError: No module named 'pkg_resources'`, chạy lại:

```powershell
python -m pip install --upgrade --force-reinstall "setuptools<81"
```

Nếu thấy lỗi `PermissionError` có đường dẫn `AppData\\Local\\Temp`, dùng thư mục
tạm trong repo cho phiên PowerShell hiện tại trước khi chạy test hoặc train:

```powershell
New-Item -ItemType Directory -Path .tmp -Force | Out-Null
$env:TEMP = (Resolve-Path .tmp).Path
$env:TMP = $env:TEMP
```

Nếu PowerShell chặn script kích hoạt, chỉ cần cho phép trong phiên terminal hiện tại rồi chạy lại lệnh activate:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

## Windows Command Prompt (CMD)

```bat
py -3.10 -m venv .venv
.venv\Scripts\activate.bat
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python generate_data.py
```

## Linux / macOS

```bash
python3.10 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python generate_data.py
```

## Kiểm tra và sử dụng

```bash
python -m pytest -v
python src/train.py
mlflow ui --backend-store-uri sqlite:///mlflow.db
```

Lưu ý: mã nguồn lab ban đầu có các mục `TODO` trong `src/train.py` và
`tests/test_train.py`. Sau khi sửa lỗi dependency, cần hoàn thành các mục này
thì test và lệnh huấn luyện mới chạy thành công.

Mở giao diện MLflow tại `http://127.0.0.1:5000` sau khi chạy lệnh cuối. Khi làm xong, thoát môi trường ảo bằng:

```bash
deactivate
```

> `.venv/`, dữ liệu sinh ra, model và các artifact cục bộ đã được bỏ qua bởi Git.
