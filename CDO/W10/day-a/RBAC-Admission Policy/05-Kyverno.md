# Bài 5: Kyverno — Policy Engine YAML-native cho Kubernetes

> **Nguồn tham khảo:** https://kyverno.io/docs/

---

## 1. Kyverno là gì?

**Kyverno** (từ tiếng Hy Lạp "κυβερνώ" — govern/cai quản) là một **policy engine** được thiết kế **đặc biệt cho Kubernetes**. Điểm khác biệt lớn nhất: **policy được viết hoàn toàn bằng YAML** — không cần học ngôn ngữ mới như Rego.

### Đặc điểm nổi bật:
- **No new language:** Viết policy bằng YAML thuần — dễ học, dễ đọc
- **Kubernetes-native:** Thiết kế riêng cho K8s, hiểu sâu về K8s resources
- **Validate + Mutate + Generate:** Hỗ trợ cả 3 loại policy
- **Image Verification:** Xác minh chữ ký container image (Cosign, Notary)
- **Policy Reports:** Báo cáo vi phạm chi tiết qua CRDs

---

## 2. Kiến trúc Kyverno

```
                kubectl apply / CI/CD
                        │
                        ▼
            ┌──────────────────────┐
            │    K8s API Server    │
            │                      │
            │  Admission Webhooks ─┼──────►  ┌──────────────────┐
            │   (Mutating +        │         │     Kyverno      │
            │    Validating)       │         │                  │
            └──────────────────────┘         │  ┌────────────┐  │
                                             │  │  Validate  │  │
                                             │  ├────────────┤  │
                                             │  │  Mutate    │  │
                                             │  ├────────────┤  │
                                             │  │  Generate  │  │
                                             │  ├────────────┤  │
                                             │  │  Verify    │  │
                                             │  │  Images    │  │
                                             │  └────────────┘  │
                                             │                  │
                                             │  Policy Reports  │
                                             └──────────────────┘
```

---

## 3. Bốn loại Policy Rules

### 3.1. Validate — Kiểm tra resource

Kiểm tra resource có tuân thủ policy không → **cho phép hoặc từ chối**.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-team-label
spec:
  validationFailureAction: Enforce   # Enforce = chặn, Audit = chỉ báo cáo
  rules:
    - name: check-team-label
      match:
        any:
          - resources:
              kinds:
                - Namespace
      validate:
        message: "Namespace phải có label 'team'"
        pattern:
          metadata:
            labels:
              team: "?*"           # ?* = phải có giá trị (không rỗng)
```

### 3.2. Mutate — Tự động chỉnh sửa resource

Tự động thêm/sửa fields trước khi resource được lưu.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-resources
spec:
  rules:
    - name: add-default-limits
      match:
        any:
          - resources:
              kinds:
                - Pod
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              - (name): "*"          # Áp dụng cho tất cả container
                resources:
                  limits:
                    memory: "256Mi"  # Thêm memory limit nếu chưa có
                    cpu: "500m"      # Thêm cpu limit nếu chưa có
```

### 3.3. Generate — Tạo resource tự động

Tự động tạo resource mới khi một resource khác được tạo.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-default-networkpolicy
spec:
  rules:
    - name: default-deny-ingress
      match:
        any:
          - resources:
              kinds:
                - Namespace
      generate:
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        name: default-deny-ingress
        namespace: "{{request.object.metadata.name}}"
        data:
          spec:
            podSelector: {}
            policyTypes:
              - Ingress            # Tạo NetworkPolicy deny-all cho mỗi NS mới
```

### 3.4. Verify Images — Xác minh container image

Kiểm tra chữ ký số của container images.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-cosign-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/my-org/*"
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      ...
                      -----END PUBLIC KEY-----
```

---

## 4. Policy vs ClusterPolicy

| | Policy | ClusterPolicy |
|---|---|---|
| **Phạm vi** | Namespace cụ thể | Toàn cluster |
| **Áp dụng cho** | Resources trong 1 namespace | Resources trong mọi namespace |
| **Ai quản lý** | Namespace admin | Cluster admin |

---

## 5. Validation Failure Actions

| Action | Mô tả |
|--------|-------|
| `Enforce` | **Chặn** resource vi phạm (deny request) |
| `Audit` | **Cho phép** nhưng ghi nhận vi phạm trong Policy Report |

```yaml
spec:
  validationFailureAction: Audit    # Mặc định, không chặn
  # hoặc
  validationFailureAction: Enforce  # Chặn vi phạm
```

---

## 6. Pattern Matching — Cú pháp kiểm tra

Kyverno dùng **pattern matching** trong YAML:

