# 🔄 Flux CD — Cheat Sheet
> Nguồn: https://fluxcd.io/flux | Alternative to Argo CD | Dành cho người mới học

---

## 1. Flux là gì?

**GitOps tool cho Kubernetes** — liên tục đồng bộ Git → Cluster tự động.

```
Argo CD  = 1 ứng dụng lớn có UI đẹp
Flux     = nhiều controller nhỏ ghép lại, Kubernetes-native hơn
```

> **Bản chất Flux:** Không phải tool deploy — mà là **tập hợp các vòng lặp reconciliation** chạy 24/7.

---

## 2. Luồng hoạt động

```
Developer → git push → Git Repository
                              ↓
                    Source Controller (clone Git)
                              ↓
              Kustomize Controller      Helm Controller
              (apply YAML → cluster)   (helm install/upgrade)
                              ↓
                      Kubernetes Cluster
```

> Nhớ sơ đồ này = hiểu 50% Flux.

---

## 3. Reconciliation — Khái niệm cốt lõi

Flux không deploy 1 lần rồi thôi. Nó **lặp đi lặp lại**:

```
Git đúng chưa? → Cluster đúng chưa? → Khác nhau? → Sửa lại → lặp tiếp
```

```
Git:     replicas: 3
Cluster: replicas: 5   ← ai đó sửa tay

→ Flux phát hiện drift → tự đưa về 3
```

**Drift** = sự lệch giữa Git và Cluster (giống OutOfSync của Argo CD).

---

## 4. Kiến trúc — Các Controller

| Controller | Nhiệm vụ | CRD tương ứng |
|------------|---------|--------------|
| **Source Controller** | Clone Git/Helm repo, tạo artifact | `GitRepository`, `HelmRepository` |
| **Kustomize Controller** | Đọc YAML → apply vào cluster | `Kustomization` |
| **Helm Controller** | Deploy/upgrade/rollback Helm chart | `HelmRelease` |
| **Notification Controller** | Gửi thông báo Slack/Teams/Webhook | `Provider`, `Alert` |
| **Image Automation** | Theo dõi image mới → tự update Git | `ImageRepository`, `ImageUpdateAutomation` |

---

## 5. Ba CRD quan trọng nhất

### GitRepository — Khai báo nguồn Git

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m              # Kiểm tra Git mỗi 1 phút
  url: https://github.com/myorg/myrepo
  ref:
    branch: main
```

### Kustomization — Deploy YAML vào cluster

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 10m
  path: ./k8s/apps/my-app        # Thư mục YAML trong repo
  sourceRef:
    kind: GitRepository
    name: my-app
  prune: true                    # Xóa resource khi Git xóa
  targetNamespace: my-namespace
```

### HelmRelease — Deploy Helm chart

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: nginx
  namespace: flux-system
spec:
  interval: 10m
  chart:
    spec:
      chart: nginx
      version: '13.x'
      sourceRef:
        kind: HelmRepository
        name: bitnami
  values:
    replicaCount: 2
```

---

## 6. Bootstrap — Bước quan trọng nhất

Bootstrap = kết nối cluster với Git, cài toàn bộ Flux.

```bash
# Cài Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap với GitHub
flux bootstrap github \
  --owner=myorg \
  --repository=fleet-infra \
  --branch=main \
  --path=clusters/my-cluster \
  --personal
```

Sau lệnh này, Flux tự động:
1. Tạo repo `fleet-infra` trên GitHub (nếu chưa có)
2. Cài tất cả controllers lên cluster
3. Kết nối cluster ↔ Git

```
Sau bootstrap: Git ↔ Flux ↔ Cluster  ✅
```

---

## 7. Cấu trúc Git Repository (chuẩn)

```
fleet-infra/
├── clusters/
│   ├── dev/
│   │   ├── flux-system/         # Flux tự tạo
│   │   ├── apps.yaml            # Kustomization trỏ vào /apps
│   │   └── infrastructure.yaml  # Kustomization trỏ vào /infrastructure
│   └── prod/
│       └── ...
├── apps/
│   ├── my-app/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── another-app/
└── infrastructure/
    ├── ingress-nginx/
    └── cert-manager/
