# Bài 6: Kyverno Verify Images — Kiểm tra chữ ký Image trước khi Deploy

> **Nguồn tham khảo:** https://kyverno.io/policies/?policytypes=verifyImages

---

## 1. Vấn đề

Bạn đã **ký image bằng Cosign** ở bước CI/CD. Nhưng làm sao đảm bảo **chỉ image đã ký mới được deploy** lên cluster?

```
Nếu không có admission verify:
  → Developer có thể deploy image CHƯA ký (bỏ qua CI/CD)
  → Attacker push image giả lên registry
  → Image bị sửa đổi sau khi ký

Kyverno Verify Images = "bảo vệ cửa vào cluster"
  → Chỉ image có chữ ký hợp lệ mới được deploy ✅
```

---

## 2. Cách hoạt động

```
kubectl apply -f deployment.yaml
        │
        ▼
┌──────────────────┐
│  K8s API Server  │
│       │          │
│  Admission       │
│  Webhook ────────┼──► Kyverno
│                  │        │
└──────────────────┘        ▼
                     Kiểm tra image:
                     ① Image có chữ ký không?
                     ② Chữ ký có hợp lệ không?
                     ③ Ký bởi đúng key/identity không?
                           │
                     ┌─────┴──────┐
                     │            │
                   ✅ PASS     ❌ FAIL
                   Allow       Deny
```

---

## 3. Policy Verify Images — Key-based

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  webhookTimeoutSeconds: 30
  rules:
    - name: verify-cosign-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"          # Áp dụng cho images từ registry này
          attestors:
            - entries:
                - keys:
                    publicKeys: |-       # Public key dùng để verify
                      -----BEGIN PUBLIC KEY-----
                      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
                      -----END PUBLIC KEY-----
```

---

## 4. Policy Verify Images — Keyless (OIDC)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-keyless-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-github-actions-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: https://rekor.sigstore.dev
```

> Chỉ chấp nhận image được ký bởi **GitHub Actions workflow cụ thể** từ branch `main`.

---

## 5. Policy phổ biến

### 5.1. Chỉ cho phép image từ registry tin cậy + có chữ ký

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"
            - "myregistry.azurecr.io/*"
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      ...
                      -----END PUBLIC KEY-----
    # Image từ registry KHÁC sẽ bị từ chối (không match rule nào)
```

### 5.2. Verify attestation (metadata bổ sung)

```yaml
verifyImages:
  - imageReferences:
      - "ghcr.io/myorg/*"
    attestations:
      - type: https://slsa.dev/provenance/v1    # SLSA provenance
        attestors:
          - entries:
              - keys:
                  publicKeys: |-
                    -----BEGIN PUBLIC KEY-----
                    ...
                    -----END PUBLIC KEY-----
        conditions:
          - all:
              - key: "{{ builder.id }}"
                operator: Equals
                value: "https://github.com/actions/runner"
```

---

## 6. Test Policy

```bash
# ❌ Deploy image KHÔNG CÓ chữ ký → BỊ CHẶN
kubectl run test --image=ghcr.io/myorg/unsigned-app:1.0
# Error: image verification failed

# ✅ Deploy image CÓ chữ ký hợp lệ → THÀNH CÔNG
kubectl run test --image=ghcr.io/myorg/signed-app:1.0
# pod/test created

# ✅ Image từ registry khác (nếu không trong imageReferences) → tùy config
kubectl run test --image=nginx:1.25
# Kết quả phụ thuộc vào policy có bắt buộc ALL images hay chỉ specific registries
```

---

## 7. Pipeline end-to-end

```
  CI/CD Pipeline                              Kubernetes Cluster
┌─────────────────┐                        ┌─────────────────────┐
│ ① Build image   │                        │                     │
│ ② Trivy scan    │                        │  Kyverno Policy:    │
│ ③ Push registry │                        │  "verify-signature" │
│ ④ Cosign SIGN   │──── deploy ──────────► │       │             │
└─────────────────┘                        │  Verify chữ ký?    │
                                           │  ✅ → Allow         │
                                           │  ❌ → Deny          │
                                           └─────────────────────┘
```

---

## 8. Tóm tắt

| Khái niệm | Vai trò |
|-----------|---------|
| **verifyImages** | Rule type trong Kyverno để verify chữ ký image |
| **imageReferences** | Image nào cần verify (wildcard pattern) |
| **attestors** | Ai đã ký (public key hoặc keyless identity) |
| **attestations** | Metadata bổ sung (SLSA provenance, scan results) |

> **Ghi nhớ:** Kyverno Verify Images = **"bảo vệ cuối cùng"** trong supply chain. CI/CD ký image (Cosign) → Kyverno verify trước khi cho deploy. Image không có chữ ký hoặc chữ ký sai → **bị từ chối**. Kết hợp với Trivy (scan) + Cosign (sign) = bộ ba bảo mật supply chain hoàn chỉnh.
