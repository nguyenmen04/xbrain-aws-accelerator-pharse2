# Bài 5: Đồ Thị Phụ Thuộc - Implicit, `depends_on`, và `-target`

Dưới đây là tóm tắt những kiến thức cốt lõi về **Đồ thị phụ thuộc (Dependency Graph)** trong Terraform. Toàn bộ code và ví dụ lệnh được giữ nguyên để tiện tra cứu.

## 1. Phụ thuộc ngầm (Implicit Dependency): Tham chiếu là tất cả

Thứ tự khai báo trên/dưới trong file `.tf` không có ý nghĩa gì với Terraform. Nó quyết định thứ tự chạy bằng cách nhìn vào **tham chiếu chéo** giữa các tài nguyên.

Ví dụ tạo bucket và bật versioning cho bucket đó:

```hcl
resource "aws_s3_bucket" "data" {
  bucket_prefix = "tf-series-bai5-"
  force_destroy = true
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

Ở đây `aws_s3_bucket_versioning` gọi thuộc tính `aws_s3_bucket.data.id`. Việc gọi này tạo ra một cạnh phụ thuộc: Terraform tự hiểu **"versioning cần bucket, phải tạo bucket trước"**. 

Kết quả khi chạy `apply`:
```yaml
$ terraform apply -auto-approve
aws_s3_bucket.data: Creating...
aws_s3_bucket.data: Creation complete after 3s [id=tf-series-bai5-20260525025940839300000001]
aws_s3_bucket_versioning.data: Creating...
aws_s3_bucket_versioning.data: Creation complete after 2s [id=...]
Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

## 2. Xem đồ thị tận mắt với `terraform graph`

Lệnh `terraform graph` xuất đồ thị định dạng DOT.

```ruby
$ terraform graph
digraph G {
  rankdir = "RL";
  node [shape = rect, fontname = "sans-serif"];
  "aws_s3_bucket.data" [label="aws_s3_bucket.data"];
  "aws_s3_bucket_versioning.data" [label="aws_s3_bucket_versioning.data"];
  "aws_s3_bucket_versioning.data" -> "aws_s3_bucket.data";
}
```

*(Mẹo: Bạn có thể vẽ ra ảnh bằng Graphviz: `terraform graph | dot -Tpng > graph.png`)*

Từ đồ thị này, Terraform sắp xếp topo (topological sort):
- Đỉnh không phụ thuộc ai thì làm trước.
- Đỉnh có phụ thuộc thì chờ đỉnh kia làm xong mới làm.
- Các tài nguyên không có cạnh nối với nhau là **độc lập**, Terraform sẽ **tạo chúng song song** (mặc định 10 thao tác đồng thời) giúp tốc độ apply cực nhanh.

## 3. Vì sao Destroy lại đảo ngược thứ tự?

Quy luật: Tạo thuận chiều, Xóa ngược chiều. 
Nếu B phụ thuộc A thì không thể xóa A trước khi B bị gỡ. Khi gõ `destroy`, quá trình diễn ra ngược lại hoàn toàn so với lúc `apply`.

```r
   apply: thuận theo cạnh           destroy: ngược cạnh
   ───────────────────────          ─────────────────────────
   1. aws_s3_bucket.data            1. aws_s3_bucket_versioning.data
            │ (xong trước)                   │ (gỡ trước)
            ▼                                ▼
   2. aws_s3_bucket_versioning.data  2. aws_s3_bucket.data
```

Kết quả console lúc xóa:
```yaml
$ terraform destroy -auto-approve
aws_s3_bucket_versioning.data: Destroying...
aws_s3_bucket_versioning.data: Destruction complete after 1s
aws_s3_bucket.data: Destroying...
aws_s3_bucket.data: Destruction complete after 0s
Destroy complete! Resources: 2 destroyed.
```

## 4. Khi tham chiếu ẩn (Tầng ứng dụng): `depends_on`

Đôi khi quan hệ phụ thuộc không được thể hiện qua tham chiếu giá trị biến. VD: Một EC2 instance cần một IAM role policy đã sẵn sàng để ứng dụng bên trong hoạt động, nhưng EC2 không gọi thuộc tính nào của policy cả. Lúc đó ta phải dùng `depends_on`:

```hcl
resource "aws_instance" "app" {
  # ... không có dòng nào tham chiếu policy ...
  depends_on = [aws_iam_role_policy.app]
}
```

**⚠️ Khuyến cáo mạnh từ HashiCorp:**
Chỉ dùng `depends_on` như giải pháp cuối cùng. Việc dùng nó khiến Terraform lập kế hoạch "dè dặt" hơn, dễ dẫn tới việc xóa/tạo lại tài nguyên nhiều hơn mức cần thiết. Luôn ưu tiên dùng tham chiếu ngầm (Implicit Dependency) nếu có thể.

## 5. `-target`: Lối thoát hiểm (Không dùng hằng ngày)

Cờ `-target` dùng để ra lệnh cho Terraform chỉ focus `apply` (hoặc `destroy`) vào một vài resource cụ thể mà bỏ qua toàn bộ phần còn lại của đồ thị. (Vẫn áp dụng đúng luật phụ thuộc với những thứ nó cần).

```bash
terraform apply -target=aws_s3_bucket.data
```

**⚠️ Cảnh báo:** Lệnh này làm cấu hình và state bị lệch nhau một cách có chủ đích. Nó là "lối thoát hiểm" để fix lỗi nóng hoặc gỡ các resource cứng đầu. Đừng bao giờ lạm dụng gõ `-target` liên tục cho "nhanh", nếu cảm thấy hạ tầng quá lớn, hãy tách State (Module/Workspace).

---

## 💡 Tóm tắt 4 điều cần nhớ:
1. **Implicit Dependency:** Tham chiếu thuộc tính của tài nguyên khác sẽ tự động tạo ra thứ tự tạo/xóa tài nguyên.
2. Terraform tạo đồ thị có hướng (sắp xếp Topo), những resource không liên quan nhau sẽ được chạy **song song**.
3. **`depends_on`:** Khai báo phụ thuộc thủ công cho những tham chiếu "ẩn" ở tầng ứng dụng, hạn chế dùng tối đa.
4. **`-target`:** Lối thoát hiểm nhắm vào resource cụ thể, không được dùng làm thói quen hàng ngày.
