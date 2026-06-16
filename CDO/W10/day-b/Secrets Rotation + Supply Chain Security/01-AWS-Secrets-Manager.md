# Bài 1: AWS Secrets Manager

> **Nguồn tham khảo:** https://docs.aws.amazon.com/secretsmanager

---

## 1. AWS Secrets Manager là gì?

**AWS Secrets Manager** là dịch vụ quản lý vòng đời của **secrets** (bí mật) một cách tập trung — giúp bạn lưu trữ, truy xuất, và **tự động rotate** các thông tin nhạy cảm.

### Secrets là gì?
Bất kỳ thông tin nào cần giữ bí mật:
- 🔑 Database credentials (username/password)
- 🔐 API keys
- 🔒 OAuth tokens
- 📜 TLS certificates
- 🗝️ SSH keys

### Tại sao cần Secrets Manager?

| Cách cũ (không tốt) | Cách mới (Secrets Manager) |
|---|---|
| Hard-code password trong code | Lưu tập trung trong Secrets Manager |
| Đặt trong file `.env` trên server | Truy xuất runtime qua API |
| Không bao giờ đổi password | **Tự động rotate** theo lịch |
| Ai cũng truy cập được | Kiểm soát quyền bằng IAM |
| Không biết ai đã xem | **Audit trail** qua CloudTrail |

---

## 2. Kiến trúc hoạt động

```
┌──────────────┐         ┌─────────────────────┐
│  Application │──API──► │  AWS Secrets Manager │
│  (ECS, EKS,  │         │                      │
│   Lambda...) │◄─────── │  ┌────────────────┐  │
└──────────────┘  secret │  │ Secret Value   │  │
                  value  │  │ (encrypted     │  │
                         │  │  with KMS)     │  │
                         │  └────────────────┘  │
                         │                      │
                         │  ┌────────────────┐  │
                         │  │ Rotation       │  │
                         │  │ (Lambda func.) │  │
                         │  └────────────────┘  │
                         └─────────────────────┘
                                   │
                              ┌────┴────┐
                              │ AWS KMS │  (mã hóa)
                              └─────────┘
```

---

## 3. Các khái niệm cốt lõi

### 3.1. Secret

Một **secret** bao gồm:
- **Secret name** — Tên duy nhất (ví dụ: `prod/myapp/db-password`)
- **Secret value** — Giá trị bí mật (string hoặc JSON)
- **Metadata** — Tags, description, rotation config
- **Versions** — Lịch sử các phiên bản giá trị

```json
// Ví dụ secret value dạng JSON
{
  "username": "admin",
  "password": "MyS3cur3P@ss!",
  "engine": "mysql",
  "host": "mydb.abc123.us-east-1.rds.amazonaws.com",
  "port": 3306,
  "dbname": "production"
}
```

### 3.2. Version Stages (Nhãn phiên bản)

| Stage | Ý nghĩa |
|-------|---------|
| `AWSCURRENT` | Phiên bản **đang được sử dụng** |
| `AWSPENDING` | Phiên bản **mới đang được rotate** |
| `AWSPREVIOUS` | Phiên bản **trước đó** (backup) |

### 3.3. Rotation (Xoay vòng tự động)

Secrets Manager có thể **tự động thay đổi** password theo lịch:

```
Ngày 1: password = "abc123"   (AWSCURRENT)
         │
    Rotation chạy (Lambda function)
         │
Ngày 30: password = "xyz789"  (AWSCURRENT)
         password = "abc123"  (AWSPREVIOUS)
```

---

## 4. Cách tạo Secret

### 4.1. Qua AWS Console

1. Mở **AWS Secrets Manager** console
2. Click **Store a new secret**
3. Chọn loại: **Credentials for Amazon RDS database** hoặc **Other type of secret**
4. Nhập key-value pairs
5. Đặt tên (ví dụ: `prod/myapp/db-creds`)
6. (Tùy chọn) Cấu hình rotation
7. Click **Store**

### 4.2. Qua AWS CLI

```bash
# Tạo secret
aws secretsmanager create-secret \
  --name "prod/myapp/db-creds" \
  --description "Database credentials for myapp" \
  --secret-string '{"username":"admin","password":"MyP@ssw0rd"}'

# Lấy giá trị secret
aws secretsmanager get-secret-value \
  --secret-id "prod/myapp/db-creds"

# Cập nhật secret
aws secretsmanager update-secret \
  --secret-id "prod/myapp/db-creds" \
  --secret-string '{"username":"admin","password":"NewP@ss123"}'

# Xóa secret (có 30 ngày recovery)
aws secretsmanager delete-secret \
  --secret-id "prod/myapp/db-creds" \
  --recovery-window-in-days 7
```

---

## 5. Rotation — Xoay vòng tự động

### Cách hoạt động:

```
1. Schedule trigger (mỗi 30 ngày)
        │
        ▼
2. Lambda function được gọi
        │
        ▼
3. Lambda tạo password MỚI
        │
        ▼
4. Lambda cập nhật password trên database
        │
        ▼
5. Lambda lưu password mới vào Secrets Manager
        │
        ▼
6. Lần sau app lấy secret → nhận password mới
```

### Cấu hình rotation:

```bash
# Bật rotation tự động (mỗi 30 ngày)
aws secretsmanager rotate-secret \
  --secret-id "prod/myapp/db-creds" \
  --rotation-lambda-arn "arn:aws:lambda:us-east-1:123456789:function:SecretsManagerRotation" \
  --rotation-rules '{"AutomaticallyAfterDays": 30}'
```

---

## 6. Truy xuất Secret trong ứng dụng

### Python (Boto3)

```python
import boto3
import json

client = boto3.client('secretsmanager')

response = client.get_secret_value(
    SecretId='prod/myapp/db-creds'
)

secret = json.loads(response['SecretString'])
db_username = secret['username']
db_password = secret['password']
```

### Trong EKS (Kubernetes)

Dùng kết hợp với **External Secrets Operator** (bài sau) để tự động sync secret từ AWS Secrets Manager vào Kubernetes Secret.

---

## 7. Bảo mật & IAM

### IAM Policy mẫu — Cho phép app đọc secret:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789:secret:prod/myapp/*"
    }
  ]
}
```

### Resource-based Policy — Giới hạn ai được truy cập secret:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalAccount": "123456789012"
        }
      }
    }
  ]
}
```

---

## 8. Chi phí

| Hạng mục | Giá |
|----------|-----|
| Mỗi secret | ~$0.40/tháng |
| Mỗi 10,000 API calls | ~$0.05 |
| Rotation Lambda | Tính riêng theo Lambda pricing |

---

## 9. Tóm tắt

| Khái niệm | Vai trò |
|-----------|---------|
| **Secret** | Thông tin nhạy cảm (password, API key…) |
| **Version Stages** | AWSCURRENT / AWSPENDING / AWSPREVIOUS |
| **Rotation** | Tự động đổi password theo lịch (Lambda) |
| **KMS** | Mã hóa secret at rest |
| **IAM** | Kiểm soát ai được truy cập secret nào |

> **Ghi nhớ:** AWS Secrets Manager giải quyết bài toán **"làm sao lưu password an toàn và tự động đổi định kỳ"**. Không bao giờ hard-code secrets trong code — luôn lấy runtime từ Secrets Manager.
