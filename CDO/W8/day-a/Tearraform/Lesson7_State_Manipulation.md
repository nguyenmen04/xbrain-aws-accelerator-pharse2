# Bài 7: Thao Tác State - import block, state mv, state rm

Bài này hướng dẫn 3 thao tác can thiệp trực tiếp vào file state của Terraform. Các thao tác này thường được dùng khi làm việc với hạ tầng thực tế để xử lý các tình huống mà chỉ dùng lệnh `apply` là không đủ.

## 1. Mục tiêu cốt lõi
*   **`import` block**: Đưa tài nguyên đã có sẵn (chưa được quản lý) vào sự kiểm soát của Terraform (kèm tính năng tự sinh cấu hình).
*   **`state mv`**: Đổi tên hoặc di chuyển tài nguyên trong state mà không làm phá hủy/tạo lại hạ tầng thật.
*   **`state rm`**: Ngừng quản lý tài nguyên (xóa khỏi state) nhưng vẫn giữ nguyên hạ tầng thật đang chạy.

---

## 2. `import` block: Đưa hạ tầng có sẵn vào quản lý (Khai báo thay vì gõ lệnh)

Khi bạn có một tài nguyên được tạo thủ công (ví dụ bằng AWS Console hay CLI) và muốn Terraform tiếp quản nó.

**Tạo một bucket mẫu bằng AWS CLI (Di sản có sẵn):**
```bash
$ aws s3api create-bucket --bucket tf-series-bai7-preexisting-1779678443 \
    --region ap-southeast-1 \
    --create-bucket-configuration LocationConstraint=ap-southeast-1
```

Từ Terraform 1.5, thay vì dùng lệnh thủ công `terraform import`, bạn sử dụng khối khai báo `import` trực tiếp trong code:

```hcl
import {
  to = aws_s3_bucket.adopted
  id = "tf-series-bai7-preexisting-1779678443"
}
```
*   `to`: Địa chỉ tài nguyên sẽ được mang trong cấu hình Terraform.
*   `id`: Định danh thật của tài nguyên trên Cloud (với S3 bucket thì chính là tên bucket).

### Tự sinh cấu hình với cờ `-generate-config-out`
Thay vì phải tự viết `resource block` từ đầu sao cho khớp với hạ tầng thật, bạn có thể yêu cầu Terraform tự động sinh code dựa vào `import` block ở trên:

```bash
$ terraform plan -generate-config-out=generated.tf
```

Terraform sẽ đọc tài nguyên thật trên AWS và sinh ra bản nháp cấu hình trong file `generated.tf`:
```hcl
# __generated__ by Terraform
# Please review these resources and move them into your main configuration files.
# __generated__ by Terraform from "tf-series-bai7-preexisting-1779678443"
resource "aws_s3_bucket" "adopted" {
  bucket              = "tf-series-bai7-preexisting-1779678443"
  bucket_namespace    = "global"
  force_destroy       = false
  object_lock_enabled = false
  region              = "ap-southeast-1"
  tags                = {}
  tags_all            = {}
}
```
*Lưu ý:* Cấu hình sinh ra là điểm khởi đầu. Bạn cần xem xét, điều chỉnh và chuyển nó vào các file cấu hình chính (ví dụ `main.tf`).

### Thực hiện import
Sau khi đã có code cấu hình, chạy lệnh apply để đưa tài nguyên vào file state:
```bash
$ terraform apply -auto-approve
```
Sau bước này, bạn có thể xóa khối `import` đi vì nó đã hoàn thành nhiệm vụ.

---

## 3. `state mv`: Đổi tên không phá hạ tầng

Khi bạn muốn đổi tên tài nguyên trong code (ví dụ từ `adopted` thành `data`) hoặc gom tài nguyên vào module. Nếu chỉ đổi trong file code `.tf`, Terraform sẽ hiểu là "xóa cái cũ, tạo cái mới". Việc này sẽ gây mất mát dữ liệu (đặc biệt với Database hay Bucket S3).

Để đổi ánh xạ trong file state để resource thật mang địa chỉ mới:
```bash
$ terraform state mv aws_s3_bucket.adopted aws_s3_bucket.data
```

```bash
Move "aws_s3_bucket.adopted" to "aws_s3_bucket.data"
Successfully moved 1 object(s).

$ terraform state list
aws_s3_bucket.data
```
*Lưu ý:*
*   Lệnh này chỉ sửa đổi file state. Bạn vẫn phải chủ động đổi tên tài nguyên tương ứng trong file code (`.tf`) cho khớp.
*   (Từ Terraform 1.1, có thể dùng khối khai báo `moved { ... }` trong code thay cho lệnh này).

---

## 4. `state rm`: Ngừng quản lý, giữ nguyên hạ tầng

Được dùng khi bạn muốn Terraform "quên" một tài nguyên, ví dụ để tách tài nguyên đó sang một project Terraform khác quản lý.

Lệnh này sẽ **gỡ tài nguyên khỏi file state**, nhưng **KHÔNG ĐỤNG TỚI HẠ TẦNG THẬT**:
```bash
$ terraform state rm aws_s3_bucket.data
Removed aws_s3_bucket.data
Successfully removed 1 resource instance(s).

$ terraform state list
$              # rỗng — Terraform không còn quản lý gì
```

Kiểm tra hạ tầng thật (AWS), bucket vẫn còn nguyên:
```bash
$ aws s3api head-bucket --bucket tf-series-bai7-preexisting-1779678443
{
    "BucketArn": "arn:aws:s3:::tf-series-bai7-preexisting-1779678443",
    "BucketRegion": "ap-southeast-1",
    "AccessPointAlias": false
}
```

### 🧹 Dọn dẹp tài nguyên
Điểm khác biệt cốt lõi giữa `state rm` và `destroy` là: `destroy` sẽ xóa luôn hạ tầng thật.
Vì `state rm` chỉ xóa trong state, nên nếu bạn muốn bỏ hẳn tài nguyên đó, bạn phải xóa tay trên Cloud (nếu không nó sẽ trở thành tài nguyên "mồ côi" gây tốn chi phí ngầm):
```bash
$ aws s3 rb s3://tf-series-bai7-preexisting-1779678443 --force
```

---

## Tổng kết chung
*   Các thao tác can thiệp trực tiếp vào State (như `mv`, `rm`) luôn đi kèm rủi ro. Hãy đảm bảo state của bạn **đã được cấu hình khóa (lock)** và **bật versioning (ví dụ trên S3)** để có thể khôi phục khi làm sai.
