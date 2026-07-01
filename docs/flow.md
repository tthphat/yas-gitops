# Flow CI/CD + GitOps

## Tổng quan kiến trúc

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   Source Repo    │     │   Docker Hub     │     │   GitOps Repo    │
│ (github.com/     │────▶│ (image registry) │◀────│ (github.com/     │
│  tthphat/yas)    │     │                  │     │  tthphat/yas-    │
│                  │     │                  │     │  gitops)         │
│  Code Java       │     │  Image tags:     │     │                  │
│  Spring Boot     │     │  latest          │     │  Helm charts     │
│                  │     │  <commit-sha>    │     │  values.yaml     │
└────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
         │                        │                        │
         │  push                 push                     │
         ▼                        ▼                        ▼
    ┌───────────────────────────────────────────────────────────┐
    │                    Jenkins CI/CD                          │
    │                                                           │
    │  1. Checkout code                                         │
    │  2. Build service (Maven)                                 │
    │  3. Build Docker image (tag = commit-sha)                 │
    │  4. Push image lên Docker Hub                             │
    │  5. Clone yas-gitops                                      │
    │  6. Sửa values.yaml (tag = commit-sha)                   │
    │  7. Commit & push yas-gitops                              │
    └──────────────────────┬────────────────────────────────────┘
                           │
                           │ trigger
                           ▼
                    ┌──────────────────┐
                    │   ArgoCD         │
                    │                  │
                    │  Phát hiện       │
                    │  drift trong     │
                    │  GitOps Repo     │
                    │                  │
                    │  Tự động sync    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │  K8S Cluster     │
                    │  (Minikube)      │
                    │                  │
                    │  Namespace: yas  │
                    │  - product       │
                    │  - cart          │
                    │  - order         │
                    │  - tax           │
                    │  - ...           │
                    └──────────────────┘
```

---

## Chi tiết từng bước

### 1. Developer commit code

- Dev làm việc trên branch riêng, ví dụ: `dev_tax_service`
- Sửa code trong service `tax`
- Commit & push lên **Source Repo** (`github.com/tthphat/yas`)

### 2. Jenkins CI build

Jenkins job `developer_build` nhận input parameter:

| Parameter | Giá trị |
|-----------|---------|
| `tax-service` | `dev_tax_service` |
| `product` | `main` |
| `cart` | `main` |
| `order` | `main` |
| ... | `main` |

Jenkins thực hiện:

```bash
# Build service được chỉ định
git checkout dev_tax_service
cd tax
mvn clean package -DskipTests

# Build Docker image với tag = commit ID
COMMIT_ID=$(git rev-parse --short HEAD)
docker build -t tthphat/yas-tax:$COMMIT_ID .
docker push tthphat/yas-tax:$COMMIT_ID

# Clone GitOps repo & sửa tag
git clone https://github.com/tthphat/yas-gitops.git
cd yas-gitops
sed -i "s/tag:.*/tag: $COMMIT_ID/" k8s/charts/tax/values.yaml
git add .
git commit -m "Update tax image tag to $COMMIT_ID"
git push origin main
```

Các service còn lại giữ tag `main` (hoặc `latest`), không build lại.

### 3. GitOps repo thay đổi

File `k8s/charts/tax/values.yaml` được cập nhật:

```yaml
backend:
  image:
    repository: ghcr.io/nashtech-garage/yas-tax
    tag: a1b2c3d  # ← commit ID mới
```

### 4. ArgoCD phát hiện & deploy

- ArgoCD quét GitOps repo mỗi 3 phút (mặc định)
- Phát hiện file `values.yaml` thay đổi
- So sánh với trạng thái hiện tại trên cluster → **OutOfSync**
- Tự động **Sync**

#### ArgoCD Sync làm gì?

ArgoCD sync không tự biết pod nào cần update. Nó làm theo cơ chế **desired state**:

1. **Render Helm chart** — kết hợp template (`deployment.yaml`, `service.yaml`) với `values.yaml` → ra YAML hoàn chỉnh, ví dụ:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product          # ← tên cố định
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: product
          image: tthphat/yas-product:a1b2c3d  # tag từ values.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
```

