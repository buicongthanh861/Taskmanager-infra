# Task Manager – K8s Deployment

Full-stack app (Frontend + Backend + PostgreSQL) deploy trên Kubernetes bằng Helm, monitoring với kube-prometheus-stack.

---

## Cấu trúc project

```
charts/
├── backend/           # Node.js API
├── frontend/          # React + Nginx
├── database/          # PostgreSQL StatefulSet
└── fullstack-monitor/ # Prometheus + Grafana + Alertmanager
```

---

## Kiến trúc

```
User
 ├── app.local       → frontend-service:80
 └── api.local/api   → backend-service:5001 → postgres:5432
```

Monitoring endpoints:

| URL                         | Service      |
|-----------------------------|--------------|
| http://cluster.prometheus   | Prometheus   |
| http://cluster.alertmanager | Alertmanager |
| http://cluster.grafana      | Grafana      |

---

## Yêu cầu

- Kubernetes cluster + `kubectl`
- Helm 3
- NGINX Ingress Controller

---

## Deploy

### 1. Tạo namespace

```bash
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace monitor
```

### 2. Tạo secrets

**Backend:**
```bash
kubectl create secret generic backend-secret \
  --namespace=backend \
  --from-literal=DB_HOST=postgres-service.backend.svc.cluster.local \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_NAME=<db_name> \
  --from-literal=DB_USER=<db_user> \
  --from-literal=DB_PASSWORD=<db_password> \
  --from-literal=JWT_SECRET=<jwt_secret> \
  --from-literal=JWT_EXPIRES_IN=7d \
  --from-literal=FRONTEND_URL=http://app.local
```

**Database:**
```bash
kubectl create secret generic postgres-secret \
  --namespace=backend \
  --from-literal=POSTGRES_DB=<db_name> \
  --from-literal=POSTGRES_USER=<db_user> \
  --from-literal=POSTGRES_PASSWORD=<db_password> \
  --from-literal=PGDATA=/var/lib/postgresql/data/pgdata
```

**Gmail (cho Alertmanager):**
```bash
kubectl create secret generic gmail-auth-secret \
  --namespace=monitor \
  --from-literal=password=<gmail_app_password>
```

> Dùng [App Password](https://support.google.com/accounts/answer/185833), không dùng password thường.

### 3. Cài các chart

```bash
helm install database  ./charts/database  --namespace backend
helm install backend   ./charts/backend   --namespace backend
helm install frontend  ./charts/frontend  --namespace frontend
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-stack ./charts/fullstack-monitor --namespace monitor
```

### 4. Cấu hình `/etc/hosts`

```
127.0.0.1  app.local api.local cluster.prometheus cluster.alertmanager cluster.grafana
```

---

## Monitoring & Alerts

Alert định nghĩa cho cả 3 service (backend, frontend, database):

| Alert               | Severity | Điều kiện                              |
|---------------------|----------|----------------------------------------|
| `*PodDead`          | critical | Pod Failed/Unknown > 1 phút            |
| `*CPUHigh`          | warning  | CPU > 80% request trong 5 phút         |
| `*MemoryHigh`       | critical | Memory > 90% limit trong 5 phút        |
| `BackendRestartHigh`| warning  | Restart > 3 lần trong 15 phút          |

Alert gửi về Gmail qua SMTP, repeat mỗi 1 giờ.

---

## HPA

| Service  | Min | Max | CPU trigger |
|----------|-----|-----|-------------|
| Backend  | 1   | 1   | 50%         |
| Frontend | 1   | 3   | 50%         |

> Cần cài Metrics Server để HPA hoạt động.

---

## Upgrade / Rollback

```bash
helm upgrade backend  ./charts/backend  --namespace backend
helm upgrade frontend ./charts/frontend --namespace frontend

helm rollback backend 1 --namespace backend
```

---

## Uninstall

```bash
helm uninstall frontend         --namespace frontend
helm uninstall backend          --namespace backend
helm uninstall database         --namespace backend
helm uninstall prometheus-stack --namespace monitor
```
