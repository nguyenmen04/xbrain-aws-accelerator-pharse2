# ⚙️ GitHub Actions — Cheat Sheet
> Nguồn: https://docs.github.com/en/actions | Dành cho người mới học

---

## 1. GitHub Actions là gì?

**Robot tự động** làm việc thay cho bạn sau mỗi sự kiện trên GitHub.

```
Trước:  Push code → Tự tay build → Tự tay test → Tự tay deploy   ❌
Sau:    Push code → GitHub Actions làm hết                         ✅
```

---

## 2. Bức tranh tổng thể

```
Workflow (.github/workflows/ci.yml)
    │
    ├── Job: build        ← chạy trên máy riêng (Runner)
    │     ├── Step: checkout code
    │     ├── Step: npm install
    │     └── Step: npm build
    │
    └── Job: deploy
          ├── Step: login Docker
          └── Step: push image
```

> **Nhớ thứ tự:** `Workflow → Job → Step → (Action hoặc Script)`

---

## 3. Thuật ngữ cốt lõi — 10 thứ phải nhớ

| # | Thuật ngữ | Hiểu đơn giản |
|---|-----------|--------------|
| 1 | **Workflow** | Toàn bộ quy trình tự động, lưu ở `.github/workflows/` |
| 2 | **Event** | Sự kiện kích hoạt workflow (push, PR, schedule...) |
| 3 | **Job** | Nhóm công việc, chạy trên 1 máy riêng |
| 4 | **Step** | Từng bước trong Job (chạy tuần tự) |
| 5 | **Runner** | Máy ảo chạy Job (Ubuntu/Windows/macOS — mới mỗi lần) |
| 6 | **Action** | Module có sẵn, tái sử dụng (`uses: actions/checkout@v4`) |
| 7 | **CI** | Tự động Build + Test mỗi khi push code |
| 8 | **CD** | Tự động Deploy sau khi CI thành công |
| 9 | **Secret** | Lưu mật khẩu/token an toàn, không hard-code trong code |
| 10 | **Artifact** | File đầu ra được lưu lại (app.jar, dist/, report...) |

---

## 4. Cấu trúc file Workflow

```yaml
name: CI Pipeline                    # Tên workflow

on:                                  # Event — khi nào chạy?
  push:
    branches: [main]
  pull_request:

jobs:
  build:                             # Job tên "build"
    runs-on: ubuntu-latest           # Runner

    steps:
      - uses: actions/checkout@v4   # Action: clone code về

      - name: Install dependencies
        run: npm install             # Script thường

      - name: Run tests
        run: npm test
```

---

## 5. Events — Sự kiện kích hoạt

```yaml
on:
  push:                    # Ai push code
  pull_request:            # Ai tạo Pull Request
  release:                 # Tạo release mới
  workflow_dispatch:       # Bấm tay trên UI
  schedule:                # Đặt lịch tự chạy
    - cron: '0 0 * * *'   # Mỗi ngày 12 giờ đêm
```

---

## 6. Secrets — Lưu thông tin nhạy cảm

```
❌ Sai:  AWS_KEY=AKIA1234567890   # Hard-code trong code = nguy hiểm

✅ Đúng:
  GitHub → Repository Settings → Secrets → New secret
  Tên: AWS_ACCESS_KEY
  Giá trị: AKIA1234567890
```

Dùng trong workflow:
```yaml
- name: Deploy to AWS
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_KEY }}
  run: aws s3 sync ./dist s3://my-bucket
```

---

## 7. CI Pipeline thực tế — Node.js

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'              # Cache để tăng tốc

      - run: npm ci
      - run: npm test
      - run: npm run lint
```

---

## 8. CD Pipeline thực tế — Build & Push Docker

```yaml
name: CD

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USER }}
          password: ${{ secrets.DOCKER_PASS }}

      - name: Build & Push Image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: myorg/myapp:${{ github.sha }}
```

---

## 9. Tính năng nâng cao hay gặp

### Cache — Tăng tốc workflow
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}
# Lần đầu: 3 phút | Lần sau: 20 giây
```

### Artifact — Lưu file đầu ra
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
# Sau đó download về máy hoặc dùng ở Job khác
```

### Matrix — Chạy nhiều môi trường cùng lúc
```yaml
jobs:
  test:
    strategy:
      matrix:
        node: [18, 20, 22]          # Chạy song song 3 version
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

### Reusable Workflow — Dùng lại ở nhiều repo
```yaml
# Repo A dùng lại workflow từ Repo Shared
jobs:
  deploy:
    uses: myorg/shared-workflows/.github/workflows/deploy.yml@main
    secrets: inherit
```

### Điều kiện chạy
```yaml
- name: Deploy (chỉ chạy trên main)
  if: github.ref == 'refs/heads/main'
  run: ./deploy.sh

- name: Notify (chỉ chạy khi thất bại)
  if: failure()
  run: ./notify-slack.sh
```

---

## 10. GitHub Actions trong DevOps hiện đại

```
Developer
    ↓
git push → GitHub
    ↓
GitHub Actions
    ├── Build & Test (CI)
    ├── Build Docker Image
    ├── Push lên ECR/Docker Hub
    └── Update K8s manifest (GitOps)
         ↓
      Argo CD phát hiện thay đổi
         ↓
      Sync lên Kubernetes
```

---

## 11. Permissions — Giới hạn quyền

```yaml
permissions:
  contents: read      # Chỉ đọc code
  packages: write     # Ghi packages (Docker registry)
  id-token: write     # Dùng OIDC (không cần lưu secret cloud)
```

> ✅ **Best Practice:** Chỉ cấp quyền tối thiểu cần thiết.

---

## 12. Quick Reference — Biến hay dùng

```yaml
${{ github.sha }}          # Commit hash hiện tại
${{ github.ref }}          # Branch name (refs/heads/main)
${{ github.actor }}        # Người trigger workflow
${{ github.repository }}   # org/repo-name
${{ secrets.MY_SECRET }}   # Secret đã lưu
${{ env.MY_VAR }}          # Environment variable
${{ runner.os }}           # Linux / Windows / macOS
```

---

## 13. Self-hosted Runner

Khi công ty muốn dùng máy chủ nội bộ thay vì máy GitHub:

```yaml
runs-on: self-hosted   # Thay vì ubuntu-latest
```

Cài agent trên máy công ty → GitHub kết nối vào để chạy workflow.  
Dùng khi cần: mạng nội bộ, phần cứng đặc biệt, bảo mật cao hơn.

---

## 14. Checklist để bắt đầu

- [ ] Tạo `.github/workflows/ci.yml` trong repo
- [ ] Viết workflow đơn giản: `checkout → install → test`
- [ ] Push code → xem workflow chạy trên tab Actions
- [ ] Thêm Secret (Settings → Secrets)
- [ ] Viết CD workflow: build Docker → push registry
- [ ] Thử Matrix build với 2 version Node.js
- [ ] Thử Reusable Workflow

---

> 📚 Docs đầy đủ: https://docs.github.com/en/actions  
> 🔍 Marketplace (10.000+ Actions có sẵn): https://github.com/marketplace?type=actions