```

---

## 8. Lệnh hay dùng (Flux CLI)

```bash
# Xem trạng thái tất cả
flux get all

# Xem GitRepository
flux get sources git

# Xem Kustomization
flux get kustomizations

# Xem HelmRelease
flux get helmreleases

# Force reconcile ngay (không chờ interval)
flux reconcile kustomization my-app
flux reconcile source git my-app

# Xem logs của controller
flux logs --kind=Kustomization --name=my-app

# Suspend (tạm dừng sync)
flux suspend kustomization my-app

# Resume
flux resume kustomization my-app
```

---

## 9. Flux vs Argo CD — So sánh nhanh

| Tiêu chí | Flux | Argo CD |
|----------|------|---------|
| **Kiến trúc** | Nhiều controller nhỏ | 1 ứng dụng lớn |
| **UI** | Không có (hoặc WeaveGitOps) | ✅ UI mạnh, trực quan |
| **Kubernetes-native** | ✅ Rất cao | Cao |
| **Dễ học ban đầu** | Khó hơn | ✅ Dễ hơn |
| **Image auto-update** | ✅ Built-in | Cần cài thêm |
| **Multi-tenancy** | Tốt | Tốt |
| **Helm support** | ✅ HelmRelease CRD | ✅ Application + Helm |
| **Khi nào chọn** | Kubernetes-native, CI tự động | Cần UI, dễ onboard team |

---

## 10. Tính năng nâng cao hay gặp

### dependsOn — Thứ tự deploy
```yaml
spec:
  dependsOn:
    - name: infrastructure    # Deploy infra trước
# → my-app chỉ deploy sau khi infrastructure Healthy
```

### Health Checks
```yaml
spec:
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: my-app
      namespace: my-namespace
# → Kustomization chỉ "Ready" khi Deployment Healthy
```

### Image Automation — CI/CD không cần pipeline
```yaml
# 1. Theo dõi Docker registry
kind: ImageRepository
spec:
  image: myorg/myapp

# 2. Chọn version mới nhất theo pattern
kind: ImagePolicy
spec:
  filterTags:
    pattern: '^v[0-9]+\.[0-9]+\.[0-9]+$'  # Semantic versioning

# 3. Tự update Git + commit
kind: ImageUpdateAutomation
spec:
  git:
    commit:
      messageTemplate: 'Auto-update image to {{range .Updated.Images}}{{println .}}{{end}}'
```

**Kết quả:** Image mới lên Docker Hub → Flux tự update Git → Flux tự deploy → không cần ai làm gì!

---

## 11. Flux trong DevOps pipeline

```
Dev push code
     ↓
GitHub Actions
     ├── Build & Test
     ├── Build Docker Image → Push ECR/Docker Hub
     └── (Flux Image Automation tự update Git)
              ↓
           Flux phát hiện Git thay đổi
              ↓
           Reconcile → Apply vào Kubernetes
```

---

## 12. Checklist bắt đầu với Flux

- [ ] Cài `flux` CLI
- [ ] Chạy `flux check --pre` (kiểm tra prerequisites)
- [ ] Bootstrap với GitHub/GitLab
- [ ] Tạo `GitRepository` trỏ vào repo của mình
- [ ] Tạo `Kustomization` deploy app YAML đầu tiên
- [ ] Sửa Git → xem Flux tự reconcile
- [ ] Thêm `HelmRelease` deploy 1 Helm chart (ví dụ: ingress-nginx)
- [ ] Thử `flux suspend` / `flux resume`

---

> 📚 Docs đầy đủ: https://fluxcd.io/flux/  
> 🚀 Getting Started: https://fluxcd.io/flux/get-started/
