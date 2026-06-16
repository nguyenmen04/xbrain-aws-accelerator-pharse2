# Bài 4: Trivy — Quét lỗ hổng bảo mật Container Image

> **Nguồn tham khảo:** https://aquasecurity.github.io/trivy

---

## 1. Trivy là gì?

**Trivy** (phát âm "trivvy") là công cụ **quét lỗ hổng bảo mật (vulnerability scanner)** mã nguồn mở, toàn diện nhất cho containers và cloud-native.

### Trivy quét được gì?

| Đối tượng | Mô tả |
|----------|-------|
| **Container Images** | Tìm CVE trong OS packages và libraries |
| **Filesystem** | Quét source code, config files |
| **Git Repositories** | Quét cả repo |
| **Kubernetes** | Quét cluster, workloads |
| **IaC (Terraform, CloudFormation)** | Tìm misconfiguration |
| **SBOM** | Tạo Software Bill of Materials |
| **Secrets** | Tìm secrets bị lộ trong code/image |

---

## 2. Tại sao cần quét Container Image?

```
Developer build image → Push lên registry → Deploy lên K8s

Nhưng image đó có thể chứa:
  ❌ CVE-2024-XXXX — lỗ hổng nghiêm trọng trong OpenSSL
  ❌ Outdated library — version cũ có known exploit
  ❌ Hardcoded secrets — password trong Dockerfile
  ❌ Misconfiguration — chạy container với root user

→ Trivy quét và PHÁT HIỆN trước khi deploy!
```

---

## 3. Cài đặt

```bash
# macOS
brew install trivy

# Linux (Ubuntu/Debian)
sudo apt-get install -y trivy

# Docker (không cần cài)
docker run aquasec/trivy image nginx:latest

# Windows (Chocolatey)
choco install trivy
```

---

## 4. Quét Container Image

### 4.1. Quét cơ bản

```bash
# Quét image nginx
trivy image nginx:1.25

# Kết quả mẫu:
# nginx:1.25 (debian 12.4)
# =============================
# Total: 142 (UNKNOWN: 0, LOW: 85, MEDIUM: 45, HIGH: 10, CRITICAL: 2)
#
# ┌──────────────┬──────────────────┬──────────┬───────────┬──────────────────┐
# │   Library    │  Vulnerability   │ Severity │  Version  │  Fixed Version   │
# ├──────────────┼──────────────────┼──────────┼───────────┼──────────────────┤
# │ libssl3      │ CVE-2024-XXXXX   │ CRITICAL │ 3.0.11-1  │ 3.0.13-1         │
# │ curl         │ CVE-2024-YYYYY   │ HIGH     │ 7.88.1    │ 7.88.1-10+deb12  │
# └──────────────┴──────────────────┴──────────┴───────────┴──────────────────┘
```

### 4.2. Chỉ hiển thị severity cao

```bash
# Chỉ hiển thị HIGH và CRITICAL
trivy image --severity HIGH,CRITICAL nginx:1.25

# Exit code 1 nếu tìm thấy (dùng trong CI/CD)
trivy image --severity CRITICAL --exit-code 1 nginx:1.25
```

### 4.3. Quét và fail CI/CD nếu có lỗ hổng

```bash
# Fail pipeline nếu có CRITICAL vulnerability
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# Nếu exit code = 1 → có vulnerability → CI/CD pipeline fail
# Nếu exit code = 0 → clean → tiếp tục deploy
```

---

## 5. Severity Levels

| Level | Mô tả | Hành động |
|-------|-------|-----------|
| **CRITICAL** | Có thể bị khai thác ngay, ảnh hưởng nghiêm trọng | ❌ PHẢI fix trước khi deploy |
| **HIGH** | Lỗ hổng nghiêm trọng nhưng khó khai thác hơn | ⚠️ Nên fix sớm |
| **MEDIUM** | Rủi ro trung bình | 📋 Lên kế hoạch fix |
| **LOW** | Rủi ro thấp | ℹ️ Theo dõi |
| **UNKNOWN** | Chưa đánh giá được | ℹ️ Kiểm tra thêm |

---

## 6. Tích hợp Trivy vào CI/CD

### GitHub Actions

```yaml
# .github/workflows/security-scan.yaml
name: Security Scan
on: push

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: table
          exit-code: 1                    # Fail nếu có vulnerability
          severity: CRITICAL,HIGH         # Chỉ fail cho HIGH+CRITICAL
```

### GitLab CI

```yaml
# .gitlab-ci.yml
trivy-scan:
  stage: test
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity CRITICAL,HIGH $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

---

## 7. Các lệnh hữu ích

```bash
# Quét image
trivy image nginx:1.25

# Quét filesystem (source code)
trivy fs .

# Quét Dockerfile
trivy config Dockerfile

# Quét Kubernetes cluster
trivy k8s --report summary

# Quét Terraform
trivy config --policy-namespaces user ./terraform/

# Tạo SBOM (Software Bill of Materials)
trivy image --format spdx-json -o sbom.json nginx:1.25

# Quét offline (không cần internet)
trivy image --skip-db-update --offline-scan myapp:1.0

# Output dạng JSON (cho automation)
trivy image --format json -o results.json nginx:1.25
```

---

## 8. Ignore lỗ hổng (khi cần)

Tạo file `.trivyignore`:

```
# .trivyignore
# Ignore CVE vì không ảnh hưởng ứng dụng của mình
CVE-2024-12345
CVE-2024-67890

# Ignore với comment
CVE-2024-11111  # False positive, không dùng module này
```

```bash
trivy image --ignorefile .trivyignore myapp:1.0
```

---

## 9. Best Practices

1. **Quét trong CI/CD** — Mỗi commit/PR phải qua scan trước khi merge
2. **Fail build cho CRITICAL** — Không deploy image có lỗ hổng nghiêm trọng
3. **Dùng base image nhỏ** — Alpine, distroless → ít package = ít vulnerability
4. **Update base image thường xuyên** — Rebuild image để nhận security patches
5. **Tạo SBOM** — Biết chính xác image chứa gì
6. **Kết hợp với Cosign** — Quét xong → ký image → verify trước deploy

---

## 10. Tóm tắt

| Khái niệm | Vai trò |
|-----------|---------|
| **Trivy** | Scanner toàn diện (CVE, secrets, IaC, SBOM) |
| **CVE** | Common Vulnerabilities & Exposures — mã lỗ hổng |
| **Severity** | Mức độ nghiêm trọng (CRITICAL → LOW) |
| **Exit code** | 0 = clean, 1 = có vulnerability (dùng trong CI/CD) |
| **SBOM** | Danh sách tất cả thành phần trong image |

> **Ghi nhớ:** Trivy là **"bước kiểm tra bảo mật"** trong CI/CD pipeline. Build image → **Trivy scan** → nếu clean thì mới push lên registry và deploy. Luôn scan trước khi deploy, đừng bao giờ deploy image chưa được quét!
