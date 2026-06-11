# Bài 4: State - Terraform Lưu Gì, Vì Sao Cần, và Drift

Dưới đây là tóm tắt kiến thức quan trọng nhất về **State** trong Terraform, thứ dễ gây sự cố nhất nếu không hiểu rõ. Toàn bộ code và ví dụ lệnh được giữ nguyên để tiện tra cứu.

## 1. Vì sao Terraform cần file State? (4 lý do chính)

Không phải tự nhiên Terraform sinh ra một file ở giữa thay vì hỏi thẳng AWS. Có 4 lý do:

1. **Ánh xạ với thực tế (Mapping):** 
   - Code của bạn chỉ biết tên local (VD: `aws_s3_bucket.demo`), còn AWS chỉ biết ID thực tế (`tf-series-bai4-...`). State làm cầu nối ghi nhớ: `"resource local demo chính là bucket id này trên AWS"`.
2. **Metadata về phụ thuộc:** 
   - State nhớ thứ tự phụ thuộc giữa các tài nguyên. Rất quan trọng khi chạy lệnh xóa (`destroy`), giúp Terraform biết không được xóa subnet trước khi xóa EC2 bên trong nó.
3. **Hiệu năng:** 
   - State đóng vai trò như một bản cache. Hỏi lại từng resource qua API mỗi lần plan là quá chậm. Terraform tính toán kế hoạch nhanh hơn nhờ dựa vào bản cache này.
4. **Đồng bộ khi làm việc nhóm:** 
   - Đặt state ở một nơi chung (remote) giúp team không apply đè lên nhau (có cơ chế lock).

## 2. State lưu gì và cách xem an toàn

State lưu toàn bộ thuộc tính của resource mà Terraform đang quản lý. Thay vì mở file JSON thô `.tfstate`, hãy dùng CLI:

```ruby
# Liệt kê mọi resource Terraform đang quản lý
$ terraform state list
aws_s3_bucket.demo

# In toàn bộ thuộc tính của một resource cụ thể (Đọc từ cache)
$ terraform state show aws_s3_bucket.demo
    bucket                      = "tf-series-bai4-20260525025632034200000001"
    hosted_zone_id              = "Z3O0J2DXBE1FTB"
    id                          = "tf-series-bai4-20260525025632034200000001"
    tags                        = {
        "Env"     = "dev"
        "Project" = "terraform-series"
    }
    ...
```

## 3. Cơ chế Refresh: So sánh 3 chiều

Mỗi lần chạy `plan` (hoặc `apply`), Terraform sẽ thực hiện bước **refresh**: Gọi API hỏi Cloud xem resource hiện ra sao và cập nhật vào bộ nhớ. Sau đó nó so sánh 3 thứ:

```r
   main.tf                terraform.tfstate            AWS (thực tế)
   (bạn MUỐN gì)          (lần cuối ĐÃ BIẾT)           (đang CÓ gì)
   Env = "dev"            Env = "dev"                  Env = "dev"
        │                       │                            │
        └───────────┬───────────┴──────────────┬────────────┘
                    ▼                           ▼
              refresh: đọc AWS, cập nhật bản trong bộ nhớ
                    │
                    ▼
              so cấu hình ⟷ thực tế  →  diff
```
Nếu cả 3 khớp nhau, `plan` sẽ báo: **"No changes"**.

## 4. Drift là gì và Xử lý ra sao?

### 4.1. Khái niệm Drift
**Drift** xảy ra khi ai đó sửa hạ tầng bên ngoài Terraform (chỉnh bằng tay trên Console AWS, dùng CLI khác). Lúc này: **Thực tế trên Cloud ≠ Cấu hình Terraform.**

### 4.2. Hai cách xử lý khi gặp Drift

**Cách 1: Xóa thay đổi tay (Mặc định)**
Terraform coi cấu hình (Code) là chân lý. Nếu bạn chạy `terraform plan` bình thường, Terraform sẽ đề xuất kéo thực tế về lại đúng như code.
Ví dụ: Ai đó đổi tag thành `production`, Terraform sẽ đề xuất đổi lại thành `dev`.

```bash
$ terraform plan
aws_s3_bucket.demo: Refreshing state... [id=tf-series-bai4-20260525025632034200000001]

  ~ update in-place

  # aws_s3_bucket.demo will be updated in-place
  ~ resource "aws_s3_bucket" "demo" {
        id   = "tf-series-bai4-20260525025632034200000001"
      ~ tags = {
          ~ "Env"     = "production" -> "dev"
            "Project" = "terraform-series"
        }
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

**Cách 2: Chấp nhận thay đổi tay (`-refresh-only`)**
Nếu thay đổi tay là cố ý, bạn dùng cờ `-refresh-only`. Lệnh này **chỉ cập nhật State cho khớp thực tế, KHÔNG đụng/sửa AWS**. (Sau đó bạn nên tự vào file `main.tf` sửa code lại cho khớp).

```python
$ terraform plan -refresh-only
Note: Objects have changed outside of Terraform

  # aws_s3_bucket.demo has changed
  ~ resource "aws_s3_bucket" "demo" {
      ~ tags = {
          ~ "Env"     = "dev" -> "production"
            "Project" = "terraform-series"
        }
    }

This is a refresh-only plan, so Terraform will not take any actions to undo
these. If you were expecting these changes then you can apply this plan to
record the updated values in the Terraform state without changing any remote
objects.
```

## 5. ⚠️ Cảnh báo cực kỳ quan trọng: State là Plaintext!

- File state (`terraform.tfstate`) chứa MỌI THUỘC TÍNH, kể cả **mật khẩu database, private key...** hoàn toàn ở dạng văn bản không mã hoá (plaintext).
- Tuyệt đối **KHÔNG** commit file này vào Git.
- Giải pháp thực tế: Cất state ở nơi an toàn (VD: AWS S3), có mã hóa và kiểm soát truy cập (Remote State).

---

## 💡 Tóm tắt 5 gạch đầu dòng:
1. **State cần vì**: Ánh xạ ID, nhớ phụ thuộc, cache cho nhanh, và đồng bộ team.
2. Dùng lệnh `terraform state list` và `terraform state show` để đọc state an toàn thay vì mở file JSON.
3. Trước khi báo thay đổi, Terraform luôn **refresh** (lấy thực tế Cloud so với Code và State hiện tại).
4. **Drift**: Xảy ra khi thao tác tay trên Cloud khác với Code. Khắc phục bằng cách `apply` (để đè thay đổi tay) hoặc `plan -refresh-only` (chấp nhận lưu vào state).
5. **State chứa mật khẩu không mã hoá. Đừng bao giờ đẩy lên Git.**
