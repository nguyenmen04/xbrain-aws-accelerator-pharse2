# Bài 5: Cosign / Sigstore — Ký và Xác minh Container Image

> **Nguồn tham khảo:** https://docs.sigstore.dev/cosign/overview

---

## 1. Cosign là gì?

**Cosign** là công cụ để **ký số (sign)** và **xác minh (verify)** container images. Thuộc dự án **Sigstore** — bộ công cụ bảo mật chuỗi cung ứng phần mềm.

### Bài toán:

```
Bạn pull image "myapp:1.0" từ registry
  → Làm sao biết image này THẬT SỰ do team bạn build?
  → Có ai đã sửa đổi image sau khi build không?
  → Image này có qua CI/CD pipeline chính thức không?

Cosign giải quyết: KÝ image sau khi build → VERIFY trước khi deploy
```

---

## 2. Hai chế độ ký

### 2.1. Key-based Signing (dùng key pair)

```
┌──────────┐                    ┌──────────────┐
│ CI/CD    │   cosign sign      │   Registry   │
│ Pipeline │──(private key)────►│   + chữ ký   │
└──────────┘                    └──────┬───────┘
                                       │
┌──────────┐   cosign verify           │
│ K8s      │◄──(public key)────────────┘
│ Cluster  │   → OK ✅ hoặc FAIL ❌
└──────────┘
```

### 2.2. Keyless Signing (OIDC — không cần quản lý key)

```
┌──────────┐  ① Xác thực OIDC    ┌──────────────┐
│ CI/CD    │─(GitHub/Google)────►│   Fulcio      │
│ Pipeline │                     │ (cấp cert     │
│          │◄── ② Certificate ───│  tạm thời)    │
│          │                     └──────────────┘
│          │  ③ Ký image với cert
│          │────────────────────►┌──────────────┐
│          │                    │   Rekor       │
│          │  ④ Ghi vào log     │ (transparency │
└──────────┘    minh bạch       │   log)        │
                                └──────────────┘
```

**Keyless = không cần tạo/lưu/rotate private key** → đơn giản hơn rất nhiều!

---

## 3. Sigstore Components

| Component | Vai trò |
|-----------|---------|
| **Cosign** | CLI tool để ký & verify images |
| **Fulcio** | Certificate Authority — cấp certificate tạm thời dựa trên OIDC |
| **Rekor** | Transparency log — ghi lại mọi chữ ký (immutable) |
| **Sigstore policy-controller** | Verify chữ ký trong K8s admission |

---

## 4. Cài đặt Cosign

```bash
# macOS
brew install cosign

# Linux
curl -LO https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign

# Windows
choco install cosign

# Kiểm tra
cosign version
```

---

## 5. Key-based Signing (dùng key pair)

### Bước 1: Tạo key pair

```bash
cosign generate-key-pair

# Tạo 2 files:
#   cosign.key  ← Private key (GIỮ BÍ MẬT!)
#   cosign.pub  ← Public key (chia sẻ được)
```

### Bước 2: Ký image

```bash
# Ký image sau khi build & push
cosign sign --key cosign.key ghcr.io/myorg/myapp:1.0

# Cosign lưu chữ ký vào registry cùng image
```

### Bước 3: Verify image

```bash
# Xác minh image (ai cũng có thể verify bằng public key)
cosign verify --key cosign.pub ghcr.io/myorg/myapp:1.0

# Output:
# Verification for ghcr.io/myorg/myapp:1.0 --
# The following checks were performed on each of these signatures:
#   - The cosign claims were validated
#   - The signatures were verified against the specified public key
# ✅ Image hợp lệ

# Nếu image bị sửa hoặc không có chữ ký:
# Error: no matching signatures ❌
```

---

## 6. Keyless Signing (OIDC — khuyến nghị)

```bash
# Ký keyless (sẽ mở browser để xác thực OIDC)
cosign sign ghcr.io/myorg/myapp:1.0

# Trong CI/CD (GitHub Actions) — tự động lấy OIDC token
cosign sign ghcr.io/myorg/myapp:1.0
# → Không cần key, dùng GitHub OIDC identity

# Verify keyless
cosign verify \
  --certificate-identity "https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/myorg/myapp:1.0
```

---

## 7. Tích hợp vào CI/CD (GitHub Actions)

```yaml
# .github/workflows/build-sign.yaml
name: Build, Scan & Sign

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write          # ← Cần cho keyless signing

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build & Push
        run: |
          docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          exit-code: 1
          severity: CRITICAL

      - name: Sign image (keyless)
        uses: sigstore/cosign-installer@v3
      - run: cosign sign ghcr.io/${{ github.repository }}:${{ github.sha }}
```

---

## 8. Pipeline hoàn chỉnh

```
① Build image
      │
      ▼
② Trivy scan (quét vulnerability)
      │
      ├── FAIL → ❌ Dừng pipeline
      │
      ▼
③ Push image lên registry
      │
      ▼
④ Cosign sign (ký image)
      │
      ▼
⑤ Deploy lên K8s
      │
      ▼
⑥ Kyverno verify (kiểm tra chữ ký trước khi cho chạy)
      │
      ├── Không có chữ ký → ❌ Từ chối
      │
      └── Có chữ ký hợp lệ → ✅ Cho phép deploy
```

---

## 9. Tóm tắt

| Thành phần | Vai trò |
|-----------|---------|
| **Cosign** | Ký & verify container images |
| **Key-based** | Dùng key pair (cần quản lý key) |
| **Keyless (OIDC)** | Dùng identity provider, không cần key |
| **Fulcio** | Cấp certificate tạm thời |
| **Rekor** | Ghi log chữ ký minh bạch (immutable) |
| **Verify** | Xác minh image chưa bị sửa đổi |

> **Ghi nhớ:** Cosign đảm bảo **"image bạn deploy đúng là image bạn build"**. Build → Scan → **Sign** → Deploy → **Verify**. Ưu tiên **keyless signing** trong CI/CD (đơn giản, không cần quản lý key).
