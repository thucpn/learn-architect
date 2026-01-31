# Hiểu Kubernetes (K8s) - Khái niệm & Kiến trúc

Tài liệu này giúp bạn hiểu Kubernetes là gì, các thành phần chính, cách vận hành, và các thuật ngữ quan trọng.

---

## Kubernetes là gì?

**Kubernetes** (K8s) là một nền tảng để quản lý và chạy ứng dụng **containerized** (chứa trong Docker) một cách tự động.

**Nôm na:**
- Bạn có nhiều container (ứng dụng)
- Kubernetes lo việc chạy chúng trên many servers, scale tự động, health check, v.v.
- Bạn chỉ nói "tôi muốn 5 replicas của app này" → K8s lo cách nó chạy

---

## Kiến trúc tổng quan

```
┌─────────────────────────────────────────────────────────────┐
│                      KUBERNETES CLUSTER                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────┐      ┌───────────────────────┐ │
│  │   MASTER (Control)     │      │   NODES (Workers)     │ │
│  ├────────────────────────┤      ├───────────────────────┤ │
│  │ - API Server           │      │  Node 1:              │ │
│  │ - Controller Manager   │      │  ├── kubelet          │ │
│  │ - Scheduler            │      │  ├── Docker/runtime   │ │
│  │ - etcd (database)      │      │  └── kube-proxy       │ │
│  │                        │      │                       │ │
│  │ (Điều phối toàn bộ)   │      │  Node 2:              │ │
│  │                        │      │  ├── kubelet          │ │
│  └────────────────────────┘      │  ├── Docker/runtime   │ │
│                                   │  └── kube-proxy       │ │
│  kubectl ───→ API Server          │                       │ │
│  (điều khiển)   (giao tiếp)       │  Node 3: ...          │ │
│                                   │                       │ │
│                                   └───────────────────────┘ │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Các Pod/Container chạy trên Nodes                   │  │
│  │  (quản lý bởi Master)                                │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Các thành phần chính

### Control Plane (Master)
Là "bộ não" của Kubernetes, nằm trên một hoặc nhiều master nodes.

**API Server**
- Tiếp nhận lệnh từ `kubectl`
- Xử lý requests (create Pod, deploy, etc.)
- Giao tiếp với các thành phần khác

**Scheduler**
- Quyết định Pod sẽ chạy trên Node nào
- Xem Node nào còn resources (CPU, memory) rồi assign Pod

**Controller Manager**
- Điều khiển các controller (Deployment Controller, ReplicaSet Controller, Service Controller, v.v.)
- Đảm bảo trạng thái thực tế (actual) khớp với mong muốn (desired)

**etcd**
- Database lưu tất cả thông tin cluster (state)
- Dùng để recover nếu master crash

### Worker Nodes
Là những máy chủ chạy các container/Pod.

**kubelet**
- Agent chạy trên mỗi Node
- Nhận lệnh từ API Server ("chạy Pod này")
- Gọi Docker/runtime để start container
- Monitor Pod health

**Container Runtime** (Docker, containerd, v.v.)
- Chạy container thực tế

**kube-proxy**
- Lo networking, service discovery
- Đảm bảo Pod A có thể gọi Pod B

---

## Các Resource chính

### Pod
**Là gì:** Đơn vị nhỏ nhất trong K8s. Chứa một hoặc nhiều container (thường 1).

**Ví dụ:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: nginx:1.19
```

**Điểm quan trọng:**
- Pod có IP riêng
- Container trong Pod share network (localhost)
- Pod tạm thời (có thể bị xóa bất cứ lúc nào)

---

### Deployment
**Là gì:** Quản lý Pod, khai báo: "tôi muốn 3 replicas của app này, dùng image này"

