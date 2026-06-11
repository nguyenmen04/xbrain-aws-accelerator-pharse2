# Lesson 6: Remote State trên S3 với `use_lockfile`

Dưới đây là những điều cốt lõi nhất cần nhớ về cách chuyển Terraform State từ máy cá nhân lên AWS S3.

## 1. Tại sao phải vứt Local State đi?
Local State (file `terraform.tfstate` lưu trên máy tính) có 3 "điểm chết":
- **Không chia sẻ được:** Đồng nghiệp không có file này nên không thể chạy code chung trên một hạ tầng.
- **Không an toàn:** File chứa dữ liệu nhạy cảm (mật khẩu, secret key) dưới dạng văn bản thô (plaintext), lỡ push lên Github là lộ toàn bộ thông tin.
- **Xung đột ghi đè:** Hai người cùng gõ `terraform apply` một lúc -> Ghi đè file -> Cấu trúc hạ tầng bị hỏng và sai lệch.

## 2. Bài toán "Con Gà - Quả Trứng" khi dùng S3
- **Vấn đề:** Muốn lưu State lên S3 thì phải có Bucket S3. Mà muốn tạo Bucket S3 thì lại phải dùng Terraform tạo.
- **Cách giải quyết (Bootstrap):** Dùng chiến thuật "chia để trị". 
  - Bước 1: Tạo một thư mục Terraform siêu nhỏ (chạy bằng local state) chỉ làm duy nhất 1 việc: Tạo ra một cái Bucket S3 (bật Versioning, Mã hóa Server-side, Chặn Public). 
  - Bước 2: Sau khi tạo xong, cấu hình Terraform của dự án chính sẽ trỏ `backend` vào cái Bucket vừa sinh ra này.

**Ví dụ code cấu hình Bootstrap (Chỉ dùng để tạo Bucket chứa State):**
```hcl
resource "aws_s3_bucket" "state" {
  bucket_prefix = "tf-series-state-"
  force_destroy = true # chỉ để dọn lab; KHÔNG bật ở thật
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "state" {
  bucket                  = aws_s3_bucket.state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

## 3. Cú lừa từ các tài liệu cũ: Quên DynamoDB đi!
- ❌ **Ngày xưa:** Để khoá State (Locking - tránh 2 người cùng chạy đè lên nhau), người ta bắt buộc phải dựng thêm một bảng **DynamoDB** rất cồng kềnh. Cách này nay đã **LỖI THỜI (Deprecated)**.
- ✅ **Bây giờ:** Chỉ cần thêm đúng 1 dòng **`use_lockfile = true`** vào cấu hình `backend "s3"`. Terraform sẽ tự khoá State ngay trên chính S3. Không cần DynamoDB, không cần thêm hạ tầng thừa thãi.

**Ví dụ code cấu hình Backend (Nằm trong dự án chính):**
```hcl
terraform {
  required_version = ">= 1.10"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }

  backend "s3" {
    bucket       = "tf-series-state-20260525030320917300000001"
    key          = "app/terraform.tfstate"
    region       = "ap-southeast-1"
    encrypt      = true
    use_lockfile = true   # Dòng cực kỳ quan trọng để khoá State bằng S3
  }
}
```

## 4. Cơ chế khoá (Locking) của `use_lockfile` hoạt động ra sao?
- Bất cứ khi nào bạn gõ `apply` hoặc `plan`, Terraform sẽ kết nối lên S3 và xin tạo ra 1 file rác tên là `<key>.tflock`.
- S3 cung cấp tính năng "Ghi có điều kiện": **"Chỉ cho phép tạo nếu object này chưa từng tồn tại"**.
- Nếu bạn gõ lệnh trước, file `.tflock` được tạo. Nếu lúc đó đồng nghiệp của bạn cũng gõ `apply`, S3 sẽ lập tức chặn người kia lại vì file `.tflock` đã tồn tại do bạn giữ. 

**Ví dụ báo lỗi khi bị kẹt khoá (Xung đột xảy ra):**
```text
Error: Error acquiring the state lock

Error message: operation error S3: PutObject, https response error
StatusCode: 412, ... api error PreconditionFailed: At least one of the
pre-conditions you specified did not hold

Lock Info:
  ID:        ce1c8cbb-5211-21f5-7878-9b867e6246cb
  Path:      tf-series-state-2026.../app/terraform.tfstate
  Operation: OperationTypeApply
  Who:       ops@workstation.local
```
- Dòng `StatusCode: 412 PreconditionFailed` chứng tỏ cơ chế khoá đang hoạt động hoàn hảo. Chạy xong Terraform tự xoá file `.tflock` này đi để nhả khoá.
- **Mẹo cứu hộ:** Nếu tiến trình đang chạy bị chết giữa chừng khiến file `.tflock` chưa kịp xoá, state sẽ bị kẹt. Dùng lệnh `terraform force-unlock <ID_của_khoá>` để gỡ thủ công.

## 5. Hai quy tắc "Bất di bất dịch"
- **Không dùng biến (variable) trong block Backend:** Block `backend "s3"` được đọc cực kỳ sớm (khi khởi tạo), trước cả lúc Terraform kịp xử lý biến. Nên toàn bộ tên Bucket, Key, Region phải **ghi cứng (hardcode)**.
- **Trình tự dọn dẹp (Destroy):** Luôn phải destroy các resource của dự án thực tế trước, vào AWS xoá sạch các object trong Bucket, rồi cuối cùng mới chạy destroy cái dự án Bootstrap tạo Bucket. Làm ngược lại sẽ gây kẹt State.
