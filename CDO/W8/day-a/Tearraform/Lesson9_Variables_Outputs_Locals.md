# Bài 9: Tham Số Hóa và Kiểm Tra Giá Trị Sớm (Variable, Output, Locals)

Cho tới giờ, mọi giá trị trong cấu hình Terraform của chúng ta đều đang được "hardcode" (viết cứng) như region, tên bucket, v.v. Điều này khiến cấu hình không thể tái sử dụng cho các môi trường khác nhau (như dev và prod). Bài này sẽ cung cấp 3 công cụ nền tảng để làm cho cấu hình linh hoạt hơn và cách bắt lỗi giá trị ngay từ sớm.

## 1. Mục tiêu
*   **Tham số hóa cấu hình**: Đưa giá trị vào bằng `variable`, xuất kết quả ra bằng `output`, và gom nhóm tính toán bằng `locals`.
*   **Dựng hàng rào kiểm tra**: Bắt lỗi sớm ở bước `plan` thay vì đợi đến lúc `apply` mới nổ lỗi (dùng `validation`, `precondition`, `postcondition`).

---

## 2. Các thành phần tham số hóa

### a. `variable`: Đầu vào của cấu hình
Dùng để khai báo một giá trị sẽ được truyền từ bên ngoài vào:

```hcl
variable "environment" {
  type        = string
  description = "Môi trường: dev | staging | prod"
  default     = "dev"
}
```
*   `type`: Ràng buộc kiểu dữ liệu (`string`, `number`, `bool`, `list`, `map`, `object`).
*   `description`: Chú thích công dụng của biến.
*   `default`: Giá trị mặc định. Nếu không có `default`, biến này trở thành *bắt buộc* phải truyền khi chạy lệnh.

**Thứ tự ưu tiên truyền giá trị (tăng dần):**
1.  File `terraform.tfvars` (tự động nạp).
2.  Biến môi trường: `TF_VAR_environment=staging`.
3.  Cờ dòng lệnh: `-var environment=staging` (Ưu tiên cao nhất).

### b. `output`: Đầu ra của cấu hình
Công bố một giá trị sau khi `apply` xong.

```hcl
output "bucket_name" {
  value       = aws_s3_bucket.app.id
  description = "Tên của bucket sau khi được tạo"
}
```
*Tác dụng:* In ra màn hình cho bạn xem, để các cấu hình khác (remote state) có thể đọc được, hoặc đóng vai trò là "kết quả trả về" của một Module (sẽ học ở Part IV). Cần che giấu thì dùng thêm `sensitive = true`.

### c. `locals`: Đặt tên cho biểu thức (Biến cục bộ)
Gán tên cho một biểu thức tính toán ở bên trong cấu hình để dùng lại nhiều nơi mà không phải viết lặp lại.

```hcl
locals {
  name_prefix   = "${var.project}-${var.environment}"
  is_production = var.environment == "prod"
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```
Tham chiếu bằng cú pháp `local.TÊN_BIẾN` (ví dụ `local.name_prefix`).
*Lưu ý:* Khác với `variable` (nhận giá trị từ ngoài vào), `locals` tính toán giá trị ngay bên trong cấu hình. Không nên lạm dụng `locals` quá mức vì nó che khuất nguồn gốc thực sự của giá trị, làm code khó đọc hơn.

**Ráp 3 thứ lại với nhau:**
```hcl
resource "aws_s3_bucket" "app" {
  bucket_prefix = "${local.name_prefix}-"
  force_destroy = var.force_destroy
  tags          = local.common_tags
}
```
*Cùng một đoạn code này, chỉ cần đổi `-var environment=prod` là ta có một bucket khác hẳn cho production mà không cần sửa code.*

---

## 3. Các cơ chế kiểm tra giá trị sớm (Bắt lỗi ở bước Plan)

Nếu truyền sai giá trị đầu vào (VD: gõ nhầm `"pruduction"`), lỗi AWS thường rất khó hiểu và nổ ra giữa chừng lúc đang `apply`. Ta cần chặn nó ngay từ bước `plan`.

### a. `validation`: Chặn input sai khi nạp biến
Được viết bên trong block `variable`.

```hcl
variable "environment" {
  type    = string
  default = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment phải là một trong: dev, staging, prod."
  }
}
```
Nếu truyền giá trị ngoài danh sách (`-var environment=production`), Terraform sẽ báo lỗi do bạn tự định nghĩa và **dừng ngay ở bước plan**, chưa hề gọi API nào của AWS.

### b. `precondition` và `postcondition`: Kiểm tra giả định quanh Resource
Nằm trong block `lifecycle` của resource. Nó kiểm tra các logic phức tạp hơn, liên quan đến nhiều biến hoặc trạng thái của resource.

```hcl
resource "aws_s3_bucket" "app" {
  bucket_prefix = "${local.name_prefix}-"
  force_destroy = var.force_destroy

  lifecycle {
    precondition {
      condition     = !local.is_production || !var.force_destroy
      error_message = "Ở prod không được bật force_destroy trên bucket."
    }
  }
}
```

*   **`precondition` (Tiền điều kiện):** Chạy *sau* khi lập plan nhưng *trước* khi tạo resource. Thường dùng để cấm các tổ hợp cấu hình nguy hiểm (như ví dụ trên: cấm môi trường prod bật force_destroy).
*   **`postcondition` (Hậu điều kiện):** Chạy *sau* khi tạo resource xong (hoặc đọc data source xong). Dùng để kiểm tra xem kết quả tạo ra có đúng mong đợi hay không (có thể dùng từ khóa `self` để soi thuộc tính của chính resource đó).

### ⏳ Sơ đồ thời điểm chạy các bước kiểm tra:
```scss
   Biến nạp vào        Plan xong              Apply                  Apply xong
   ───────────         ─────────              ─────                  ──────────
   validation   ──►    precondition   ──►     [tạo/sửa resource] ──► postcondition
   (giá trị biến)      (giả định đầu vào)                            (kết quả đúng?)
```

*(Ghi chú: Còn một block `check` dùng để cảnh báo hạ tầng ngoài vòng đời mà không chặn apply, sẽ được đề cập ở bài 17).*

---

## Tổng kết
*   `variable`: Đưa giá trị từ ngoài vào.
*   `output`: Xuất kết quả ra.
*   `locals`: Tính toán và dùng lại giá trị bên trong cấu hình.
*   `validation`, `precondition`, `postcondition`: Giúp thiết lập hàng rào bảo vệ, bắt lỗi bằng các thông báo thân thiện và dễ hiểu ngay từ khâu `plan` trước khi hạ tầng thật bị ảnh hưởng.
