# K3s + ArgoCD + Observability Lab Notes

## 1. Final architecture

```text
GitHub
  -> ArgoCD
  -> K3s
     -> frontend
     -> Prometheus
     -> Grafana
     -> Loki
     -> Promtail
```

## 2. What each component does

- `K3s`: Kubernetes cluster chạy workload.
- `ArgoCD`: theo dõi Git repo và sync manifest/chart xuống cluster.
- `frontend`: app lab để test GitOps và sinh traffic/log.
- `Prometheus`: scrape và lưu metrics.
- `Grafana`: query Prometheus/Loki và hiển thị dashboard.
- `Loki`: lưu và query logs.
- `Promtail`: đọc log container trên node và gửi vào Loki.

## 3. GitOps flow

```text
Edit YAML
  -> git commit
  -> git push
  -> ArgoCD detect change
  -> ArgoCD sync
  -> Cluster state changes
```

Điểm đã verify:

- bootstrap `root-app`
- `root-app` tạo app con
- `frontend` deploy qua GitOps
- scale `replicas: 2 -> 5` chỉ bằng Git push, không dùng `kubectl apply`

## 4. App of Apps mindset

- `root-app` quản lý thư mục `argocd-apps/`
- mỗi file trong `argocd-apps/` là một `ArgoCD Application`
- mỗi app con quản lý workload hoặc Helm chart riêng

Ví dụ:

```text
root-app
  -> frontend app
  -> prometheus app
  -> grafana app
  -> loki app
  -> promtail app
```

## 5. Kubernetes resource chain

```text
Deployment
  -> ReplicaSet
  -> Pods

Service
  -> chọn Pod qua labels
```

Ý nghĩa:

- `Deployment`: mong muốn chạy bao nhiêu replica
- `ReplicaSet`: giữ đủ số pod
- `Pod`: workload chạy thật
- `Service`: route traffic vào pod

## 6. Metrics pipeline

```text
Kubernetes / exporters
  -> Prometheus scrape
  -> Prometheus store metrics
  -> Grafana query Prometheus
  -> Dashboard / Explore
```

Đã test:

- query `up`
- query CPU / memory / pod count
- tự tạo dashboard metrics

## 7. Logs pipeline

```text
Container logs
  -> Promtail
  -> Loki
  -> Grafana Explore / dashboard
```

Đã test:

- query `{namespace="frontend"}`
- thấy nginx access log của frontend
- tạo dashboard Loki cơ bản

## 8. Useful PromQL

Pods by namespace:

```promql
count by (namespace) (kube_pod_info)
```

Node available memory in GB:

```promql
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```

Node CPU usage by instance:

```promql
sum by (instance) (rate(node_cpu_seconds_total{mode!="idle"}[5m]))
```

## 9. Useful LogQL

Frontend logs:

```logql
{namespace="frontend"}
```

Frontend GET requests:

```logql
count_over_time({namespace="frontend"} |= "GET"[5m])
```

Log volume by namespace:

```logql
sum by (namespace) (count_over_time({namespace=~".+"}[5m]))
```

## 10. Key troubleshooting lessons

### ArgoCD repo connect failed

Dependency chain:

```text
ArgoCD UI
  -> argocd-server
  -> argocd-repo-server
  -> CoreDNS / cluster DNS
  -> Internet access
  -> GitHub
  -> SSH trust
  -> SSH auth
  -> private repo
```

Root cause found:

- cluster DNS forward của CoreDNS đang trỏ vào `/etc/resolv.conf`
- Ubuntu dùng `127.0.0.53` (`systemd-resolved`)
- repo-server không resolve được `github.com`

Fix:

- sửa CoreDNS `forward . /etc/resolv.conf`
- đổi thành public DNS upstream

### Prometheus stack too complex for lab

`kube-prometheus-stack` gây thêm complexity:

- CRD
- Prometheus Operator
- admission webhook
- TLS/certificate
- finalizer stuck delete

Lesson:

- với lab học flow, chọn chart đơn giản trước
- không nhất thiết dùng stack production-oriented ngay từ đầu

### Stuck deleting ArgoCD application

Nếu app có:

- `deletionTimestamp`
- finalizer
- operation vẫn `Running`

thì app có thể bị stuck delete.

Case đã gặp:

- app `prometheus` stuck vì finalizer
- phải patch bỏ finalizer để gỡ app ra khỏi trạng thái treo

## 11. Observability mindset

- `Prometheus` = metrics backend
- `Grafana` = visualization layer
- `Loki` = logs backend
- `Promtail` = log shipper

Một câu dễ nhớ:

```text
Prometheus hỏi metrics
Promtail đẩy logs
Grafana là nơi mình nhìn cả metrics lẫn logs
```

## 12. What was completed

- K3s cluster running
- ArgoCD running
- private GitHub repo connected via SSH
- frontend deployed via GitOps
- ArgoCD App of Apps working
- Prometheus running
- Grafana running
- Loki running
- Promtail running
- metrics visible in Grafana
- logs visible in Grafana

## 13. Good next topics

- `SLI`
- `SLO`
- `Error Budget`
- `Burn Rate`
- `OTel SDK`
- `OTel Collector`

Recommended order:

```text
Metrics / Logs / Dashboards
  -> SLI / SLO
  -> Error Budget / Burn Rate
  -> OTel SDK + Collector
```
