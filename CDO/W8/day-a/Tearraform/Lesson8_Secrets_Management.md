# Bài 8: Quản Lý Secrets - sensitive, ephemeral, và write-only arguments

Xuyên suốt các bài trước, chúng ta đã biết một rủi ro lớn của Terraform: **State lưu mọi thứ dưới dạng plaintext (văn bản thuần), kể cả mật khẩu**. Bài này sẽ đi sâu vào 3 cơ chế xử lý secret của Terraform, từ cũ đến mới, để giải quyết triệt để vấn đề này.

## Mục tiêu
Phân biệt rõ chức năng của 3 cơ chế:
1.  **`sensitive`**: Làm gì và KHÔNG làm gì.
2.  **`ephemeral resources`** (từ Terraform 1.10).
3.  **`write-only arguments`** (từ Terraform 1.11).

---

## 1. `sensitive`: Chỉ che màn hình (Không bảo vệ State)
Bạn có thể đánh dấu một biến (`variable`) hoặc giá trị đầu ra (`output`) là `sensitive` để Terraform không in nó ra terminal khi chạy lệnh.

```hcl
variable "" {
  type      = string
  sensitive = true
  default   = "super-secret-123"
}

output "password_echo" {
  value     = var.
  sensitive = true
}
```

Khi chạy `terraform apply`, output sẽ bị che lại:
```sql
$ terraform apply -auto-approve
  + password_echo = (sensitive value)
  + conn_string   = (sensitive value)
...
Outputs:

conn_string   = <sensitive>
password_echo = <sensitive>
```

**Tính lan truyền:** Bất kỳ thao tác nào sử dụng biến `sensitive` (ví dụ: ghép chuỗi, tính độ dài) cũng sẽ biến kết quả thành `sensitive`. Bạn buộc phải khai báo tường minh `sensitive = true` ở `output` nếu không Terraform sẽ báo lỗi.
Muốn xem giá trị thật, bạn phải dùng cờ `-raw`: `terraform output -raw password_echo`

### ❌ Nhược điểm cốt lõi của `sensitive`:
Thử kiểm tra file state bằng lệnh `grep`:
```ruby
$ grep 'super-secret-123' terraform.tfstate
super-secret-123
```
👉 **Mật khẩu vẫn nằm thẳng (plaintext) trong file state.** Tính năng `sensitive` chỉ là "chống nhìn trộm qua vai" trên màn hình, chứ không chống đọc file state. Ai lấy được file state là lấy được mật khẩu.

---

## 2. `ephemeral resources`: Giá trị "phù du" (Terraform 1.10+)
Tài nguyên `ephemeral` (phù du) là loại tài nguyên chỉ tồn tại trong đúng 1 lần chạy (`apply`). **Terraform không hề lưu thông tin về chúng vào state hay plan.**

Thay vì dùng `resource`, bạn khai báo bằng khối `ephemeral`:
```hcl
ephemeral "aws_secretsmanager_secret_version" "db" {
  secret_id = "my-db-secret"
}
```
**Ứng dụng:** Dùng để đọc một secret từ nơi cất trữ an toàn (như AWS Secrets Manager, HashiCorp Vault) ngay trong lúc đang apply, để lấy giá trị cấu hình cho tài nguyên khác. Secret được lấy ra sẽ không bao giờ "chạm" vào file state.

---

## 3. `write-only arguments`: Gửi đi rồi quên (Terraform 1.11+)
Ý tưởng của `write-only` là: Terraform gửi secret cho provider (ví dụ AWS) để cấp phát hạ tầng, nhưng sau đó **vứt bỏ giá trị đi mà không lưu vào state hay plan**.

*   **Quy ước tên:** Tham số sẽ có hậu tố `_wo` (write-only) và đi kèm một tham số `_wo_version`.
*   Bản thân giá trị `_wo` không vào state. State chỉ lưu con số `_wo_version` để Terraform biết khi nào bạn muốn cập nhật secret (khi bạn tăng version lên).

### Chứng minh hiệu quả của write-only:
Dựng 2 secret cạnh nhau, một kiểu cũ, một kiểu write-only:

```hcl
variable "secret_value" {
  type      = string
  sensitive = true
  default   = "p@ssw0rd-bai8-demo"
}

# KIỂU CŨ: giá trị sẽ vào state
resource "aws_secretsmanager_secret_version" "legacy" {
  secret_id     = aws_secretsmanager_secret.legacy.id
  secret_string = var.secret_value
}

# KIỂU MỚI: write-only, giá trị KHÔNG vào state
resource "aws_secretsmanager_secret_version" "wo" {
  secret_id                = aws_secretsmanager_secret.wo.id
  secret_string_wo         = var.secret_value
  secret_string_wo_version = 1
}
```

Kiểm tra số lần mật khẩu xuất hiện trong file state sau khi apply:
```shell
$ grep -o 'p@ssw0rd-bai8-demo' terraform.tfstate | wc -l
1
```
Kết quả trả về `1` (thuộc về resource `legacy`). Soi kỹ vào state:
```rust
legacy -> secret_string: p@ssw0rd-bai8-demo | secret_string_wo: None
wo     -> secret_string:                    | secret_string_wo: None
```
Ở bản `wo`, cả hai trường đều rỗng. Giá trị hoàn toàn không bị lưu lại!

Kiểm tra trên AWS, giá trị vẫn được ghi nhận thành công:
```scss
$ aws secretsmanager get-secret-value --secret-id tf-series-bai8-wo --query SecretString --output text
p@ssw0rd-bai8-demo
```
👉 Secret đã được truyền an toàn tới đích (AWS) mà không để lại bất cứ dấu vết nào trong file `terraform.tfstate`.

---

## 💡 Best Practice: Nguồn secret nên đến từ đâu?
Kể cả khi dùng `write-only`, bạn cũng **không nên hardcode** secret trực tiếp vào file `.tf`. Lộ trình an toàn toàn diện nhất cho secret là:
1.  Truyền từ ngoài vào thông qua biến môi trường (`TF_VAR_secret_value`).
2.  Hoặc hoàn hảo nhất: Dùng `ephemeral resource` để đọc bí mật thẳng từ hệ thống Vault/Secrets Manager. Sau đó truyền trực tiếp giá trị phù du đó vào thuộc tính `_wo` (write-only) của một resource khác.
**Kết quả:** Kết hợp "đọc không lưu" (ephemeral) + "ghi không lưu" (write-only) => File state của bạn sẽ sạch bóng mật khẩu từ đầu đến cuối.

---

## 🧹 Dọn dẹp
```cpp
$ terraform destroy -auto-approve
Destroy complete! Resources: 4 destroyed.
```

## Tổng kết Part II
*   `sensitive`: Chỉ che giá trị trên màn hình, vẫn lưu plaintext vào state (chống nhìn qua vai).
*   `ephemeral` (1.10): Đọc giá trị mà không ghi vào state/plan.
*   `write-only arguments` (1.11): Truyền secret đi rồi vứt bỏ, không lưu vào state. Đây là giải pháp hiện đại để tránh lộ lọt secret thay vì chỉ dựa vào mã hóa bucket chứa state.

*Khép lại Part II về State. Part III tiếp theo sẽ tập trung vào tham số hóa cấu hình với Variables, Outputs, và Locals.*