2. **Diff** — So sánh YAML vừa render với resource đang chạy trên K8S (qua `metadata.name`)

3. **Apply** — `kubectl apply -f` lên namespace `yas`

4. **K8S tự xử lý phần còn lại**:
   - Thấy Deployment `product` đã tồn tại → cập nhật image tag
   - Deployment tạo ReplicaSet mới → pull image từ Docker Hub
   - ReplicaSet tạo pod mới, xóa pod cũ (rolling update)

→ **ArgoCD quản lý YAML (cấu hình mong muốn), K8S quản lý pod + pull image.**

#### Làm sao ArgoCD áp dụng đúng resource?

Mỗi resource trong K8S có **`metadata.name`** làm định danh duy nhất trong namespace. Khi ArgoCD apply:

| Resource | metadata.name | K8S hành động |
|----------|---------------|---------------|
| Deployment | `product` | Cập nhật Deployment `product` (nếu có) |
| Service | `product-service` | Cập nhật Service `product-service` |
| Ingress | `product-ingress` | Cập nhật Ingress `product-ingress` |

ArgoCD không cần biết pod cụ thể — nó apply Deployment, K8S tự quản lý pod thông qua ReplicaSet.

### 5. Kết quả

```bash
kubectl get pods -n yas
NAME                                 READY   STATUS    RESTARTS
tax-6bbdc694fc-l977l                 2/2     Running   0
product-766566f544-lnmns             2/2     Running   0
cart-5d947db4d5-bwfms                2/2     Running   0
...
```

Developer truy cập service qua NodePort để test:

```
http://<worker-node-ip>:<node-port>
```

---

## Cấu trúc GitOps Repo

```
yas-gitops/
├── k8s/
│   ├── argocd-apps/          # ArgoCD Application YAMLs
│   │   ├── product.yaml
│   │   ├── cart.yaml
│   │   ├── order.yaml
│   │   ├── tax.yaml
│   │   └── ...
│   ├── charts/               # Helm charts cho từng service
│   │   ├── backend/          # Chart chung (template Deployment, Service, Ingress)
│   │   ├── product/
│   │   │   ├── Chart.yaml    # Phụ thuộc backend
│   │   │   ├── values.yaml   # Config riêng (image tag, name, ingress)
│   │   │   └── charts/       # Dependency đã build (backend-0.1.0.tgz)
│   │   ├── cart/
│   │   └── ...
│   ├── deploy/               # Infrastructure YAML (Postgres, Kafka, Keycloak)
│   └── root-app/
│       └── root.yaml         # App of Apps (tạo toàn bộ app con 1 lần)
└── docs/
    └── flow.md               # File này
```

---

## Các thành phần chính

| Thành phần | Vai trò | Công nghệ |
|------------|---------|-----------|
| **Source Repo** | Chứa code Java Spring Boot | GitHub |
| **Docker Hub** | Lưu Docker images | Docker Hub |
| **GitOps Repo** | Chứa cấu hình deploy (Helm chart) | GitHub |
| **CI** | Build code, build image, push lên registry | Jenkins |
| **CD** | Deploy tự động khi có thay đổi trong GitOps | ArgoCD |
| **K8S** | Chạy ứng dụng | Minikube (1 node) |
| **Helm** | Đóng gói K8S YAML thành chart | Helm 3 |

---

## So sánh CI và CD

| | CI (Continuous Integration) | CD (Continuous Deployment) |
|--|-----------------------------|----------------------------|
| **Làm gì?** | Build code, test, tạo image | Deploy image lên cluster |
| **Công cụ** | Jenkins / GitHub Actions | ArgoCD |
| **Repo** | Source repo (`yas`) | GitOps repo (`yas-gitops`) |
| **Kết quả** | Docker image trên Docker Hub | Pod/service đang chạy trên K8S |
| **Khi nào chạy** | Khi dev commit/push code | Khi GitOps repo thay đổi |
