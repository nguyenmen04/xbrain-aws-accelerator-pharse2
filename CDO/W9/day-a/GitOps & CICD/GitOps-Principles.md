# 📐 GitOps Principles — Cheat Sheet
> Nguồn: https://opengitops.dev | CNCF GitOps Working Group

---

## 1. OpenGitOps là gì?

**Không phải tool** — mà là bộ **nguyên tắc** định nghĩa "thế nào mới là GitOps".

```
OpenGitOps = Luật chơi
Argo CD     = Công cụ tuân theo luật đó
Flux CD     = Công cụ tuân theo luật đó
```

> Biết GitOps Principles → hiểu tại sao Argo CD/Flux hoạt động như vậy.

---

## 2. Trước và sau GitOps

```
❌ Trước (Push model):
Dev → CI/CD Pipeline → kubectl apply → Kubernetes
                            ↑
                     pipeline cần giữ credential cluster

✅ Sau (GitOps):
Dev → git push → Git Repository
                       ↑
              Argo CD / Flux tự kéo về → Kubernetes
```

**Điểm khác biệt:** Không ai deploy trực tiếp vào cluster — chỉ sửa Git.

---

## 3. 4 Nguyên tắc cốt lõi

### Principle 1: Declarative — Mô tả trạng thái, không ra lệnh

```yaml
# ✅ Declarative (GitOps) — Mô tả "muốn gì"
spec:
  replicas: 3

# ❌ Imperative (không phải GitOps) — Ra lệnh "làm gì"
kubectl create pod
kubectl create pod
kubectl create pod
```

> Quan tâm đến **WHAT** (muốn gì), không phải **HOW** (làm thế nào).

---

### Principle 2: Versioned & Immutable — Lưu trong Git, có lịch sử

```
Git log:
  commit A — replicas: 2  (Nguyen, 2025-06-01)
  commit B — replicas: 5  (Tran,   2025-06-08)

→ Biết ai sửa, sửa khi nào, rollback bằng: git revert
```

**GitOps ghét nhất:**
```bash
ssh prod-server && vim config.yaml   # ❌ Git không biết gì
kubectl edit deployment my-app       # ❌ Tạo drift, không có lịch sử
```

> Mọi thay đổi **phải đi qua Git commit** — không ngoại lệ.

---

### Principle 3: Pulled Automatically — Cluster tự kéo, không phải Pipeline đẩy

```
Push Model (CI/CD cũ):          Pull Model (GitOps):
GitHub Actions                  Git Repository
      ↓ (cần credential)               ↑
  kubectl apply           Argo CD / Flux (nằm trong cluster)
      ↓                                ↓
  Cluster                          Cluster
```

**Tại sao Pull an toàn hơn Push?**

| | Push | Pull |
|-|------|------|
| Credential | CI/CD giữ key của cluster | Cluster tự giữ key của Git |
| Bị hack CI/CD | Attacker có quyền prod | Attacker chỉ đọc được Git |
| Mở firewall | Cluster phải mở cho CI/CD | Cluster chỉ cần ra ngoài được |

---

### Principle 4: Continuously Reconciled — Tự sửa drift liên tục

```
Git:     replicas: 3
Cluster: replicas: 10  ← ai đó kubectl scale

→ Argo CD / Flux phát hiện drift
→ Tự sửa về 3
→ Không cần con người làm gì
```

**Reconciliation Loop** = vòng lặp chạy mãi:
```
So sánh Desired State (Git) vs Actual State (Cluster)
         ↓ có khác không?
    Sửa lại cho khớp
         ↓
    Lặp lại (mỗi vài phút)
```

> ⚠️ Nếu hệ thống **không tự sửa drift** → chưa phải GitOps, chỉ là IaC.

---

## 4. GitOps vs CI/CD — Điểm khác nhau

| | CI/CD truyền thống | GitOps |
|--|-------------------|--------|
| Model | Push | Pull |
| Ai deploy | Pipeline | Agent trong cluster |
| Chạy khi nào | Triggered 1 lần | Liên tục 24/7 |
| Tự sửa drift | ❌ Không | ✅ Có |
| Trung tâm | Source code | Git (state) |
| Rollback | Chạy lại pipeline | `git revert` |

---

## 5. Vì sao GitOps "hợp" với Kubernetes?

Kubernetes đã tư duy Declarative + Reconciliation từ đầu:

```
Deployment: replicas: 3
      ↓
Deployment Controller liên tục đảm bảo đúng 3 Pod
```

GitOps chỉ **nâng tư duy đó lên cấp cluster**:

```
Git: replicas: 3
      ↓
Argo CD / Flux liên tục đảm bảo cluster đúng với Git
```

---

## 6. Bảng ghi nhớ — 4 nguyên tắc vs Argo CD/Flux

| Nguyên tắc | Argo CD thực hiện bằng | Flux thực hiện bằng |
|-----------|----------------------|-------------------|
| **Declarative** | Application YAML | GitRepository + Kustomization YAML |
| **Versioned** | Git repo của bạn | Git repo của bạn |
| **Pulled** | App Controller kéo từ Repo Server | Source Controller kéo từ Git |
| **Reconciled** | Auto Sync + Self Heal | Reconciliation Loop (interval) |

---

## 7. Chốt lại — 30 giây trước phỏng vấn

```
GitOps = 4 nguyên tắc:

1. Declarative      — Mô tả "muốn gì", không ra lệnh "làm gì"
2. Versioned        — Mọi thứ lưu trong Git, có lịch sử, rollback dễ
3. Pulled           — Cluster tự kéo, không phải pipeline đẩy vào
4. Continuously Reconciled — Tự phát hiện và sửa drift 24/7
```

> **Câu trả lời khi phỏng vấn hỏi "GitOps là gì?":**  
> *"GitOps là mô hình vận hành trong đó Git là nguồn sự thật duy nhất. Mọi thay đổi hạ tầng đều qua Git commit, và một agent (Argo CD/Flux) trong cluster tự động kéo về và reconcile liên tục — không cần con người deploy thủ công và tự sửa khi có drift."*

---

> 📚 Docs: https://opengitops.dev  
> 📄 Spec đầy đủ: https://opengitops.dev/spec
