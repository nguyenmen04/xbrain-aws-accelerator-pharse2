# Bài 4: ValidatingAdmissionPolicy — Policy native trong Kubernetes

> **Nguồn tham khảo:** https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy

---

## 1. ValidatingAdmissionPolicy là gì?

**ValidatingAdmissionPolicy** là tính năng **native (tích hợp sẵn)** trong Kubernetes (GA từ v1.30) cho phép bạn viết admission policies **mà không cần webhook bên ngoài** như Gatekeeper hay Kyverno.

### Đặc điểm chính:
- **In-process:** Chạy ngay trong API Server, không cần service bên ngoài
- **Dùng CEL (Common Expression Language):** Thay vì Rego hay YAML
- **Không cần cài đặt thêm:** Đã có sẵn trong K8s 1.30+
- **Hiệu suất cao:** Không có network call tới webhook
- **Declarative:** Khai báo policy bằng Kubernetes YAML

### So sánh với Admission Webhook truyền thống:

| | Admission Webhook | ValidatingAdmissionPolicy |
|---|---|---|
| **Triển khai** | Cần deploy service riêng | Native trong API Server |
| **Ngôn ngữ** | Bất kỳ (Go, Python…) | CEL |
| **Độ phức tạp** | Cao (phải viết HTTP server) | Thấp (chỉ cần YAML) |
| **Hiệu suất** | Network call → latency | In-process → nhanh hơn |
| **Maintenance** | Phải maintain webhook service | Không cần |

---

## 2. Ba thành phần chính

### 2.1. ValidatingAdmissionPolicy — Định nghĩa policy

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "demo-policy.example.com"
spec:
  failurePolicy: Fail               # Fail hoặc Ignore nếu CEL error
  matchConstraints:
    resourceRules:
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["deployments"]
  validations:
    - expression: "object.spec.replicas <= 5"
      message: "Deployment không được có quá 5 replicas"
```

### 2.2. ValidatingAdmissionPolicyBinding — Gán policy cho cluster

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "demo-binding-test.example.com"
spec:
  policyName: "demo-policy.example.com"
  validationActions: [Deny]          # Deny, Warn, hoặc Audit
  matchResources:
    namespaceSelector:
      matchLabels:
        environment: test             # Chỉ áp dụng cho namespace có label này
```

### 2.3. Parameter Resources (Tùy chọn) — Tham số hóa policy

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: "replica-limit-params"
  namespace: default
data:
  maxReplicas: "10"
```

---

## 3. CEL (Common Expression Language) — Ngôn ngữ viết Policy

CEL là ngôn ngữ **biểu thức nhẹ, an toàn, nhanh** được Google phát triển.

### 3.1. Các biến có sẵn trong CEL

| Biến | Mô tả |
|------|-------|
| `object` | Resource đang được tạo/cập nhật |
| `oldObject` | Resource trước khi cập nhật (chỉ có với UPDATE) |
| `request` | Thông tin admission request (user, operation…) |
| `params` | Tham số từ parameter resource |
| `namespaceObject` | Namespace chứa resource |
| `authorizer` | Kiểm tra quyền RBAC |

### 3.2. Ví dụ CEL expressions

```cel
# Kiểm tra label tồn tại
object.metadata.labels.?team.hasValue()

# Kiểm tra container image không dùng tag 'latest'
object.spec.containers.all(c, !c.image.endsWith(":latest"))

# Kiểm tra resource limits được set
object.spec.containers.all(c, c.resources.?limits.?memory.hasValue())

# Kiểm tra annotation có format hợp lệ
object.metadata.annotations["contact-email"].matches("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$")

# So sánh old và new object (chỉ với UPDATE)
object.spec.replicas <= oldObject.spec.replicas  // không cho tăng replicas
```

---

## 4. Ví dụ thực tế

### 4.1. Cấm dùng image tag "latest"

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "disallow-latest-tag"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["deployments"]
  validations:
    - expression: >
        object.spec.template.spec.containers.all(c,
          !c.image.endsWith(':latest') && c.image.contains(':')
        )
      message: "Tất cả container phải dùng image tag cụ thể, không được dùng 'latest' hoặc bỏ trống tag"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "disallow-latest-tag-binding"
spec:
  policyName: "disallow-latest-tag"
  validationActions: [Deny]
```