**Ví dụ:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:1.0
```

**K8s sẽ:**
- Tạo 3 Pod
- Nếu 1 Pod crash → tạo Pod mới
- Nếu bạn thay image → rolling update

---

### Service
**Là gì:** Tạo endpoint ổn định để truy cập Pod (vì Pod IP thay đổi)

**Ví dụ:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

**Types:**
- `ClusterIP`: chỉ Pod khác trong cluster có thể gọi
- `NodePort`: expose trên node port (external)
- `LoadBalancer`: tạo load balancer (cloud)

---

### ConfigMap & Secret
**ConfigMap:** Lưu cấu hình (không nhạy cảm)
**Secret:** Lưu thông tin nhạy cảm (passwords, tokens)

---

### Namespace
**Là gì:** Phân chia cluster thành các "phòng" tách biệt

**Ví dụ:**
- `default`: namespace mặc định
- `kube-system`: các component của K8s
- `prod`, `staging`, `dev`: tách theo environment

---

## Cách vận hành (Control Loop)

K8s hoạt động theo nguyên tắc **Desired State = Actual State**.

```
1. Bạn khai báo:
   "Tôi muốn 3 replicas"
   
2. K8s lưu vào etcd:
   desired_state = 3 replicas
   
3. Scheduler check:
   "Có 3 Pod không?"
   actual_state = 1 Pod
   
4. Không khớp!
   → Controller tạo 2 Pod nữa
   
5. Check lại:
   actual_state = 3 Pod
   ✅ Match! Done
```

**Liên tục check:** Nếu bạn xóa 1 Pod, Controller phát hiện → tạo lại ngay.

---

## Thuật ngữ quan trọng

### Pod
Đơn vị nhỏ nhất, chứa container.

### Replica / ReplicaSet
- **Replica:** 1 bản sao của Pod
- **ReplicaSet:** quản lý số replica (thường do Deployment quản lý)

### Label & Selector
- **Label:** tag gắn trên resource (ví dụ `app: nginx`)
- **Selector:** dùng để lọc resource (ví dụ `selector: {app: nginx}`)

Service dùng Selector để biết Pod nào cần expose.

### Rolling Update
Cập nhật Pod từng cái một mà không downtime.

```
Pod v1, Pod v1, Pod v1      (cũ, 3 cái)
     ↓
Pod v1, Pod v1, Pod v2      (1 cái mới)
     ↓
Pod v1, Pod v2, Pod v2      (2 cái mới)
     ↓
Pod v2, Pod v2, Pod v2      (3 cái mới) ✅
```

### Health Check
- **Liveness Probe:** "Pod còn sống không?" (nếu fail → restart)
- **Readiness Probe:** "Pod sẵn sàng nhận traffic không?" (nếu fail → loại khỏi Service)

### Scale
Tăng/giảm số replica.

```bash
kubectl scale deployment my-app --replicas=5
# Từ 3 → 5 Pod
```

### Rollback
Quay về phiên bản cũ.

```bash
kubectl rollout undo deployment my-app
# Quay về revision trước
```

---

## Vòng đời Pod

```
┌─────────────────────────────────────────────┐
│ 1. Pending                                  │
│ - Pod đã được tạo                           │
│ - Đang chờ được schedule tới Node           │
└─────────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 2. ContainerCreating                        │
│ - Scheduler chọn xong Node                  │
│ - Container đang được pull & start          │
└─────────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 3. Running                                  │
│ - Container đang chạy                       │
│ - Sẵn sàng nhận traffic                     │
└─────────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────────┐
│ 4. Succeeded / Failed / Unknown             │
│ - Pod đã kết thúc                           │
│ - (Job) hoặc bị crash (ReplicaSet tạo lại) │
└─────────────────────────────────────────────┘
```

---

## Request vs Limit

### Request
- "Tôi cần ít nhất bao nhiêu CPU/memory?"
- Scheduler dùng này để chọn Node
- Nếu không có Request, Scheduler không biết Pod cần bao nhiêu

### Limit
- "Tối đa Pod được dùng bao nhiêu?"
- Nếu vượt → Pod bị kill

**Ví dụ:**
```yaml
resources:
  requests:
    cpu: "100m"        # tối thiểu 100 milli-CPU
    memory: "128Mi"    # tối thiểu 128 MB
  limits:
    cpu: "500m"        # tối đa 500 milli-CPU
    memory: "512Mi"    # tối đa 512 MB
