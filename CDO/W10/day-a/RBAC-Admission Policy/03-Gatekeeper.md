# Bài 3: OPA Gatekeeper — Policy Enforcement cho Kubernetes

> **Nguồn tham khảo:** https://open-policy-agent.github.io/gatekeeper/website/docs/

---

## 1. Gatekeeper là gì?

**Gatekeeper** là project tích hợp **OPA vào Kubernetes** như một **Admission Controller**. Nó cho phép bạn:

- **Validate** (kiểm tra) các resource trước khi chúng được tạo/cập nhật trong cluster
- **Mutate** (biến đổi) resource tự động trước khi lưu vào etcd
- **Audit** các resource hiện có trong cluster xem có vi phạm policy không

### Tại sao cần Gatekeeper khi đã có OPA?
- OPA là **general-purpose** policy engine → cần tích hợp thủ công với K8s
- Gatekeeper **đã tích hợp sẵn** với Kubernetes Admission Webhook
- Gatekeeper cung cấp **CRDs** (Custom Resource Definitions) để quản lý policy bằng `kubectl`
- Hỗ trợ **audit** resource đã tồn tại

---

## 2. Kiến trúc Gatekeeper

```
Client (kubectl, CI/CD, etc.)
        │
        ▼
┌─────────────────────┐
│   K8s API Server    │
│                     │
│  Admission Webhook ─┼──────► ┌───────────────────────┐
│                     │        │      Gatekeeper        │
└─────────────────────┘        │                        │
                               │  ConstraintTemplate    │
                               │      (Rego policy)     │
                               │           │            │
                               │           ▼            │
                               │     Constraint         │
                               │  (áp dụng policy       │
                               │   cho resource cụ thể) │
                               └───────────────────────┘
```

### Luồng hoạt động:
1. User tạo/sửa resource (ví dụ: `kubectl apply -f pod.yaml`)
2. K8s API Server gửi **admission review request** tới Gatekeeper webhook
3. Gatekeeper đánh giá request dựa trên **ConstraintTemplate + Constraint**
4. Trả về **allow** hoặc **deny** (kèm message giải thích)

---

## 3. Hai khái niệm cốt lõi: ConstraintTemplate vs Constraint

### 3.1. ConstraintTemplate — "Khuôn mẫu" policy

**ConstraintTemplate** định nghĩa:
- **Logic policy** bằng Rego (cái gì sẽ bị kiểm tra)
- **Parameters** mà policy chấp nhận (để tái sử dụng)

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels        # Tên CRD sẽ được tạo
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:                     # Parameter: danh sách label bắt buộc
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        import future.keywords.in

        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Resource thiếu các label bắt buộc: %v", [missing])
        }
```

### 3.2. Constraint — "Áp dụng" policy cụ thể

**Constraint** sử dụng ConstraintTemplate để áp dụng policy cho **resource types cụ thể** với **parameters cụ thể**.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels                  # Tên từ ConstraintTemplate
metadata:
  name: ns-must-have-team-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]             # Áp dụng cho Namespace
  parameters:
    labels: ["team", "environment"]      # Labels bắt buộc
```

### So sánh ConstraintTemplate vs Constraint

| | ConstraintTemplate | Constraint |
|---|---|---|
| **Vai trò** | Định nghĩa logic kiểm tra (WHAT) | Áp dụng kiểm tra (WHERE & HOW) |
| **Ai viết** | Platform/Security team | Cluster admin |
| **Chứa gì** | Rego code + parameter schema | Target resources + parameter values |
| **Tái sử dụng** | Được dùng bởi nhiều Constraint | Mỗi Constraint = 1 instance cụ thể |
| **Ví dụ** | "Kiểm tra label bắt buộc" | "Namespace phải có label 'team'" |

---

## 4. Enforcement Actions (Hành động khi vi phạm)

Gatekeeper hỗ trợ 3 chế độ:

| Action | Mô tả | Khi nào dùng |
|--------|-------|-------------|
| `deny` | **Từ chối** request vi phạm | Production enforcement |
| `dryrun` | Ghi nhận vi phạm nhưng **không chặn** | Testing, rollout dần |
| `warn` | Trả về **warning** nhưng vẫn allow | Giai đoạn chuyển đổi |

```yaml
spec:
  enforcementAction: dryrun   # Chỉ audit, không chặn
```

---

## 5. Audit — Kiểm tra resource đã tồn tại

Gatekeeper tự động **audit** tất cả resource hiện có trong cluster theo schedule:

```bash
# Xem danh sách vi phạm
kubectl get constraint ns-must-have-team-label -o yaml
```

Phần `status.violations` sẽ liệt kê tất cả resources vi phạm:

```yaml
status:
  totalViolations: 2
  violations:
    - enforcementAction: deny
      kind: Namespace
      message: "Resource thiếu các label bắt buộc: {\"team\"}"
      name: kube-system
```

---

## 6. Mutation (Biến đổi resource tự động)

Ngoài validation, Gatekeeper còn hỗ trợ **mutation** — tự động chỉnh sửa resource:

```yaml
apiVersion: mutations.gatekeeper.sh/v1
kind: Assign
metadata:
  name: add-default-team-label
spec:
  applyTo:
    - groups: [""]
      kinds: ["Namespace"]
      versions: ["v1"]
  match:
    scope: Namespaced
  location: "metadata.labels.team"
  parameters:
    assign:
      value: "default-team"    # Tự động thêm label nếu thiếu
```

---

## 7. Gatekeeper Library — Policy có sẵn

Gatekeeper cung cấp **thư viện policy sẵn có** tại:
🔗 https://open-policy-agent.github.io/gatekeeper-library/

Các policy phổ biến:
- **K8sRequiredLabels** — Bắt buộc label
- **K8sContainerLimits** — Bắt buộc resource limits cho container
- **K8sBlockNodePort** — Cấm dùng NodePort service
- **K8sDisallowedTags** — Cấm dùng image tag `latest`
- **K8sPSPPrivilegedContainer** — Cấm container chạy privileged

---

## 8. Cài đặt Gatekeeper

```bash
# Cài đặt bằng Helm
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper/gatekeeper --name-template=gatekeeper \
  --namespace gatekeeper-system --create-namespace

# Hoặc cài bằng kubectl
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.22.0/deploy/gatekeeper.yaml
```

---

## 9. Workflow thực tế

```
1. Platform team viết ConstraintTemplate (Rego logic)
        │
        ▼
2. Cluster admin tạo Constraint (áp dụng cho resource cụ thể)
        │
        ▼
3. Gatekeeper đăng ký Admission Webhook với K8s API Server
        │
        ▼
4. Mỗi khi có request CREATE/UPDATE → Gatekeeper đánh giá
        │
        ├── PASS → Allow request
        │
        └── FAIL → Deny + trả về error message
```

---

## 10. Tóm tắt

| Khái niệm | Vai trò |
|-----------|---------|
| **Gatekeeper** | Tích hợp OPA vào K8s qua Admission Webhook |
| **ConstraintTemplate** | Khuôn mẫu policy (chứa Rego logic + parameters) |
| **Constraint** | Instance cụ thể của template (target resources + values) |
| **Audit** | Kiểm tra resource hiện có xem có vi phạm không |
| **Mutation** | Tự động chỉnh sửa resource trước khi lưu |

> **Ghi nhớ:** ConstraintTemplate = **"Cái gì cần kiểm tra"** (tái sử dụng), Constraint = **"Kiểm tra ở đâu với giá trị gì"** (áp dụng cụ thể). Luôn bắt đầu với `enforcementAction: dryrun` trước khi chuyển sang `deny` trong production!
