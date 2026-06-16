# Bài 1: Kubernetes RBAC (Role-Based Access Control)

> **Nguồn tham khảo:** https://kubernetes.io/docs/reference/access-authn-authz/rbac

---

## 1. RBAC là gì?

**RBAC (Role-Based Access Control)** là phương pháp kiểm soát quyền truy cập tài nguyên trong Kubernetes dựa trên **vai trò (role)** của người dùng hoặc service account.

RBAC sử dụng API group `rbac.authorization.k8s.io` để đưa ra quyết định ủy quyền (authorization), cho phép admin cấu hình policy **một cách linh hoạt** thông qua Kubernetes API.

### Tại sao cần RBAC?
- **Nguyên tắc Least Privilege (đặc quyền tối thiểu):** Chỉ cấp đúng quyền cần thiết
- **Multi-tenancy:** Nhiều team/user dùng chung cluster mà không ảnh hưởng lẫn nhau
- **Audit & Compliance:** Theo dõi ai được phép làm gì

---

## 2. Bốn đối tượng chính trong RBAC

RBAC xoay quanh **4 loại Kubernetes object**:

### 2.1. Role (phạm vi namespace)

**Role** định nghĩa một tập hợp các quyền (permissions) **trong một namespace cụ thể**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]           # "" = core API group
  resources: ["pods"]        # Tài nguyên: pods
  verbs: ["get", "watch", "list"]  # Hành động được phép
```

### 2.2. ClusterRole (phạm vi toàn cluster)

**ClusterRole** giống Role nhưng áp dụng cho **toàn bộ cluster**, hoặc cho các tài nguyên không thuộc namespace nào (nodes, persistentvolumes…).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

> **Lưu ý:** ClusterRole cũng có thể dùng để cấp quyền cho tài nguyên trong namespace khi được binding với RoleBinding.

### 2.3. RoleBinding (gán Role cho user/group/SA trong namespace)

**RoleBinding** gán một Role (hoặc ClusterRole) cho **subjects** (user, group, service account) trong một namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 2.4. ClusterRoleBinding (gán ClusterRole cho toàn cluster)

**ClusterRoleBinding** gán ClusterRole cho subjects **trên toàn bộ cluster**.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: managers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 3. Sơ đồ quan hệ giữa các đối tượng RBAC

```
┌──────────────┐         ┌──────────────────┐         ┌──────────────┐
│   Subject     │◄────────│   RoleBinding     │────────►│    Role       │
│ (User/Group/  │         │ (namespace-scope) │         │ (namespace)  │
│  ServiceAcc.) │         └──────────────────┘         └──────────────┘
│               │
│               │         ┌──────────────────┐         ┌──────────────┐
│               │◄────────│ ClusterRoleBinding│────────►│ ClusterRole   │
│               │         │ (cluster-scope)   │         │ (cluster)    │
└──────────────┘         └──────────────────┘         └──────────────┘
```

---

## 4. Các verbs phổ biến trong RBAC

| Verb | Mô tả | HTTP Method |
|------|--------|-------------|
| `get` | Đọc một resource cụ thể | GET |
| `list` | Liệt kê tất cả resources | GET |
| `watch` | Theo dõi thay đổi (streaming) | GET |
| `create` | Tạo mới resource | POST |
| `update` | Cập nhật toàn bộ resource | PUT |
| `patch` | Cập nhật một phần resource | PATCH |
| `delete` | Xóa resource | DELETE |
| `deletecollection` | Xóa hàng loạt | DELETE |

---

## 5. Subjects — Ai được gán quyền?

| Loại Subject | Ví dụ | Mô tả |
|-------------|-------|-------|
| **User** | `jane` | Người dùng thực tế (quản lý bên ngoài K8s) |
| **Group** | `developers` | Nhóm người dùng |
| **ServiceAccount** | `default` | Tài khoản dịch vụ dùng bởi Pod/application |

---

## 6. Aggregated ClusterRoles

Kubernetes cho phép **kết hợp nhiều ClusterRole** thành một bằng cách dùng `aggregationRule`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: []  # rules được tự động điền từ các ClusterRole khớp label
```

---

## 7. Default ClusterRoles có sẵn

Kubernetes tự tạo sẵn một số ClusterRole quan trọng:

| ClusterRole | Mô tả |
|-------------|--------|
| `cluster-admin` | Toàn quyền trên cluster (super-admin) |
| `admin` | Quản trị trong namespace (tạo Role, RoleBinding) |
| `edit` | Đọc/ghi hầu hết resources trong namespace |
| `view` | Chỉ đọc trong namespace (không xem secrets) |

---

## 8. Best Practices

1. **Luôn áp dụng Least Privilege** — Chỉ cấp quyền tối thiểu cần thiết
2. **Dùng ServiceAccount riêng** cho mỗi application, không dùng `default`
3. **Ưu tiên RoleBinding + ClusterRole** thay vì ClusterRoleBinding khi chỉ cần quyền trong namespace
4. **Tránh dùng wildcard `*`** trong verbs/resources trừ khi thật sự cần
5. **Review quyền định kỳ** bằng `kubectl auth can-i --list`

---

## 9. Lệnh kubectl hữu ích

```bash
# Kiểm tra user có quyền gì
kubectl auth can-i create pods --as jane

# Liệt kê tất cả quyền của user
kubectl auth can-i --list --as jane

# Xem tất cả RoleBindings trong namespace
kubectl get rolebindings -n default

# Xem tất cả ClusterRoleBindings
kubectl get clusterrolebindings

# Tạo Role nhanh
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n default

# Tạo RoleBinding nhanh
kubectl create rolebinding read-pods --role=pod-reader --user=jane -n default
```

---

## 10. Tóm tắt

| Khái niệm | Phạm vi | Chức năng |
|-----------|---------|-----------|
| Role | Namespace | Định nghĩa quyền trong 1 namespace |
| ClusterRole | Cluster | Định nghĩa quyền trên toàn cluster |
| RoleBinding | Namespace | Gán Role/ClusterRole cho subject trong namespace |
| ClusterRoleBinding | Cluster | Gán ClusterRole cho subject trên toàn cluster |

> **Ghi nhớ:** RBAC chỉ cho phép **cấp quyền (allow)**, không có cơ chế **từ chối (deny)**. Mặc định, nếu không có rule nào cho phép thì sẽ bị từ chối.
