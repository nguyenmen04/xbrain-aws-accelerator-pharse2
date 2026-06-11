# Bài 1: Infrastructure as Code, Terraform Là Gì, và Làm Quen CLI

## 1. Infrastructure as Code (IaC) là gì?

IaC = Viết hạ tầng bằng code thay vì bấm tay trên Console.

Thay vì:
- Tự tạo EC2
- Tự tạo S3
- Tự tạo VPC
- Tự cấu hình Security Group

=> Viết thành file `.tf`

**Lợi ích:**
✅ Tái sử dụng được
✅ Có thể lưu Git
✅ Review bằng Pull Request
✅ Triển khai giống nhau ở Dev/Test/Staging/Production
✅ Phát hiện được Drift

## 2. Terraform là gì?

Terraform là công cụ IaC của HashiCorp.

Nó cho phép mô tả:
- AWS
- Azure
- GCP
- Kubernetes
- GitHub
- Cloudflare

bằng code.

Terraform thuộc kiểu: **Declarative (khai báo)**

Bạn chỉ nói: *"Tôi muốn có 1 EC2"*

Terraform tự tính:
- Tạo cái gì trước
- Tạo cái gì sau
- API nào cần gọi

## 3. 3 thành phần quan trọng nhất

### Provider
Plugin giúp Terraform giao tiếp với nền tảng.

Ví dụ:
```hcl
provider "aws" {
  region = "ap-southeast-1"
}
```
Provider AWS biết:
- EC2 là gì
- S3 là gì
- IAM là gì

### Resource
Là đối tượng hạ tầng.

Ví dụ:
```hcl
resource "aws_s3_bucket" "demo" {
  bucket = "my-bucket"
}
```
Resource có thể là:
- EC2
- S3
- VPC
- IAM Role
- Security Group

### State
File: `terraform.tfstate`

Terraform dùng để ghi nhớ:
- Resource nào đã tạo
- ID thực tế là gì
- Thuộc tính hiện tại là gì

State là thứ cực kỳ quan trọng. Nếu mất state:
- Terraform không biết đã tạo gì
- Có thể tạo trùng tài nguyên

## 4. Workflow Terraform

Đây là thứ sẽ dùng mỗi ngày.

- **Bước 1: Write**
Viết file `.tf`
`resource ...`

- **Bước 2: Plan**
`terraform plan`
Terraform sẽ so sánh Config, State, và Cloud thực tế rồi báo:
  - `+` create
  - `~` update
  - `-` destroy
Chưa thay đổi gì cả.

- **Bước 3: Apply**
`terraform apply`
Terraform thực sự gọi API (Ví dụ: Create S3 Bucket, Create EC2, Create IAM Role).

- **Bước 4: Destroy**
`terraform destroy`
Xóa toàn bộ hạ tầng Terraform quản lý.

## 5. Terraform Core và Provider khác nhau

*(Rất hay hỏi trong phỏng vấn)*

**Terraform Core**
Làm nhiệm vụ:
- Đọc file `.tf`
- Đọc state
- Tính diff
- Tạo execution plan
- Quản lý dependency graph

Core KHÔNG biết AWS là gì.

**AWS Provider**
Biết: EC2 API, S3 API, IAM API
Provider sẽ gọi API AWS thật.

**Luồng hoạt động:**
Terraform Core → AWS Provider → AWS API → AWS Resources

## 6. Terraform không tạo tài nguyên trực tiếp

Terraform → AWS Provider → AWS SDK → AWS API → EC2/S3/IAM

Đây là lý do Terraform có thể quản lý hàng nghìn dịch vụ khác nhau.

## 7. Dependency Graph

Terraform tự xây dựng đồ thị phụ thuộc.

Ví dụ:
VPC → Subnet → EC2

Bạn không cần viết: "create VPC, create subnet, create EC2". Terraform tự suy luận.

## 8. Drift là gì?

Drift xảy ra khi:
Terraform nghĩ: `EC2 type = t3.micro`
Nhưng ai đó vào AWS Console sửa thành: `t3.medium`

Lúc này: **Config ≠ Reality**

Terraform sẽ phát hiện drift trong lần plan tiếp theo.

## 9. Những lệnh CLI phải thuộc lòng

- Khởi tạo project: `terraform init` (Tải Provider, Module, tạo thư mục `.terraform/`)
- Kiểm tra cú pháp: `terraform validate`
- Xem kế hoạch: `terraform plan`
- Thực thực thi: `terraform apply`
- Xóa hạ tầng: `terraform destroy`
- Format code: `terraform fmt`
- Xem provider: `terraform providers`
- Xem state: `terraform state`
- Mở console HCL: `terraform console`

## 10. Thứ tự học trong series này

Theo mình, nên ghi nhớ lộ trình:

1. **Part 1:** IaC, Terraform, Provider, Resource, State, Plan/Apply
2. **Part 2-5:** Provider, Resource, Dependency Graph, Variables, Outputs
3. **Part 6-10:** State, Remote State, Backend, Locking
4. **Part 11-15:** Modules, Workspaces, Import
5. **Part 16-20:** Production, CI/CD, Testing, Best Practices

---

## Tóm tắt 10 dòng cần thuộc

