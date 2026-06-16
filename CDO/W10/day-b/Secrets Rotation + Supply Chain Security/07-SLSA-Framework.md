# Bài 7: SLSA Framework — Bảo mật Chuỗi Cung Ứng Phần Mềm

> **Nguồn tham khảo:** https://slsa.dev

---

## 1. SLSA là gì?

**SLSA** (đọc là "salsa") — **Supply-chain Levels for Software Artifacts** — là một **framework bảo mật** giúp đánh giá và cải thiện mức độ an toàn của chuỗi cung ứng phần mềm.

### Chuỗi cung ứng phần mềm (Supply Chain) là gì?

```
Source Code → Build → Package → Registry → Deploy → Production
     ↑          ↑        ↑          ↑          ↑
   Có thể    Có thể   Có thể    Có thể    Có thể
   bị sửa    bị sửa   bị sửa    bị sửa    bị sửa

→ MỖI BƯỚC đều là điểm tấn công tiềm năng!
```

### Tại sao quan trọng?

Các vụ tấn công supply chain nổi tiếng:
- **SolarWinds (2020):** Attacker chèn malware vào build process
- **Codecov (2021):** Script CI bị sửa để đánh cắp credentials
- **Log4Shell (2021):** Vulnerability trong library phổ biến
- **3CX (2023):** Ứng dụng chính thức bị chèn malware

---

## 2. SLSA Levels (v1.0)

SLSA định nghĩa **4 cấp độ** bảo mật, từ cơ bản đến nâng cao:

### Build Track (Focus chính)

| Level | Tên | Yêu cầu | Mô tả |
|-------|-----|---------|-------|
| **Build L0** | No guarantees | Không có gì | Không có bảo mật supply chain |
| **Build L1** | Provenance exists | Có provenance | Biết artifact được build ở đâu, bằng gì |
| **Build L2** | Hosted build | Build trên dịch vụ hosted | Build chạy trên CI/CD platform (không phải local) |
| **Build L3** | Hardened builds | Build platform không bị can thiệp | Build process được bảo vệ, tamper-proof |

---

## 3. Provenance — Khái niệm cốt lõi

**Provenance** = **"lý lịch"** của artifact — ghi lại:
- **Ai** build? (identity)
- **Build ở đâu?** (build platform)
- **Source code nào?** (git commit)
- **Build bằng gì?** (build script, commands)
- **Khi nào?** (timestamp)

### Ví dụ Provenance:

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [{
    "name": "ghcr.io/myorg/myapp",
    "digest": { "sha256": "abc123..." }
  }],
  "predicateType": "https://slsa.dev/provenance/v1",
  "predicate": {
    "buildDefinition": {
      "buildType": "https://github.com/actions/runner",
      "externalParameters": {
        "workflow": ".github/workflows/build.yml",
        "ref": "refs/heads/main"
      }
    },
    "runDetails": {
      "builder": {
        "id": "https://github.com/actions/runner/github-hosted"
      },
      "metadata": {
        "invocationId": "https://github.com/myorg/myapp/actions/runs/12345"
      }
    }
  }
}
```

→ Từ provenance, bạn biết chính xác image này **được build từ commit nào, bởi CI/CD nào, lúc nào**.

---

## 4. Mỗi Level yêu cầu gì?

### Build L1 — Provenance exists

```
✅ Tạo provenance cho mỗi build
✅ Provenance ghi lại: source, builder, build process
❌ Chưa cần: hosted build, tamper protection
```

**Cách đạt:** Dùng GitHub Actions + `slsa-github-generator`

### Build L2 — Hosted build platform

```
✅ Tất cả yêu cầu L1
✅ Build chạy trên hosted service (GitHub Actions, Google Cloud Build…)
✅ Provenance được tạo bởi build platform (không phải user)
❌ Chưa cần: hardened builds
```

**Cách đạt:** Build trên GitHub Actions (không build trên laptop)

### Build L3 — Hardened builds

```
✅ Tất cả yêu cầu L2
✅ Build runs trong isolated environment
✅ Provenance không thể bị giả mạo bởi user
✅ Build platform có kiểm soát truy cập chặt
```

**Cách đạt:** Dùng SLSA GitHub Generator (tạo provenance non-forgeable)

---

## 5. Áp dụng SLSA với GitHub Actions

### Tạo SLSA Provenance cho container image:

```yaml
# .github/workflows/slsa-build.yaml
name: SLSA Build

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write         # Cho keyless signing

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      - name: Build & Push
        id: build
        run: |
          docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
          # Output digest cho provenance
          echo "digest=$(docker inspect --format='{{index .RepoDigests 0}}' ghcr.io/${{ github.repository }}:${{ github.sha }})" >> $GITHUB_OUTPUT

  # Generate SLSA provenance
  provenance:
    needs: build
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v2.1.0
    with:
      image: ghcr.io/${{ github.repository }}
      digest: ${{ needs.build.outputs.digest }}
    secrets:
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

