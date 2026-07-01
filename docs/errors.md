Có vài vấn đề:
1. search — CrashLoopBackOff, xem log:
kubectl logs -n yas deployment/search --tail=20
2. yas-order-backend — InvalidImageName (sai tên image). Pod lạ, có thể là dư từ deploy trước:
kubectl delete pod -n yas yas-order-backend-5fd4b4d49b-9rt4b yas-order-backend-7fcf46644b-tmxrb
3. test-pod, test-pod-allowed — NotReady, có thể từ phần Service Mesh (bạn Mạnh).
Báo kết quả kubectl logs của search