1. Terraform là công cụ IaC dạng khai báo (Declarative).
2. File `.tf` là nguồn sự thật (Source of Truth).
3. Provider dùng để giao tiếp với AWS/GCP/Azure.
4. Resource là đơn vị hạ tầng Terraform quản lý.
5. State (`terraform.tfstate`) ghi nhớ trạng thái thực tế.
6. Terraform Core không biết AWS là gì.
7. AWS Provider mới là thứ gọi AWS API.
8. Workflow chuẩn: `init` → `validate` → `plan` → `apply` → `destroy`.
9. `plan` không thay đổi gì, chỉ xem trước.
10. Trong Terraform, hiểu Provider + Resource + State là đã nắm được khoảng 70% nền tảng.

---

## Các khái niệm mở rộng (Ghi chú bổ sung)

### 1. Drift là gì?
Drift (Configuration Drift) là tình trạng hạ tầng thực tế khác với hạ tầng được mô tả trong Terraform.
Ví dụ: Terraform khai báo `instance_type = "t3.micro"` nhưng ai đó vào Console sửa thành `t3.medium`.
Kết quả: Terraform Config ≠ AWS Reality => Drift.
Terraform sẽ phát hiện Drift khi chạy: `terraform plan`

### 2. HashiCorp là gì?
HashiCorp là công ty tạo ra: Terraform, Vault, Consul, Nomad, Packer, Vagrant.
Terraform chính là sản phẩm nổi tiếng nhất của HashiCorp.

### 3. Azure là gì?
Microsoft Azure là nền tảng Cloud của Microsoft.
Tương tự:
- AWS (Amazon)
- Azure (Microsoft)
- GCP (Google)
Terraform có thể quản lý Azure thông qua Azure Provider.

### 4. Kubernetes là gì?
Kubernetes là hệ thống quản lý container.
Giúp: Deploy, Scale, Load Balance, Auto Healing.
Terraform có thể tạo: Cluster, Node Pool, Namespace trên Kubernetes.

### 5. Cloudflare là gì?
Cloudflare cung cấp: DNS, CDN, WAF, DDoS Protection, SSL.
Terraform có Cloudflare Provider để quản lý bằng code.

### 6. Provider là gì?
Provider là cầu nối giữa Terraform và nền tảng bên ngoài.
Ví dụ: `provider "aws" {}`
AWS Provider biết EC2 API, S3 API, IAM API (Terraform Core không biết những thứ này).

### 7. Plugin là gì?
Plugin là một chương trình mở rộng chức năng.
Trong Terraform: AWS Provider, Azure Provider, Cloudflare Provider đều là các Plugin.
Khi chạy `terraform init`, Terraform sẽ tải plugin về thư mục `.terraform/`.

### 8. Diff là gì?
Diff = Difference. Là phần khác nhau giữa Terraform Config và Current State.
Ví dụ: Hiện tại `t3.micro`, bạn sửa thành `t3.small`. Terraform tính toán sự thay đổi. Đó chính là Diff.

### 9. State là gì?
State là file `terraform.tfstate`.
Chứa: Resource ID, ARN, IP Address, Metadata.
Terraform dùng State để nhớ: "Tôi đã tạo những gì rồi?".

### 10. Dependency Graph là gì?
Dependency Graph là sơ đồ phụ thuộc giữa các Resource.
Ví dụ: EC2 không thể tạo trước Subnet, Subnet không thể tạo trước VPC.
Terraform tự tạo Graph này.

---

## Luồng hoạt động Terraform (Bằng lời và Sơ đồ)

### Luồng hoạt động (bằng lời)
- **Bước 1:** Developer viết `main.tf` mô tả hạ tầng mong muốn.
- **Bước 2:** Terraform đọc Config (`.tf`).
- **Bước 3:** Terraform đọc `terraform.tfstate` để biết hiện tại có gì.
- **Bước 4:** Terraform hỏi Cloud Provider xem thực tế đang tồn tại gì.
- **Bước 5:** Terraform tính Desired State vs Current State.
- **Bước 6:** Sinh ra Diff.
- **Bước 7:** Terraform tạo Execution Plan.
- **Bước 8:** Khi chạy `terraform apply`, Terraform gửi lệnh cho Provider.
- **Bước 9:** Provider gọi API thật (RunInstances, CreateBucket, etc.).
- **Bước 10:** Cloud tạo Resource.
- **Bước 11:** Terraform cập nhật State mới.

### Sơ đồ tổng quát
```text
                main.tf
                   │
                   ▼
         ┌─────────────────┐
         │ Terraform Core  │
         └─────────────────┘
                   │
                   │ đọc
                   ▼
          terraform.tfstate
                   │
                   │ so sánh
                   ▼
             Current Reality
                   │
                   ▼
               Diff
                   │
                   ▼
               Plan
                   │
                   ▼
               Apply
                   │
                   ▼
         ┌─────────────────┐
         │    Provider     │
         │  (AWS/Azure)    │
         └─────────────────┘
                   │
                   ▼
              Cloud API
                   │
                   ▼
              Resources
         (EC2, S3, VPC...)
                   │
                   ▼
         Update terraform.tfstate
```

### Công thức nhớ nhanh
```text
Config (.tf) + State (.tfstate) + Provider
      ↓
Terraform Plan
      ↓
Terraform Apply
      ↓
Cloud Resources
      ↓
Update State
```

> **Ghi nhớ:** Chỉ cần hiểu chắc `Config → State → Diff → Plan → Apply → Provider → Cloud Resource` là đã nắm được luồng hoạt động cốt lõi của Terraform.