---

## 6. Verify SLSA Provenance

```bash
# Cài slsa-verifier
go install github.com/slsa-framework/slsa-verifier/v2/cli/slsa-verifier@latest

# Verify provenance của image
slsa-verifier verify-image ghcr.io/myorg/myapp@sha256:abc123 \
  --source-uri github.com/myorg/myapp \
  --source-branch main

# Output:
# Verified SLSA provenance for ghcr.io/myorg/myapp@sha256:abc123
# - Builder: https://github.com/slsa-framework/slsa-github-generator
# - Source: github.com/myorg/myapp@refs/heads/main
# - Build Level: SLSA Build L3 ✅
```

---

## 7. SLSA + Các công cụ khác

```
Source Code
    │
    ▼
┌─────────────────────────────────────────────┐
│              CI/CD Pipeline                  │
│                                              │
│  ① Build image                               │
│  ② Trivy scan (quét vulnerability)           │  ← Bài 4
│  ③ Cosign sign (ký image)                    │  ← Bài 5
│  ④ SLSA provenance (tạo lý lịch)            │  ← Bài 7
│  ⑤ Push to registry                         │
└──────────────────────┬───────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│           Kubernetes Cluster                 │
│                                              │
│  ⑥ Kyverno verify signature                 │  ← Bài 6
│  ⑦ Kyverno verify SLSA provenance           │  ← Bài 6+7
│  ⑧ ESO sync secrets                         │  ← Bài 2
│  ⑨ Deploy application                       │
└─────────────────────────────────────────────┘
```

---

## 8. Tóm tắt

| Khái niệm | Vai trò |
|-----------|---------|
| **SLSA** | Framework đánh giá bảo mật supply chain |
| **Provenance** | "Lý lịch" artifact — ai build, từ đâu, bằng gì |
| **Build L1** | Có provenance |
| **Build L2** | Build trên hosted platform |
| **Build L3** | Build platform hardened, provenance tamper-proof |
| **Supply Chain** | Toàn bộ quá trình: code → build → deploy |

> **Ghi nhớ:** SLSA không phải tool — mà là **framework/tiêu chuẩn** để đánh giá mức độ bảo mật supply chain. Bắt đầu từ **L1** (có provenance) rồi dần nâng lên **L3**. Kết hợp với Trivy (scan) + Cosign (sign) + Kyverno (verify) để có pipeline bảo mật hoàn chỉnh.

---

## Tổng kết toàn khóa: Supply Chain Security Pipeline

```
┌─────────────────────────────────────────────────────────┐
│                     SECRETS MANAGEMENT                   │
│                                                          │
│  AWS Secrets Manager ──► ESO ──► K8s Secrets             │
│  (lưu trữ + rotate)   (sync)   (dùng trong Pod)         │
│                                                          │
│  Sealed Secrets ──► Git ──► K8s Secrets                  │
│  (mã hóa)        (commit)  (giải mã trên cluster)       │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                 SUPPLY CHAIN SECURITY                    │
│                                                          │
│  Build ──► Trivy ──► Cosign ──► SLSA ──► Push            │
│           (scan)    (sign)   (provenance)                 │
│                                                          │
│  Deploy ──► Kyverno Verify ──► Allow/Deny                │
│            (check signature                              │
│             + provenance)                                │
└─────────────────────────────────────────────────────────┘
```
