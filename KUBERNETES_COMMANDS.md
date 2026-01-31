# Các lệnh Kubernetes (kubectl) thường dùng

Tài liệu này tập trung vào các lệnh kubectl cơ bản, dễ hiểu, không có ArgoCD.

---

## 1) Kiểm tra trạng thái cluster

### Xem node (máy chủ)
```bash
kubectl get nodes
```
- Liệt kê tất cả các node trong cluster
- READY = node sẵn sàng

### Xem namespace (phân vùng)
```bash
kubectl get namespaces
```
- Xem các namespace có trong cluster
- Mặc định là `default`

### Xem namespace hiện tại
```bash
kubectl config current-context
```
- Kiểm tra đang làm việc với cluster/namespace nào

---

## 2) Apply manifests (deploy ứng dụng)

### Apply một file
```bash
kubectl apply -f deployment.yaml
```
- Tạo hoặc cập nhật resource được định nghĩa trong file

### Apply toàn bộ thư mục
```bash
kubectl apply -f apps/nginx-demo/
```
- Apply tất cả file YAML trong thư mục

### Apply nhiều file cùng lúc
```bash
kubectl apply -f deployment.yaml -f service.yaml -f configmap.yaml
```

### Dry-run (kiểm tra trước khi apply)
```bash
kubectl apply -f deployment.yaml --dry-run=client
```
- Kiểm tra file có hợp lệ không mà không apply thực tế

### Apply kustomize overlay
```bash
kubectl apply -k apps/nginx-demo/overlays/prod
```
- Apply từ thư mục Kustomize

---

## 3) Xem resources

### Xem Deployment
```bash
kubectl get deployments
```
- Liệt kê tất cả Deployment

### Xem chi tiết Deployment
```bash
kubectl get deployment nginx-demo -o wide
```
- Xem chi tiết (image, replicas, age...)

### Xem Pods
```bash
kubectl get pods
```
- Liệt kê tất cả Pod

### Xem Pods với label
```bash
kubectl get pods -l app=nginx-demo
```
- Lọc Pod theo label `app=nginx-demo`

### Xem Service
```bash
kubectl get svc
```
- Xem tất cả Service (svc = service)

### Xem ConfigMap
```bash
kubectl get configmap
```

### Xem Secret
```bash
kubectl get secret
```

### Xem tất cả resources
```bash
kubectl get all
```

---

## 4) Xem chi tiết (describe)

### Mô tả Deployment
```bash
kubectl describe deployment nginx-demo
```
- Xem tất cả thông tin chi tiết về Deployment

### Mô tả Pod
```bash
kubectl describe pod nginx-demo-7db4d97c76-2s889
```
- Xem chi tiết 1 Pod (events, status, containers...)

### Mô tả Service
```bash
kubectl describe svc nginx-demo
```

---

## 5) Xem logs (output của Pod)

### Xem logs của 1 Pod
```bash
kubectl logs nginx-demo-7db4d97c76-2s889
```
- Xem output (logs) của container trong Pod

### Xem logs liên tục (follow)
```bash
kubectl logs -f nginx-demo-7db4d97c76-2s889
```
- `-f` = follow (giống `tail -f`)

### Xem logs của tất cả Pods với label
```bash
kubectl logs -l app=nginx-demo --all-containers=true
```
- Xem logs từ tất cả Pods có label `app=nginx-demo`

### Xem logs 10 dòng cuối
```bash
kubectl logs nginx-demo-7db4d97c76-2s889 --tail=10
```

---

## 6) Port-forward (truy cập từ localhost)

### Port-forward Service
```bash
kubectl port-forward svc/nginx-demo 8081:80
```
- Mở cổng 8081 trên localhost → 80 trên Service
- Chạy ở foreground (Ctrl+C để dừng)

### Port-forward Service (background)
```bash
kubectl port-forward svc/nginx-demo 8081:80 &
```
- `&` = chạy nền

### Port-forward Pod
```bash
kubectl port-forward pod/nginx-demo-7db4d97c76-2s889 8081:80
```

### Test kết nối
```bash
curl http://localhost:8081
```
- Kiểm tra service/pod có trả lời HTTP không

---

## 7) Thực thi lệnh trong Pod (exec)

### Chạy lệnh trong Pod
```bash
kubectl exec -it nginx-demo-7db4d97c76-2s889 -- sh
```
- `-i` = interactive, `-t` = tty, `--` = lệnh tiếp theo
- `sh` = shell bash/sh (tuỳ image)

### Chạy lệnh và xem output
```bash
kubectl exec nginx-demo-7db4d97c76-2s889 -- ls /etc/nginx
```
- Không có `-i`, chỉ xem output

---