### 4.2. Bắt buộc resource requests và limits

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "require-resource-limits"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["deployments"]
  validations:
    - expression: >
        object.spec.template.spec.containers.all(c,
          c.resources.?limits.?cpu.hasValue() &&
          c.resources.?limits.?memory.hasValue()
        )
      message: "Tất cả container phải có resource limits (cpu và memory)"
```

### 4.3. Policy có tham số (parameterized)

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "check-replicas"
spec:
  failurePolicy: Fail
  paramKind:
    apiVersion: v1
    kind: ConfigMap
  matchConstraints:
    resourceRules:
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["deployments"]
  validations:
    - expression: "object.spec.replicas <= int(params.data.maxReplicas)"
      messageExpression: "'Replicas vượt quá giới hạn ' + params.data.maxReplicas"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "check-replicas-binding"
spec:
  policyName: "check-replicas"
  validationActions: [Deny]
  paramRef:
    name: "replica-limit-params"
    namespace: default
```

---

## 5. Validation Actions (Hành động khi vi phạm)

| Action | Mô tả |
|--------|-------|
| `Deny` | **Từ chối** request, trả về error |
| `Warn` | Trả về **warning** nhưng vẫn cho phép |
| `Audit` | Ghi **audit log** nhưng không chặn |

> **Tip:** Có thể kết hợp nhiều actions: `[Warn, Audit]` — vừa cảnh báo vừa ghi log.

---

## 6. Failure Policy

Khi CEL expression bị lỗi (ví dụ: truy cập field không tồn tại):

| Policy | Mô tả |
|--------|-------|
| `Fail` | Từ chối request (an toàn hơn) |
| `Ignore` | Bỏ qua policy bị lỗi (ít gián đoạn hơn) |

---

## 7. So sánh với Gatekeeper

| | ValidatingAdmissionPolicy | Gatekeeper |
|---|---|---|
| **Ngôn ngữ** | CEL | Rego |
| **Cài đặt** | Không cần (native K8s 1.30+) | Cần cài thêm |
| **Mutation** | ❌ Chưa hỗ trợ | ✅ Hỗ trợ |
| **Audit resource có sẵn** | ❌ Không | ✅ Có |
| **Thư viện policy sẵn** | ❌ Ít | ✅ Phong phú |
| **Phức tạp** | Đơn giản (CEL expressions) | Phức tạp hơn (Rego full language) |
| **Hiệu suất** | Cao (in-process) | Thấp hơn (webhook call) |
| **Use case** | Validation đơn giản → trung bình | Policy phức tạp, mutation, audit |

---

## 8. Khi nào dùng ValidatingAdmissionPolicy?

✅ **Nên dùng khi:**
- Cluster đang chạy K8s 1.30+
- Policy validation đơn giản (kiểm tra labels, limits, image tags…)
- Muốn giảm dependency vào third-party tools
- Cần hiệu suất cao, không muốn thêm webhook latency

❌ **Không nên dùng khi:**
- Cần **mutation** (tự động sửa resource)
- Cần **audit** resource đã tồn tại trong cluster
- Policy quá phức tạp mà CEL không đáp ứng được
- Cluster chạy K8s < 1.30

---

## 9. Tóm tắt

| Thành phần | Vai trò |
|-----------|---------|
| **ValidatingAdmissionPolicy** | Định nghĩa policy + CEL expressions |
| **ValidatingAdmissionPolicyBinding** | Gán policy cho resources/namespaces cụ thể |
| **Parameter Resource** | Tham số hóa policy (ConfigMap, CRD…) |
| **CEL** | Ngôn ngữ biểu thức để viết validation rules |

> **Ghi nhớ:** ValidatingAdmissionPolicy là lựa chọn **nhẹ, nhanh, native** cho các validation policy đơn giản đến trung bình. Nếu cần policy phức tạp hơn hoặc mutation, hãy dùng Gatekeeper hoặc Kyverno.
