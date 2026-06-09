# 🚀 Argo CD — Cheat Sheet
> Nguồn: https://argo-cd.readthedocs.io | Getting Started → App of Apps

---

## 1. Argo CD là gì?

**GitOps CD tool cho Kubernetes** — Git là nguồn sự thật duy nhất.

```
Trước:  Dev → kubectl apply → Cluster       ❌ không audit, không rollback
Sau:    Dev → git push → Argo CD → Cluster  ✅ có audit, tự phục hồi, dễ rollback
```

---

## 2. Kiến trúc

```
Web UI / CLI / CI
       ↓
  API Server  ──→  Repo Server (clone Git, render Helm/Kustomize)
       ↓
App Controller (so sánh Git vs Cluster → OutOfSync? → Sync)
       ↓
    Redis (cache)
```

---

## 3. Thuật ngữ cốt lõi

| Thuật ngữ | Ý nghĩa |
|-----------|---------|
| **Application** | CRD: "deploy cái gì, từ đâu, vào đâu" |
| **Target State** | Trạng thái mong muốn trong Git |
| **Live State** | Trạng thái thực tế trên cluster |
| **Sync** | Đưa cluster về đúng trạng thái Git |
| **OutOfSync** | Cluster ≠ Git |
| **Synced** | Cluster = Git |
| **Healthy** | App đang chạy bình thường |
| **Degraded** | App có lỗi (pod crash, v.v.) |
| **Refresh** | So sánh Git mới nhất vs Live State |
| **Drift** | Cluster bị ai đó sửa tay → lệch với Git |

> ⚠️ `Synced ≠ Healthy` — deploy đúng nhưng app vẫn có thể crash.

---

## 4. Cài đặt nhanh

```bash
# Cài Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Truy cập UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Lấy mật khẩu admin
argocd admin initial-password -n argocd

# Login CLI
argocd login localhost:8080
```

---

## 5. Application — YAML mẫu

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myrepo.git
    targetRevision: HEAD   # branch / tag / commit
    path: k8s/my-app       # thư mục trong repo
  destination:
    server: https://kubernetes.default.svc
    namespace: my-namespace
  syncPolicy:
    automated:
      prune: true       # xóa resource khi Git xóa
      selfHeal: true    # tự phục hồi khi bị sửa tay
    syncOptions:
      - CreateNamespace=true
```

---

## 6. Sync / Auto Sync / Self Heal / Prune

| Tính năng | Bật bằng | Tác dụng |
|-----------|----------|---------|
| **Manual Sync** | `argocd app sync <name>` | Người bấm tay |
| **Auto Sync** | `automated: {}` | Git thay đổi → tự deploy |
| **Self Heal** | `selfHeal: true` | Cluster bị sửa tay → tự sửa về Git |
| **Prune** | `prune: true` | Git xóa resource → cluster cũng xóa |

---

## 7. AppProject — Phân quyền đa team

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-backend
  namespace: argocd
spec:
  sourceRepos:
    - https://github.com/myorg/backend.git   # chỉ dùng repo này
  destinations:
    - namespace: backend-*                    # chỉ deploy namespace này
      server: https://kubernetes.default.svc
  roles:
    - name: dev
      policies:
        - p, proj:team-backend:dev, applications, sync, team-backend/*, allow
      groups:
        - myorg:backend-devs
```

---

## 8. ApplicationSet — Tạo hàng loạt Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-apps
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: dev   ; url: https://dev.example.com
          - cluster: prod  ; url: https://prod.example.com
  template:
    metadata:
      name: '{{cluster}}-myapp'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/myapp.git
        path: 'k8s/{{cluster}}'
        targetRevision: HEAD
      destination:
        server: '{{url}}'
        namespace: myapp
```

**Các Generator:**
- **List** — danh sách cố định
- **Cluster** — đọc từ clusters đã đăng ký trong Argo CD
- **Git** — đọc thư mục/file từ Git repo
- **Matrix** — nhân 2 generators: 3 env × 4 apps = 12 Applications

---

## 9. App of Apps Pattern

**Một Root App quản lý nhiều Child Apps** — dùng để bootstrap cluster.

```
Git repo:
  /apps/
    ingress-nginx.yaml   ← Application manifest
    cert-manager.yaml    ← Application manifest
    monitoring.yaml      ← Application manifest

Root App trỏ vào /apps/ → tự tạo và quản lý 3 Child Apps
```

**Root Application:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # xóa root → tự xóa child
spec:
  source:
    repoURL: https://github.com/myorg/cluster-config.git
    path: apps/
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

| | App of Apps | ApplicationSet |
|--|------------|----------------|
| Dùng khi | Bootstrap cluster, quản lý thủ công | Deploy cùng app sang nhiều clusters |
| Cách hoạt | Root → Child App manifests | Template + Generator → auto create |
| Khuyến nghị | Alternative | ✅ Recommended hiện tại |

---

## 10. Sync Wave & Hooks

**Sync Wave** — kiểm soát thứ tự deploy:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # deploy trước (số nhỏ → trước)
    argocd.argoproj.io/sync-wave: "0"    # sau
    argocd.argoproj.io/sync-wave: "1"    # cuối
```

**Hooks** — chạy script tại các thời điểm:
```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync    # chạy TRƯỚC sync
    argocd.argoproj.io/hook: PostSync   # chạy SAU sync thành công
    argocd.argoproj.io/hook: SyncFail   # chạy khi sync THẤT BẠI
```

Ví dụ: DB Migration job → `PreSync`, Smoke Test → `PostSync`

---

## 11. Lệnh hay dùng

```bash
argocd app list                          # Xem tất cả apps
argocd app get <name>                    # Xem chi tiết app
argocd app sync <name>                   # Sync app
argocd app diff <name>                   # Xem diff Git vs Cluster
argocd app history <name>                # Xem lịch sử deploy
argocd app rollback <name> <id>          # Rollback về version cũ
argocd cluster add <context>             # Thêm cluster ngoài
argocd cluster list                      # Xem danh sách clusters
argocd proj create <name>                # Tạo AppProject
```

---

## 12. Interview — 5 câu quan trọng nhất

**Q: Tại sao dùng Argo CD thay vì kubectl apply?**
> Audit trail (Git commit), rollback dễ, tự phục hồi drift, quản lý multi-cluster tập trung.

**Q: OutOfSync vs Degraded?**
> `OutOfSync` = Sync Status (cluster ≠ Git). `Degraded` = Health Status (app bị lỗi). Độc lập nhau.

**Q: Self Heal hoạt động thế nào?**
> Phát hiện drift (cluster bị sửa tay), tự sync lại về Git trong vài phút.

**Q: App of Apps vs ApplicationSet?**
> App of Apps: root app quản lý child apps (bootstrap). ApplicationSet: template + generator tự tạo hàng loạt app (multi-cluster).

**Q: AppProject dùng để làm gì?**
> Phân quyền đa team: giới hạn repo, namespace, cluster, loại resource, user nào được làm gì.

---

> 📚 Docs đầy đủ: https://argo-cd.readthedocs.io/en/stable/