| Pattern | Ý nghĩa | Ví dụ |
|---------|---------|-------|
| `"?*"` | Phải có giá trị (không rỗng) | `team: "?*"` |
| `"*"` | Bất kỳ giá trị nào (kể cả rỗng) | `env: "*"` |
| `"!latest"` | Không được là "latest" | `tag: "!latest"` |
| `">=1 & <=5"` | Trong khoảng 1-5 | `replicas: ">=1 & <=5"` |
| `">0"` | Lớn hơn 0 | `replicas: ">0"` |

---

## 7. Biến và Context

Kyverno hỗ trợ nhiều **biến tích hợp** và cho phép tạo biến tùy chỉnh:

### 7.1. Biến tích hợp

```yaml
# Request info
"{{request.operation}}"              # CREATE, UPDATE, DELETE
"{{request.object.metadata.name}}"    # Tên resource
"{{request.namespace}}"               # Namespace

# Service Account
"{{serviceAccountName}}"

# Thời gian
"{{time.now()}}"
```

### 7.2. Context variables (lấy dữ liệu từ cluster)

```yaml
rules:
  - name: check-configmap
    context:
      - name: allowedRegistries
        configMap:
          name: allowed-registries
          namespace: default
    validate:
      message: "Image registry không được phép"
      deny:
        conditions:
          - key: "{{request.object.spec.containers[0].image}}"
            operator: AnyNotIn
            value: "{{allowedRegistries.data.registries}}"
```

---

## 8. Policy Reports

Kyverno tự động tạo **Policy Reports** (CRD) để theo dõi trạng thái tuân thủ:

```bash
# Xem policy report trong namespace
kubectl get policyreport -n default

# Xem chi tiết
kubectl get policyreport -n default -o yaml

# Xem cluster-wide report
kubectl get clusterpolicyreport
```

Output mẫu:
```yaml
results:
  - policy: require-team-label
    rule: check-team-label
    result: fail
    message: "Namespace phải có label 'team'"
    resources:
      - name: my-namespace
        kind: Namespace
```

---

## 9. So sánh Kyverno vs Gatekeeper vs ValidatingAdmissionPolicy

| Tiêu chí | Kyverno | Gatekeeper | VAP (native) |
|----------|---------|------------|-------------|
| **Ngôn ngữ policy** | YAML | Rego | CEL |
| **Độ khó học** | ⭐ Dễ nhất | ⭐⭐⭐ Khó | ⭐⭐ Trung bình |
| **Validate** | ✅ | ✅ | ✅ |
| **Mutate** | ✅ | ✅ | ❌ |
| **Generate** | ✅ | ❌ | ❌ |
| **Verify Images** | ✅ | ❌ | ❌ |
| **Policy Reports** | ✅ Native | ⚠️ Qua status | ❌ |
| **Audit existing** | ✅ | ✅ | ❌ |
| **Cài đặt** | Helm/YAML | Helm/YAML | Không cần |
| **Community** | CNCF Incubating | CNCF Graduated (OPA) | K8s native |

---

## 10. Cài đặt Kyverno

```bash
# Cài đặt bằng Helm
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm install kyverno kyverno/kyverno \
  --namespace kyverno --create-namespace

# Cài policy library (tùy chọn)
helm install kyverno-policies kyverno/kyverno-policies \
  --namespace kyverno

# Kiểm tra
kubectl get pods -n kyverno
```

---

## 11. Best Practices

1. **Bắt đầu với `Audit` mode** → chuyển sang `Enforce` khi đã test kỹ
2. **Dùng `exclude`** để loại trừ system namespaces (`kube-system`, `kyverno`)
3. **Test policy trên staging** trước khi áp dụng production
4. **Sử dụng Kyverno CLI** (`kyverno apply`) để test policy offline
5. **Monitor Policy Reports** để theo dõi vi phạm liên tục

```bash
# Test policy offline với Kyverno CLI
kyverno apply policy.yaml --resource resource.yaml

# Kiểm tra policy trong cluster
kyverno test .
```

---

## 12. Tóm tắt

| Khái niệm | Vai trò |
|-----------|---------|
| **Kyverno** | Policy engine YAML-native cho Kubernetes |
| **ClusterPolicy** | Policy áp dụng toàn cluster |
| **Policy** | Policy áp dụng trong 1 namespace |
| **Validate** | Kiểm tra và cho phép/từ chối resource |
| **Mutate** | Tự động chỉnh sửa resource |
| **Generate** | Tạo resource mới tự động |
| **VerifyImages** | Xác minh chữ ký container image |
| **Policy Reports** | Báo cáo tuân thủ policy |

> **Ghi nhớ:** Kyverno là lựa chọn tuyệt vời cho team muốn bắt đầu nhanh với policy enforcement mà **không cần học ngôn ngữ mới**. Nếu team đã quen Rego, Gatekeeper có thể phù hợp hơn. Nếu chỉ cần validation đơn giản và đang chạy K8s 1.30+, ValidatingAdmissionPolicy là lựa chọn nhẹ nhất.
