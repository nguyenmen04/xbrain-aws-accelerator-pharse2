# Bài 2: External Secrets Operator (ESO)

> **Nguồn tham khảo:** https://external-secrets.io/latest

---

## 1. ESO là gì?

**External Secrets Operator (ESO)** là một Kubernetes operator giúp **tự động đồng bộ secrets** từ các dịch vụ quản lý bí mật bên ngoài (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, GCP Secret Manager…) vào **Kubernetes Secrets**.

### Bài toán ESO giải quyết:

```
KHÔNG CÓ ESO:
  AWS Secrets Manager ──(thủ công)──► kubectl create secret ──► K8s Secret
  → Phải tạo/cập nhật thủ công mỗi khi secret thay đổi

CÓ ESO:
  AWS Secrets Manager ──(tự động)──► ESO ──(tự động)──► K8s Secret
  → ESO tự động sync, kể cả khi secret bị rotate
```

---

## 2. Kiến trúc

```
┌─────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                 │
│                                                      │
│  ┌──────────────┐      ┌─────────────────────┐      │
│  │ ExternalSecret│─────►│ External Secrets     │      │
│  │ (CRD)        │      │ Operator (controller)│      │
│  └──────────────┘      └──────────┬────────────┘     │
│                                   │                   │
│  ┌──────────────┐      ┌──────────▼────────────┐     │
│  │ K8s Secret   │◄─────│  SecretStore /         │     │
│  │ (auto-created│      │  ClusterSecretStore    │     │
│  │  & synced)   │      │  (CRD - kết nối tới   │     │
│  └──────────────┘      │   provider bên ngoài)  │     │
│                        └──────────┬─────────────┘     │
└───────────────────────────────────┼───────────────────┘
                                    │ API call
                          ┌─────────▼──────────┐
                          │  External Provider  │
                          │  - AWS Secrets Mgr  │
                          │  - HashiCorp Vault  │
                          │  - Azure Key Vault  │
                          │  - GCP Secret Mgr   │
                          └────────────────────┘
```

---

## 3. Ba CRDs cốt lõi

### 3.1. SecretStore / ClusterSecretStore — "Kết nối tới provider"

Định nghĩa **cách kết nối** tới dịch vụ secret bên ngoài.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore                    # Phạm vi: 1 namespace
metadata:
  name: aws-secrets-store
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa  # ServiceAccount có IRSA
```

| | SecretStore | ClusterSecretStore |
|---|---|---|
| **Phạm vi** | 1 namespace | Toàn cluster |
| **Ai tạo** | Namespace admin | Cluster admin |
| **Dùng khi** | Mỗi team có provider riêng | Dùng chung 1 provider |

### 3.2. ExternalSecret — "Secret nào cần sync?"

Khai báo **secret nào** từ provider cần được sync vào K8s.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: default
spec:
  refreshInterval: 1h               # ← Sync lại mỗi 1 giờ
  secretStoreRef:
    name: aws-secrets-store          # ← Tham chiếu SecretStore
    kind: SecretStore
  target:
    name: db-creds-k8s               # ← Tên K8s Secret sẽ được tạo
    creationPolicy: Owner
  data:
    - secretKey: username             # ← Key trong K8s Secret
      remoteRef:
        key: prod/myapp/db-creds      # ← Tên secret trên AWS
        property: username            # ← Field cụ thể trong JSON
    - secretKey: password
      remoteRef:
        key: prod/myapp/db-creds
        property: password
```

### 3.3. K8s Secret (auto-generated) — Kết quả

ESO tự động tạo K8s Secret:

```yaml
# Secret này được ESO tạo tự động, KHÔNG cần tạo thủ công
apiVersion: v1
kind: Secret
metadata:
  name: db-creds-k8s
  namespace: default
type: Opaque
data:
  username: YWRtaW4=          # base64("admin")
  password: TXlQQHNzdzByZA==  # base64("MyP@ssw0rd")
```

---

## 4. refreshInterval — Tự động sync

**refreshInterval** là khoảng thời gian ESO kiểm tra và sync lại secret từ provider.

```yaml
spec:
  refreshInterval: 1h    # Mỗi 1 giờ
  # refreshInterval: 15m  # Mỗi 15 phút
  # refreshInterval: 0    # Không tự động sync (chỉ sync 1 lần)
```

### Tại sao quan trọng?

```
Lúc 00:00  AWS secret = "password-A"  →  K8s secret = "password-A"
Lúc 00:30  AWS rotation → "password-B"
Lúc 01:00  refreshInterval trigger    →  K8s secret = "password-B" ✅
```

Nếu `refreshInterval: 0` → secret bị rotate trên AWS nhưng K8s vẫn giữ password cũ → app bị lỗi!

---

## 5. Các Provider được hỗ trợ

| Provider | Service |
|----------|---------|
| **AWS** | Secrets Manager, Parameter Store |
| **GCP** | Secret Manager |
| **Azure** | Key Vault |
| **HashiCorp** | Vault |
| **IBM** | Secrets Manager |
| **Oracle** | Vault |
| **Kubernetes** | Secrets (cross-namespace/cluster) |
| **1Password** | Connect |
| **Doppler** | Secrets |

---

## 6. Cài đặt ESO

```bash
# Cài bằng Helm
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace

# Kiểm tra
kubectl get pods -n external-secrets
```

---

## 7. Ví dụ đầy đủ: AWS Secrets Manager → K8s

### Bước 1: Tạo SecretStore

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-store
  namespace: app
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        jwt:
          serviceAccountRef:
            name: eso-sa
```

### Bước 2: Tạo ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: app
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: aws-store
    kind: SecretStore
  target:
    name: app-secrets-k8s
  data:
    - secretKey: DB_HOST
      remoteRef:
        key: prod/app/database
        property: host
    - secretKey: DB_PASSWORD
      remoteRef:
        key: prod/app/database
        property: password
    - secretKey: API_KEY
      remoteRef:
        key: prod/app/api-key
```

### Bước 3: Sử dụng trong Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: app
spec:
  containers:
  - name: app
    image: myapp:1.0
    envFrom:
      - secretRef:
          name: app-secrets-k8s   # ← K8s Secret do ESO tạo
```

---

## 8. Kiểm tra trạng thái

```bash
# Xem ExternalSecret status
kubectl get externalsecret -n app
# NAME          STORE       REFRESH INTERVAL   STATUS
# app-secrets   aws-store   30m                SecretSynced ✅

# Xem chi tiết
kubectl describe externalsecret app-secrets -n app

# Xem K8s Secret đã được tạo
kubectl get secret app-secrets-k8s -n app -o yaml
```

---

## 9. Tóm tắt

| CRD | Vai trò |
|-----|---------|
| **SecretStore** | Cấu hình kết nối tới provider (AWS, Vault…) |
| **ClusterSecretStore** | Giống SecretStore nhưng dùng cho toàn cluster |
| **ExternalSecret** | Khai báo secret nào cần sync + refreshInterval |
| **K8s Secret** | Được ESO tạo & cập nhật tự động |

> **Ghi nhớ:** ESO = **cầu nối** giữa secret provider bên ngoài và Kubernetes. Secrets được lưu an toàn ở provider, ESO tự động sync vào K8s Secret theo `refreshInterval`. Khi AWS rotate password → ESO tự động cập nhật K8s Secret → app luôn dùng password mới nhất.
