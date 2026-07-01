# Jenkins Pipeline CI/CD

## Tổng quan

Jenkinsfile nằm ở `yas/Jenkinsfile` (source repo). Pipeline tự động:
1. Build service Java thành Docker image
2. Push image lên Docker Hub
3. Cập nhật tag trong GitOps repo (`yas-gitops`)
4. ArgoCD phát hiện thay đổi → sync → deploy lên K8S

---

## Pipeline Diagram

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────────┐    ┌───────────────┐    ┌──────────┐
│ Pipeline    │───▶│ Detect       │───▶│ Build & Test │───▶│ Build       │───▶│ Push         │───▶│ Update       │───▶│ Cleanup  │
│ Info        │    │ Changed      │    │ (Maven)      │    │ Docker      │    │ Docker Hub   │    │ GitOps Repo  │    │          │
│             │    │ Services     │    │              │    │ Image       │    │              │    │ (values.yaml)│    │          │
└─────────────┘    └──────────────┘    └──────────────┘    └─────────────┘    └──────────────┘    └───────┬───────┘    └──────────┘
                                                                                                          │
                                                                                                          ▼
                                                                                                   ┌──────────────┐
                                                                                                   │  ArgoCD      │
                                                                                                   │  auto sync   │
                                                                                                   │  → deploy    │
                                                                                                   └──────────────┘
```

---

## Chi tiết từng stage

### 1. Pipeline Info

In ra thông tin cơ bản: branch, commit SHA, image tag, log commit cuối.

```groovy
stage('Pipeline Info') {
    steps {
        echo " Branch    : ${env.BRANCH_NAME}"
        echo " Commit SHA: ${env.COMMIT_SHA}"
        echo " Image Tag : ${env.IMAGE_TAG}"
        sh 'git log -1 --oneline'
    }
}
```

### 2. Detect Changed Services

Dùng `git diff --name-only` để tìm file nào thay đổi trong commit hiện tại so với commit trước. Từ đó xác định service nào cần build.

```groovy
def changedFiles = sh(
    script: 'git diff --name-only HEAD~1 HEAD',
    returnStdout: true
).trim()

def changedServices = allServices.findAll { service ->
    changedFiles.contains("${service}/")
}
```

- Nếu commit chỉ sửa `product/src/...` → chỉ build service `product`
- Nếu commit sửa nhiều service → build tất cả service liên quan
- Nếu lần đầu chạy (không có HEAD~1) → build tất cả

### 3. Build & Test (Maven)

Chạy `mvn clean install` cho từng service đã detect:

```bash
mvn clean install -pl <service> -am -DskipTests=false --batch-mode
```

- `-pl <service>`: build đúng service đó
- `-am`: build cả dependency (common-library)
- `-DskipTests=false`: chạy unit test và integration test

### 4. Build Docker Image

Build Docker image với 2 tags:

| Tag | Mục đích | Ví dụ |
|-----|----------|-------|
| `commit SHA` | Xác định chính xác version | `tthphat/yas-tax:a1b2c3d` |
| `IMAGE_TAG` | `latest` (main) hoặc SHA | `tthphat/yas-tax:latest` |

```bash
docker build -t tthphat/yas-tax:a1b2c3d -t tthphat/yas-tax:latest ./tax
```

### 5. Push to Docker Hub

Đăng nhập Docker Hub (dùng Jenkins credential), push cả 2 tags:

```bash
docker push tthphat/yas-tax:a1b2c3d
docker push tthphat/yas-tax:latest
```

### 6. Update GitOps Repo

Stage này **chỉ chạy khi branch = `main`**. Nó:

1. Clone `yas-gitops` repo
2. Tìm file `values.yaml` tương ứng với service vừa build
3. Sửa dòng `tag: latest` → `tag: <commit-SHA>`
4. Commit & push lên `main`

Ví dụ: sau khi build service `tax`, file `k8s/charts/tax/values.yaml` được sửa:

```yaml
backend:
  image:
    repository: ghcr.io/nashtech-garage/yas-tax
    tag: a1b2c3d          # ← thay đổi từ 'latest'
```

**Mapping source → chart:**

| Source folder | Chart folder |
|---------------|--------------|
| `product/` | `k8s/charts/product/` |
| `cart/` | `k8s/charts/cart/` |
| `backoffice/` | `k8s/charts/backoffice-ui/` |
| `storefront/` | `k8s/charts/storefront-ui/` |

### 7. Cleanup

Xoá image khỏi ổ cứng Jenkins agent để tránh đầy disk:

```bash
docker rmi tthphat/yas-tax:a1b2c3d tthphat/yas-tax:latest || true
```

---

## Toàn bộ flow

```
[Dev commit code] → [Jenkins detect service thay đổi] → [Build + Test]
    → [Build Docker image] → [Push Docker Hub] → [Update values.yaml trong yas-gitops]
    → [Commit & push yas-gitops] → [ArgoCD auto-sync] → [K8S deploy pod mới]
```

## Yêu cầu Jenkins

| Cấu hình | Giá trị |
|----------|---------|
| GitHub webhook | Trỏ từ `yas` repo đến Jenkins |
| Jenkins credential `dckr_pat_...` | Docker Hub token |
| Jenkins credential GitHub | Token/SSH để push `yas-gitops` |
| Plugins | GitHub Integration, Pipeline |
