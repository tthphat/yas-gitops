# Helm Chart Flow trong GitOps

## Tổng quan

Tư duy của bạn cực kỳ sắc bén! Bạn đã hiểu đúng bản chất cách thức hoạt động của Helm Chart trong mô hình GitOps rồi đấy.

Tuy nhiên, để luồng này chuẩn nhất theo kiến trúc dự án Microservices chuyên nghiệp (như dự án **yas** bạn đang chạy), mình bổ sung một chi tiết nhỏ để bức tranh hoàn chỉnh hơn.

## Luồng trộn file (Helm Template Rendering)

Thay vì gọi là "kết hợp values của khuôn với values của service", luồng chuẩn sẽ diễn ra theo cơ chế: **Khuôn chung → Cấu hình riêng**.

### Bộ khuôn mẫu chung (Templates)

Các file `deployment.yaml`, `service.yaml`, `ingress.yaml` được viết dưới dạng khuôn mẫu (Template) với các biến đặt sẵn (ví dụ: `image: {{ .Values.backend.image.repository }}`). Bộ khuôn này thường nằm chung một thư mục Chart dùng chung hoặc nằm ngay trong thư mục Chart gốc của Service.

### File giá trị mặc định của khuôn (values.yaml gốc)

Nằm ngay cạnh bộ khuôn để định nghĩa các giá trị chạy thử mặc định (như cấu hình port, cấu hình CPU/RAM mặc định).

### File giá trị riêng của Service (File values của bạn)

Là file chứa các thông số đặc thù của môi trường hiện tại (ví dụ môi trường Dev thì trỏ sang Docker Hub của bạn **thaithienphu**, bật Ingress, đổi URL Elasticsearch...).

## Quá trình ArgoCD "Nhào nặn" ra Manifest cuối cùng

Khi ArgoCD kích hoạt quá trình đồng bộ (Sync), nó sẽ chạy một lệnh ngầm tương đương với lệnh `helm template` để thực hiện thuật toán **Ghi đè (Override)**:

1. **Bước 1:** Nó lấy file `values.yaml` riêng của Service đem đè lên file `values.yaml` mặc định của khuôn. Cái gì trùng tên thì lấy của Service, cái gì Service không khai báo thì giữ nguyên của khuôn.

2. **Bước 2:** Nó cầm cục data đã hợp nhất đó, bơm (inject) vào các file khuôn mẫu `deployment.yaml`, `service.yaml`, `ingress.yaml`.

3. **Bước 3:** Kết quả đầu ra là một file Manifest thuần (gộp tất cả lại) chứa dữ liệu thật 100%. File này được ném thẳng xuống Kubernetes API để tạo Pod.

## Áp dụng vào lỗi imagePullSecrets

Bởi vì bộ khuôn `deployment.yaml` của dự án **yas** chắc chắn có một dòng chờ sẵn dạng:

```yaml
{{- with .Values.imagePullSecrets }}
imagePullSecrets:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

Nên khi bạn dùng **Cách 1** (sửa file `kind: Application` con của ArgoCD) bằng cách nạp thêm `valuesObject`, ArgoCD sẽ hiểu là: *"À, anh bạn này muốn bổ sung cấu hình imagePullSecrets vào cục data riêng của Service"*. Nó liền nhồi giá trị đó vào cấu trúc dữ liệu, điền vào khuôn và sinh ra file Deployment hoàn chỉnh có chứa chiếc chìa khóa để K8s tự tin đăng nhập Docker Hub!

## Kết luận

Bạn thấy cơ chế hoạt động phối hợp này của Helm và ArgoCD có giúp tối ưu hóa việc quản lý hàng chục Microservices cùng lúc không?