```

---

## Network (Networking)

### Pod Network
- Mỗi Pod có IP riêng
- Pod A ping Pod B được ngay (trong cluster)
- IP Pod thay đổi khi Pod restart

### Service Network
- Service có IP ổn định (Cluster IP)
- Service proxy request tới Pod

### Ingress
- Expose HTTP(S) ở layer 7 (domain-based)
- Cần Ingress Controller (nginx-ingress, traefik)

```
Internet → Ingress (domain routing) → Service → Pod
```

---

## Storage

### Ephemeral (tạm thời)
- Container data mất khi Pod xóa

### Persistent (bền vững)
- **PersistentVolume (PV):** storage thực tế
- **PersistentVolumeClaim (PVC):** request storage

```yaml
# PVC request 1GB
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

## RBAC (Role-Based Access Control)

Kiểm soát ai có quyền làm gì.

**Ví dụ:** ServiceAccount A chỉ được phép read Pod (không xóa).

---

## Namespace Isolation

```
┌─────────────────────────┐
│ Namespace: prod         │
│ ├── Pods                │
│ ├── Services            │
│ └── Configs             │
└─────────────────────────┘

┌─────────────────────────┐
│ Namespace: dev          │
│ ├── Pods                │
│ ├── Services            │
│ └── Configs             │
└─────────────────────────┘
```

---

## Cluster vs Node vs Pod

```
Cluster = toàn bộ K8s system
  ├── Master (Control Plane)
  └── Nodes (Workers)
       ├── Node 1 (máy chủ vật lý/VM)
       │   ├── Pod A (container)
       │   ├── Pod B
       │   └── Pod C
       ├── Node 2
       │   ├── Pod D
       │   └── Pod E
```

---

## Workflow thực tế: Deploy app

```
1. Bạn viết Deployment manifest (YAML)
2. kubectl apply -f deployment.yaml
3. API Server nhận request → lưu vào etcd
4. Controller phát hiện "muốn 3 Pod nhưng chưa có"
5. Scheduler chọn Nodes
6. kubelet on nodes bắt đầu containers
7. Service expose pods
8. App chạy! 🎉
```

---

## Tóm tắt khái niệm

| Thuật ngữ | Ý nghĩa |
|-----------|---------|
| **Cluster** | Toàn bộ K8s system |
| **Master/Control Plane** | Bộ não quản lý |
| **Node** | Máy chủ chạy container |
| **Pod** | Đơn vị nhỏ nhất (chứa container) |
| **Deployment** | Quản lý Pod replicas |
| **Service** | Endpoint ổn định để truy cập |
| **ConfigMap** | Cấu hình (public) |
| **Secret** | Cấu hình (private) |
| **Namespace** | Phân chia cluster |
| **Label** | Tag trên resource |
| **Selector** | Dùng để lọc theo label |
| **Replica** | Bản sao Pod |
| **Health Check** | Liveness & Readiness Probe |
| **Rolling Update** | Cập nhật từng cái một |
| **PVC** | Request storage |
| **Ingress** | HTTP(S) routing |
| **RBAC** | Quyền truy cập |

---

## Điểm quan trọng

1. **Declarative:** Bạn khai báo "muốn gì" (YAML), K8s lo cách làm
2. **Self-healing:** Pod crash → tạo lại; Node down → move Pod
3. **Scalable:** Tăng replicas dễ dàng
4. **Rolling updates:** Cập nhật mà không downtime
5. **Multi-cloud:** Chạy ở AWS, GCP, Azure, on-premise như nhau

---

## Học tiếp

- **Chạy kubectl commands** để thấy cách K8s hoạt động
- **Viết manifests** để hiểu cách declare resources
- **Deploy apps** để thấy Deployment, Service, Scaling
- **Troubleshoot** bằng logs, describe, events

---

**Bây giờ bạn có nền tảng để hiểu Kubernetes! Hãy thực hành với các lệnh kubectl.** 🚀
