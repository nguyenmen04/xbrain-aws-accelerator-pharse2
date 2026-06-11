# Bài 2: Provider, Resource Đầu Tiên, và Vòng Đời init - plan - apply - destroy

Dưới đây là tóm tắt những kiến thức cốt lõi và bắt buộc phải nhớ trong bài học này, kèm theo các ví dụ code thực tế:

## 1. Khai báo Provider và Block `terraform {}`

Mở file `main.tf` và khai báo:

```hcl
terraform {
  required_version = ">= 1.10"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "ap-southeast-1"
}
```

**Lưu ý:**
- **`required_version`**: Ràng buộc phiên bản Terraform Core đang sử dụng để đảm bảo tính đồng nhất môi trường cho team.
- **Pin phiên bản Provider**: Dùng toán tử `~>` (pessimistic operator), ví dụ `~> 6.0`. Terraform sẽ cho phép tải các bản vá lỗi (6.x) nhưng không tự động nhảy lên bản major (7.0). Luôn nên pin phiên bản từ đầu.
- **Quản lý Credential**: **Tuyệt đối không** ghi cứng credential (Access Key/Secret Key) vào trong file `.tf`. Provider có thể tự động tìm credential qua biến môi trường hoặc AWS CLI profile.

## 2. Vòng đời Terraform (Lệnh CLI)

### 2.1. Chuẩn bị: `terraform init`

```shell
$ terraform init
Initializing provider plugins found in the configuration...
- Finding hashicorp/aws versions matching "~> 6.0"...
- Installing hashicorp/aws v6.46.0...
```
- Lệnh này tải binary của Provider về lưu trong thư mục `.terraform/` (thư mục này **không** được commit lên Git).
- Sinh ra **`.terraform.lock.hcl`**: File khóa phiên bản provider chứa chính xác version và checksum. **Phải commit file này lên Git** để cả team dùng chung 1 phiên bản.

### 2.2. Kiểm tra code: `terraform fmt` và `terraform validate`

```shell
$ terraform fmt -check
$ terraform validate
Success! The configuration is valid.
```
- `terraform fmt`: Tự động format lại code (thụt lề, canh lề).
- `terraform validate`: Kiểm tra cú pháp HCL và tính hợp lệ của tham số mà không cần gọi API Cloud. Giúp bắt lỗi đánh máy nhanh chóng.

## 3. Khai báo Resource và Output

Ví dụ tạo một S3 Bucket trong `main.tf`:

```hcl
resource "aws_s3_bucket" "first" {
  bucket_prefix = "tf-series-bai2-"
  force_destroy = true

  tags = {
    Project = "terraform-series"
    Bai     = "02"
  }
}

output "bucket_name" {
  value = aws_s3_bucket.first.id
}

output "bucket_arn" {
  value = aws_s3_bucket.first.arn
}
```
- Có hai nhãn: **Kiểu tài nguyên** (`aws_s3_bucket`) và **Tên local** (`first`). Tên local chỉ để gọi biến bên trong code, Cloud không quan tâm.
- Tên thực tế trên Cloud được định nghĩa qua `bucket_prefix` hoặc `bucket`.

### 3.1. Lên kế hoạch: `terraform plan`

```markdown
$ terraform plan

Terraform will perform the following actions:

  # aws_s3_bucket.first will be created
  + resource "aws_s3_bucket" "first" {
      + arn                         = (known after apply)
      + bucket                      = (known after apply)
      + bucket_prefix               = "tf-series-bai2-"
      + force_destroy               = true
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```
- So sánh `Code` + `State` + `Cloud` và in ra kế hoạch: `+` tạo mới, `~` cập nhật, `-` xóa.
- **`(known after apply)`**: Terraform chưa biết được giá trị thực (chờ AWS tạo xong mới có ARN/ID).

### 3.2. Áp dụng thực thi: `terraform apply`

```shell
$ terraform apply -auto-approve
...
aws_s3_bucket.first: Creating...
aws_s3_bucket.first: Creation complete after 2s [id=tf-series-bai2-20260525025042897800000001]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```
- Thực sự gọi API lên Cloud Provider để tạo tài nguyên. Các giá trị `(known after apply)` giờ đã có số thật.

## 4. State File (`terraform.tfstate`) và Tính Idempotent

Bạn có thể dùng lệnh để xem state hiện tại:
```shell
$ terraform state list
aws_s3_bucket.first

$ terraform show
# aws_s3_bucket.first:
resource "aws_s3_bucket" "first" {
    arn                         = "arn:aws:s3:::tf-series-bai2-2026..."
    bucket                      = "tf-series-bai2-2026..."
...
```

Và file `terraform.tfstate` thực chất là JSON như sau:
```json
{
  "version": 4,
  "terraform_version": "1.15.4",
  "serial": 2,
  ...
}
```

- **State File là gì?** Là "cuốn sổ" lưu trữ ánh xạ giữa tên local trong code và resource thật trên Cloud.
- **⚠️ CẢNH BÁO BẢO MẬT:** File state chứa TẤT CẢ các thuộc tính, bao gồm cả thông tin nhạy cảm (password, private key) ở dạng bản rõ. **Tuyệt đối KHÔNG ĐƯỢC commit file `terraform.tfstate` lên Git.**
- **Tính Idempotent (Lũy đẳng):** Nếu bạn chạy lại plan/apply mà không sửa code:
```shell
$ terraform plan
aws_s3_bucket.first: Refreshing state... 
No changes. Your infrastructure matches the configuration.
```
Cơ chế: Terraform sẽ đối chiếu cả Code, State và Cloud. Nếu khớp, nó báo `No changes` chứ không tạo thêm bucket trùng lặp.

### 4.1. Dọn dẹp: `terraform destroy`
```shell
$ terraform destroy -auto-approve
...
aws_s3_bucket.first: Destroying...
aws_s3_bucket.first: Destruction complete after 1s
```
- Đọc file state và gửi API xóa toàn bộ tài nguyên. 

---

## 💡 Tóm tắt siêu ngắn cần khắc cốt ghi tâm:

1. **`~> 6.0`**: Cấm nhảy phiên bản major.
2. **Thư mục `.terraform/`**: Bỏ vào `.gitignore`.
3. **File `.terraform.lock.hcl`**: Phải commit lên Git.
4. **`terraform validate`**: Check lỗi cú pháp nhanh không cần gọi API.
5. **`(known after apply)`**: Chờ tạo xong Cloud trả về kết quả mới biết.
6. **Tên local của resource**: Dùng để gọi biến bên trong code Terraform, Cloud không quan tâm.
7. **`terraform.tfstate`**: **Có chứa password/secret. KHÔNG ĐƯA LÊN GIT.**
8. **Idempotent**: Chạy 100 lần cũng ra cùng 1 kết quả hạ tầng, vì trước khi tính toán nó sẽ đối chiếu lại toàn bộ hiện trạng.
