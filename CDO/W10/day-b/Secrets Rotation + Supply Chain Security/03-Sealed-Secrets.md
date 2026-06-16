# Bài 3: Sealed Secrets — Mã hóa Secrets để lưu trong Git

> **Nguồn tham khảo:** https://github.com/bitnami-labs/sealed-secrets

---

## 1. Sealed Secrets là gì?

**Sealed Secrets** (by Bitnami) là giải pháp cho phép bạn **mã hóa Kubernetes Secrets** để có thể **lưu trữ an toàn trong Git** — giải quyết bài toán GitOps.

### Bài toán:

```
K8s Secret (base64) = DỄ DÀNG giải mã → KHÔNG AN TOÀN để commit vào Git

apiVersion: v1
kind: Secret
data:
  password: TXlQQHNzdzByZA==    ← base64 decode = "MyP@ssw0rd" 😱

→ Nếu commit file này vào Git, ai cũng đọc được password!
```

### Giải pháp Sealed Secrets:

```
SealedSecret (encrypted) = CHỈ cluster mới giải mã được → AN TOÀN trong Git

apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
spec:
  encryptedData:
    password: AgBy3i...Ek=     ← Mã hóa bằng public key, chỉ cluster giải được ✅
```

---

## 2. Kiến trúc hoạt động

```
Developer Machine                      Kubernetes Cluster
┌─────────────────┐                   ┌─────────────────────────┐
│                  │                   │  Sealed Secrets         │
│  kubeseal CLI    │                   │  Controller             │
│       │          │                   │       │                 │
│  Secret.yaml     │                   │  Private Key 🔐         │
│       │          │                   │  (chỉ có trong cluster) │
│  ┌────▼────────┐ │    commit to Git  │       │                 │
│  │ SealedSecret│─┼──────────────────►│  ┌────▼────────┐       │
│  │ (encrypted) │ │                   │  │ Giải mã      │       │
│  └─────────────┘ │                   │  │    ↓         │       │
│                  │                   │  │ K8s Secret   │       │
│  Public Key 🔑   │                   │  │ (plain)      │       │
│  (ai cũng có)   │                   │  └──────────────┘       │
└─────────────────┘                   └─────────────────────────┘
```

### Luồng:
1. Developer dùng `kubeseal` + **public key** để mã hóa Secret → SealedSecret
2. Commit SealedSecret vào Git (an toàn!)
3. GitOps (ArgoCD/Flux) deploy SealedSecret lên cluster
4. Sealed Secrets Controller dùng **private key** giải mã → tạo K8s Secret

---

## 3. So sánh với External Secrets Operator

| | Sealed Secrets | External Secrets Operator |
|---|---|---|
| **Cách hoạt động** | Mã hóa secret, lưu trong Git | Sync secret từ provider bên ngoài |
| **Secret lưu ở đâu** | Trong Git (encrypted) | Trên provider (AWS, Vault…) |
| **Rotation** | Thủ công (phải re-seal) | Tự động (refreshInterval) |
| **Cần provider?** | Không | Cần (AWS SM, Vault…) |
| **GitOps friendly** | ✅ Rất tốt | ⚠️ Secret không nằm trong Git |
| **Phù hợp** | Team nhỏ, không có Vault | Team lớn, đã có secret provider |

---

## 4. Cài đặt

### Cài Sealed Secrets Controller trên cluster:

```bash
# Cài bằng Helm
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# Cài kubeseal CLI (macOS)
brew install kubeseal

# Cài kubeseal CLI (Linux)
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.0/kubeseal-0.27.0-linux-amd64.tar.gz
tar -xvzf kubeseal-*.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

---

## 5. Sử dụng Sealed Secrets

### Bước 1: Tạo Secret bình thường (chưa apply)

```bash
# Tạo file secret (ĐỪNG apply lên cluster)
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=MyP@ssw0rd \
  --dry-run=client -o yaml > secret.yaml
```

### Bước 2: Mã hóa bằng kubeseal

```bash
# Mã hóa secret → SealedSecret
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# Xóa file secret gốc (không cần nữa)
rm secret.yaml
```

### Bước 3: Xem kết quả

```yaml
# sealed-secret.yaml — AN TOÀN để commit vào Git
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-creds
  namespace: default
spec:
  encryptedData:
    username: AgBy3i4OJSWK+PiTySYZZA9rO...==
    password: AgCtr+Y2JA9XM+mvJQMdHj3...==
  template:
    metadata:
      name: db-creds
      namespace: default
    type: Opaque
```

### Bước 4: Apply SealedSecret

```bash
# Commit vào Git
git add sealed-secret.yaml
git commit -m "Add encrypted database credentials"

# Apply lên cluster
kubectl apply -f sealed-secret.yaml

# Controller tự động giải mã → tạo K8s Secret
kubectl get secret db-creds -o yaml
```

---

## 6. Scopes — Phạm vi mã hóa

| Scope | Mô tả | Flag |
|-------|-------|------|
| **strict** (mặc định) | Gắn với name + namespace cụ thể | `--scope strict` |
| **namespace-wide** | Dùng cho bất kỳ name nào trong namespace | `--scope namespace-wide` |
| **cluster-wide** | Dùng ở bất kỳ đâu trong cluster | `--scope cluster-wide` |

```bash
# Mã hóa với scope namespace-wide
kubeseal --scope namespace-wide --format yaml < secret.yaml > sealed.yaml
```

---

## 7. Key Management

### Xem public key:
```bash
kubeseal --fetch-cert > pub-cert.pem
```

### Key rotation (tự động mỗi 30 ngày):
- Controller tự tạo key pair mới
- Key cũ vẫn được giữ để giải mã SealedSecrets cũ
- SealedSecrets cũ vẫn hoạt động, không cần re-seal

### Backup private key (QUAN TRỌNG):
```bash
# Backup private key — cần cho disaster recovery
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-keys-backup.yaml

# GIỮ FILE NÀY AN TOÀN! Mất key = mất khả năng giải mã!
```

---

## 8. Tóm tắt

| Thành phần | Vai trò |
|-----------|---------|
| **kubeseal CLI** | Mã hóa Secret → SealedSecret (dùng public key) |
| **SealedSecret** | CRD chứa encrypted data (an toàn trong Git) |
| **Controller** | Giải mã SealedSecret → K8s Secret (dùng private key) |
| **Public key** | Ai cũng có — dùng để mã hóa |
| **Private key** | Chỉ có trong cluster — dùng để giải mã |

> **Ghi nhớ:** Sealed Secrets = **"mã hóa một chiều"**. Ai cũng có thể mã hóa (public key), nhưng chỉ cluster mới giải mã được (private key). → An toàn để commit vào Git. Luôn **backup private key** để phòng disaster recovery!