## 8) Xóa resources

### Xóa Deployment
```bash
kubectl delete deployment nginx-demo
```
- Xóa Deployment (Pods cũng bị xóa)

### Xóa Pod
```bash
kubectl delete pod nginx-demo-7db4d97c76-2s889
```
- Xóa 1 Pod (Deployment sẽ tạo lại nó)

### Xóa Service
```bash
kubectl delete svc nginx-demo
```

### Xóa từ file
```bash
kubectl delete -f deployment.yaml
```

### Xóa tất cả trong namespace
```bash
kubectl delete all --all
```
- ⚠️ Cẩn thận! Xóa tất cả resources trong namespace hiện tại

---

## 9) Scale (tăng/giảm số Pod)

### Scale Deployment
```bash
kubectl scale deployment nginx-demo --replicas=5
```
- Thay đổi số replica từ 2 → 5

### Scale về 1 replica
```bash
kubectl scale deployment nginx-demo --replicas=1
```

---

## 10) Edit (sửa trực tiếp trên cluster)

### Edit Deployment
```bash
kubectl edit deployment nginx-demo
```
- Mở editor, sửa YAML, save → apply tự động
- ⚠️ Không khuyến khích (nên sửa file → apply)

### Edit Service
```bash
kubectl edit svc nginx-demo
```

---

## 11) Watch (theo dõi real-time)

### Watch Pods
```bash
kubectl get pods -w
```
- `-w` = watch (giống `watch` command)
- Theo dõi thay đổi realtime

### Watch Deployment
```bash
kubectl get deployment -w
```

---

## 12) Rollout (cập nhật/rollback)

### Xem history các revision
```bash
kubectl rollout history deployment nginx-demo
```

### Rollback về revision trước
```bash
kubectl rollout undo deployment nginx-demo
```

### Rollback về revision cụ thể
```bash
kubectl rollout undo deployment nginx-demo --to-revision=1
```

### Xem status rollout
```bash
kubectl rollout status deployment nginx-demo
```

---

## 13) Top (CPU/Memory usage)

### CPU & Memory của Pods
```bash
kubectl top pods
```
- Cần metrics-server cài trên cluster

### CPU & Memory của Nodes
```bash
kubectl top nodes
```

---

## 14) Namespace

### Đổi namespace mặc định
```bash
kubectl config set-context --current --namespace=argocd
```
- Sau đó `kubectl get pods` sẽ xem trong namespace `argocd`

### Xem resource trong namespace cụ thể
```bash
kubectl get pods -n argocd
```
- `-n` = namespace

### Tạo namespace
```bash
kubectl create namespace my-app
```

---

## 15) Apply khi có lỗi

### Validate trước
```bash
kubectl apply -f deployment.yaml --validate=true
```

### Force apply (không khuyến khích)
```bash
kubectl apply -f deployment.yaml --force
```

---

## Ví dụ thực tế

### Workflow: Deploy → Check → Debug

```bash
# 1. Deploy
kubectl apply -f apps/nginx-demo/

# 2. Kiểm tra Pods
kubectl get pods -l app=nginx-demo

# 3. Nếu Pod không ready, xem chi tiết
kubectl describe pod <pod-name>

# 4. Xem logs
kubectl logs <pod-name>

# 5. Port-forward & test
kubectl port-forward svc/nginx-demo 8081:80 &
curl http://localhost:8081

# 6. Nếu muốn sửa, thay file rồi apply lại
kubectl apply -f apps/nginx-demo/

# 7. Xem thay đổi realtime
kubectl get pods -w
```

---

## Những lệnh cần nhớ nhất

| Tác vụ | Lệnh |
|--------|------|
| Apply manifest | `kubectl apply -f file.yaml` |
| Xem Pods | `kubectl get pods` |
| Xem chi tiết | `kubectl describe pod <name>` |
| Xem logs | `kubectl logs <pod-name>` |
| Port-forward | `kubectl port-forward svc/<svc-name> 8081:80` |
| Xóa | `kubectl delete pod <name>` |
| Scale | `kubectl scale deployment <name> --replicas=3` |
| Xem realtime | `kubectl get pods -w` |
| Thực thi lệnh | `kubectl exec -it <pod> -- sh` |

---

## Tips

- Dùng `-o wide` hoặc `-o yaml` để xem chi tiết hơn
- Dùng `--watch` hoặc `-w` để theo dõi realtime
- Luôn sửa file YAML rồi apply (không edit trực tiếp trên cluster)
- Dùng label để lọc resources: `-l app=my-app`
- Dùng namespace để tách environments: `-n prod`

---

**Bây giờ bạn có thể sử dụng các lệnh này để thực hành!** 🚀
