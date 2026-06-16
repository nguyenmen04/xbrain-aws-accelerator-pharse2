# Bài 2: Open Policy Agent (OPA) & Ngôn ngữ Rego

> **Nguồn tham khảo:** https://www.openpolicyagent.org/docs

---

## 1. OPA là gì?

**OPA (Open Policy Agent)** — đọc là "oh-pa" — là một **policy engine mã nguồn mở, đa mục đích** dùng để đưa ra quyết định chính sách (policy decisions) trong toàn bộ stack công nghệ.

### Đặc điểm chính:
- **General-purpose:** Không chỉ dùng cho Kubernetes — dùng được cho API authorization, Terraform, CI/CD, microservices…
- **Decoupled (tách biệt):** Policy được tách ra khỏi code ứng dụng
- **Declarative:** Viết policy bằng ngôn ngữ khai báo (Rego) thay vì imperative code
- **Lightweight:** Chạy như một process đơn lẻ, không cần database

---

## 2. Kiến trúc hoạt động của OPA

```
┌─────────────────────────────────────────────────┐
│                    Ứng dụng / Service            │
│                                                  │
│  1. Nhận request ──► 2. Gửi query tới OPA        │
│                           │                      │
│                           ▼                      │
│                   ┌──────────────┐               │
│                   │     OPA      │               │
│                   │              │               │
│                   │  ┌────────┐  │               │
│                   │  │ Policy │  │  (Rego)       │
│                   │  └────────┘  │               │
│                   │  ┌────────┐  │               │
│                   │  │  Data  │  │  (JSON)       │
│                   │  └────────┘  │               │
│                   └──────┬───────┘               │
│                          │                       │
│  4. Thực thi quyết định ◄─ 3. Trả về decision   │
└─────────────────────────────────────────────────┘
```

### Luồng hoạt động:
1. Service nhận request từ client
2. Service gửi **query** (kèm input data) tới OPA
3. OPA đánh giá query dựa trên **Policy + Data** → trả về **decision** (allow/deny, hoặc dữ liệu bất kỳ)
4. Service thực thi theo quyết định từ OPA

---

## 3. Rego — Ngôn ngữ viết Policy

**Rego** (đọc là "ray-go") là ngôn ngữ **khai báo (declarative)** được thiết kế riêng cho OPA để viết policy.

### 3.1. Đặc điểm của Rego
- Lấy cảm hứng từ **Datalog** (ngôn ngữ query)
- Hỗ trợ **JSON/YAML** data natively
- Có **built-in functions** phong phú (string, regex, JWT, HTTP…)
- **Immutable** — không có biến thay đổi giá trị

### 3.2. Cấu trúc cơ bản

```rego
# Package khai báo namespace cho policy
package authz

# Import dữ liệu
import rego.v1

# Rule mặc định — deny tất cả
default allow := false

# Rule cho phép — chỉ allow nếu TẤT CẢ điều kiện đều đúng
allow if {
    input.method == "GET"
    input.path == ["api", "public"]
}

# Rule cho phép user admin
allow if {
    input.user == "admin"
}
```

### 3.3. Giải thích các khái niệm Rego

| Khái niệm | Mô tả | Ví dụ |
|-----------|-------|-------|
| **Package** | Namespace cho policy | `package authz` |
| **Rule** | Khai báo một quyết định | `allow if { ... }` |
| **Input** | Dữ liệu đầu vào (từ query) | `input.method` |
| **Data** | Dữ liệu tĩnh được load sẵn | `data.roles` |
| **Default** | Giá trị mặc định nếu không rule nào match | `default allow := false` |

### 3.4. Ví dụ thực tế: API Authorization

```rego
package httpapi.authz

import rego.v1

# Mặc định: từ chối tất cả
default allow := false

# Cho phép GET cho tất cả user đã đăng nhập
allow if {
    input.method == "GET"
    input.user != ""
}

# Cho phép POST chỉ cho admin
allow if {
    input.method == "POST"
    input.user == "admin"
}

# Cho phép truy cập resource của chính mình
allow if {
    input.method == "GET"
    input.path = ["users", input.user]
}
```

**Input mẫu (JSON):**
```json
{
  "method": "GET",
  "path": ["users", "bob"],
  "user": "bob"
}
```

**Output:** `allow = true` (vì user bob truy cập resource của chính mình)

---

## 4. Các kiểu dữ liệu trong Rego

| Kiểu | Ví dụ | Mô tả |
|------|-------|-------|
| **String** | `"hello"` | Chuỗi ký tự |
| **Number** | `42`, `3.14` | Số |
| **Boolean** | `true`, `false` | Đúng/sai |
| **Null** | `null` | Giá trị rỗng |
| **Array** | `[1, 2, 3]` | Mảng có thứ tự |
| **Object** | `{"key": "value"}` | Key-value pairs |
| **Set** | `{1, 2, 3}` | Tập hợp không trùng lặp |

---

## 5. Các phép toán và hàm phổ biến

```rego
# So sánh
x == y          # bằng
x != y          # không bằng
x < y           # nhỏ hơn

# Chuỗi
contains(input.path, "admin")
startswith(input.path, "/api")
sprintf("user: %s", [input.user])

# Tập hợp
count(input.roles)
input.role in {"admin", "editor"}

# Duyệt mảng / object
some i
input.items[i].name == "target"
```

---

## 6. OPA dùng ở đâu? (Use Cases)

| Use Case | Mô tả |
|----------|-------|
| **Kubernetes Admission Control** | Kiểm tra request trước khi tạo resource (qua Gatekeeper) |
| **API Gateway Authorization** | Cho phép/từ chối request HTTP |
| **Terraform Planning** | Validate infrastructure changes trước khi apply |
| **Microservice Authorization** | Kiểm soát quyền giữa các service |
| **CI/CD Pipeline** | Enforce policy trong quy trình deploy |
| **Data Filtering** | Lọc dữ liệu trả về dựa trên quyền user |

---

## 7. Cách chạy OPA cơ bản

```bash
# Cài đặt OPA (macOS)
brew install opa

# Chạy OPA server
opa run --server

# Đánh giá policy từ file
opa eval -i input.json -d policy.rego "data.authz.allow"

# Test policy
opa test policy.rego policy_test.rego -v

# Chạy OPA Playground trực tuyến
# https://play.openpolicyagent.org/
```

---

## 8. Testing Policy trong OPA

OPA có hệ thống **testing tích hợp**:

```rego
# policy_test.rego
package authz

import rego.v1

test_allow_get_public if {
    allow with input as {
        "method": "GET",
        "path": ["api", "public"],
        "user": "alice"
    }
}

test_deny_post_non_admin if {
    not allow with input as {
        "method": "POST",
        "path": ["api", "data"],
        "user": "alice"
    }
}
```

---

## 9. Tóm tắt

| Thành phần | Vai trò |
|-----------|---------|
| **OPA** | Policy engine — nhận query, trả về decision |
| **Rego** | Ngôn ngữ viết policy (declarative) |
| **Input** | Dữ liệu đầu vào từ mỗi request |
| **Data** | Dữ liệu tĩnh (roles, permissions, config…) |
| **Decision** | Kết quả đánh giá (allow/deny hoặc structured data) |

> **Ghi nhớ:** OPA tách biệt **policy khỏi code**. Thay vì hard-code logic authorization trong ứng dụng, bạn viết policy bằng Rego và để OPA đánh giá → dễ maintain, audit, và thay đổi mà không cần deploy lại ứng dụng.
