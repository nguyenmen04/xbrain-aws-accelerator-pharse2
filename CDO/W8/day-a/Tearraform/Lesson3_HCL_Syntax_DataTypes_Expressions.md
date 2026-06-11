# Bài 3: HCL Từ Trong Ra Ngoài - Block, Kiểu Dữ Liệu, Biểu Thức

Dưới đây là tóm tắt những kiến thức cốt lõi về ngôn ngữ HCL (HashiCorp Configuration Language). Toàn bộ code, biến và lệnh được giữ nguyên để tiện tra cứu và giải thích.

## 1. Ba thành phần của cú pháp HCL

HCL được xây quanh hai khối cơ bản: **argument** và **block**.

### Cấu trúc của Block và Argument
Dưới đây là phân tích chi tiết từng thành phần trong một block:

```wasm
resource  "aws_s3_bucket"  "first"  {
   │           │              │      └── body (thân block)
   │           │              └───────── label 2: tên local
   │           └──────────────────────── label 1: kiểu resource
   └──────────────────────────────────── type: loại block

    bucket_prefix = "tf-series-bai2-"
    └── identifier ┘ └─ biểu thức ─┘
    └──────────── argument ────────────┘

    tags = {                # giá trị kiểu map
      Project = "terraform-series"
    }
}
```
- **Argument**: Gán một giá trị cho một tên (VD: `bucket_prefix = "tf-series-bai2-"`). Bên trái là identifier, bên phải là biểu thức.
- **Block**: Là vùng chứa, gồm `type` (kiểu block), có thể có hoặc không có `label` (nhãn), và `body` nằm trong cặp ngoặc nhọn `{}`. 
  - `resource` cần 2 label.
  - `provider` cần 1 label.
  - `terraform` không cần label nào.
- **Comment (Chú thích)**: Dùng `#` (khuyên dùng), `//` (tự động bị format thành `#`), hoặc `/* */` (nhiều dòng).

## 2. Phòng thí nghiệm: `terraform console`

Dùng để test thử biểu thức HCL ngay tại chỗ, in ra kết quả mà không cần `apply`.

```shell
$ echo 'upper("hello")' | terraform console
"HELLO"
$ echo '5 + 3 * 2' | terraform console
11
```
*Lưu ý: Console tuân thủ đúng thứ tự ưu tiên toán tử (nhân chia trước, cộng trừ sau).*

## 3. Sáu kiểu giá trị trong Terraform

### 3.1. Ba kiểu nguyên thủy (Primitive types)
- **`string`** (chuỗi): Nằm trong ngoặc kép `"ap-southeast-1"`.
- **`number`** (số): Cả số nguyên lẫn số thực (15, 6.283).
- **`bool`** (logic): `true` hoặc `false`.

```bash
$ echo 'true && false' | terraform console
false
```

### 3.2. Hai nhóm kiểu gộp (Collection/Structural types)
- **`list` / `tuple`**: Dãy giá trị có thứ tự, đánh số từ 0. 
  - VD: `["us-east-1a", "us-east-1c"]`
  - Khác biệt: `list` đòi các phần tử cùng kiểu, `tuple` cho phép trộn nhiều kiểu.
- **`map` / `object`**: Nhóm giá trị gắn nhãn tên (key-value).
  - Khác biệt: `map` đòi các giá trị cùng kiểu, `object` cho phép mỗi key có value kiểu khác nhau.

```bash
$ echo '{ name = "web", port = 443 }' | terraform console
{
  "name" = "web"
  "port" = 443
}
```

### 3.3. Kiểu đặc biệt: `null`
- Biểu thị "sự vắng mặt" hoặc bị bỏ qua. 
- Khi gán `null` cho một argument, Terraform xử lý như thể bạn **không hề viết argument đó ra** (nó sẽ xài giá trị mặc định).
- `null` khác hoàn toàn với chuỗi rỗng `""` hay số `0`.

## 4. Biểu thức (Expressions) và Hàm (Functions)

### Toán tử ba ngôi (Ternary Operator)
Cú pháp: `điều_kiện ? giá_trị_đúng : giá_trị_sai`
Thường dùng để bật/tắt cấu hình tuỳ môi trường.

```bash
$ echo '1 == 1 ? "yes" : "no"' | terraform console
"yes"
```

### Nội suy chuỗi (String Interpolation)
Nhúng biểu thức vào trong chuỗi bằng cú pháp `${...}`

```bash
$ echo '"web-${1 + 1}"' | terraform console
"web-2"
```
*(Thực tế hay dùng: `"${var.env}-web"`)*

### Hàm dựng sẵn (Built-in Functions)
Terraform có hàng trăm hàm dựng sẵn cho chuỗi, mạng, tính toán... **Bạn không thể tự viết hàm mới.**

```shell
$ echo 'length(["a", "b", "c"])' | terraform console
3
$ echo 'tostring(42)' | terraform console
"42"
$ echo 'cidrsubnet("10.0.0.0/16", 8, 2)' | terraform console
"10.0.2.0/24"
```
*Giải thích hàm `cidrsubnet`: Cắt dải mạng `/16`, thêm 8 bit thành dải `/24`, và lấy dải subnet thứ 2.*

## 5. Block `terraform {}` khai báo những gì?

Block `terraform {}` không mô tả hạ tầng. Nó thiết lập cách Terraform chạy cấu hình. Trong đó có 6 thành phần, nhưng bạn chỉ cần nhớ 3 thành phần chính:
- **`required_version`**: Phiên bản Terraform CLI nào được phép chạy code này.
- **`required_providers`**: Các provider plugin cần tải.
- **`backend`**: Nơi dùng để cất file state (mặc định ở local, thực tế hay dùng S3).

## 6. Thứ tự dòng không quan trọng

Trong HCL, thứ tự bạn khai báo các block từ trên xuống dưới **không quyết định thứ tự thực thi**. 
- Bạn có thể khai báo một S3 bucket sau một Security Group có dùng bucket đó, kết quả vẫn không đổi.
- Lý do: Terraform là ngôn ngữ **Khai báo (Declarative)**. Khi chạy, nó sẽ đọc toàn bộ file và tự dựng lên một **Đồ thị phụ thuộc (Dependency Graph)** từ các biến được tham chiếu (VD: `aws_s3_bucket.first.arn`), sau đó mới quyết định sẽ tạo cái gì trước, cái gì sau. 

---

## 💡 Tóm tắt 4 dòng cần thuộc:
1. HCL chỉ có 2 khối cấu trúc: **Argument** (`key = value`) và **Block** (`type "label" { body }`).
2. Có 6 kiểu giá trị: `string`, `number`, `bool`, `list/tuple`, `map/object`, `null` (mang nghĩa bỏ qua).
3. Nội suy dùng `${...}`, kiểm tra hàm/biểu thức nhanh bằng `terraform console`.
4. Thứ tự viết code (dòng trên/dưới) không ảnh hưởng đến thứ tự tạo tài nguyên, Terraform tự lo tính phụ thuộc (Dependency Graph).
