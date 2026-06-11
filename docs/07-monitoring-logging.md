# Monitoring and Logging

## Overview

CloudPulse AI implements a comprehensive observability stack using the **kube-prometheus-stack** Helm chart, deployed and managed by ArgoCD. The stack provides metrics collection, visualization, and alerting for all application services and Kubernetes infrastructure components.

---

## Monitoring Stack Components

| Component | Version | Purpose |
|---|---|---|
| **Prometheus** | kube-prometheus-stack | Metrics collection and storage |
| **Grafana** | Included in kube-prometheus-stack | Metrics visualization and dashboards |
| **Alertmanager** | Included in kube-prometheus-stack | Alert routing and notification |
| **kube-state-metrics** | Included | Kubernetes object state metrics |
| **node-exporter** | Included | Node-level CPU/memory/disk metrics |
| **ServiceMonitor (backend)** | Custom | Scrapes `/actuator/prometheus` |
| **ServiceMonitor (AI service)** | Custom | Scrapes `/metrics` |

---

## ArgoCD Monitoring Application

The monitoring stack is deployed and managed by ArgoCD:

```yaml
# argocd/monitoring/kube-prometheus-stack-app.yaml
```

This installs the full `kube-prometheus-stack` chart into the `monitoring` namespace.

---

## ServiceMonitor Configuration

ServiceMonitors instruct Prometheus to scrape metrics from the application services.

### Backend ServiceMonitor

```yaml
# monitoring/servicemonitor-backend.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cloudpulse-backend
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
      - dev
      - prod
  selector:
    matchLabels:
      app: cloudpulse-backend
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 15s
```

The Spring Boot backend exposes metrics at `/actuator/prometheus` using the **Micrometer + Prometheus** integration.

### AI Service ServiceMonitor

```yaml
# monitoring/servicemonitor-ai-service.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cloudpulse-ai-service
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
      - dev
      - prod
  selector:
    matchLabels:
      app: cloudpulse-ai-service
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

The FastAPI AI service exposes metrics at `/metrics` using `prometheus_fastapi_instrumentator`.

---

## Metrics Collected

### Infrastructure Metrics (kube-state-metrics + node-exporter)

| Metric | Description |
|---|---|
| `kube_pod_status_phase` | Pod phase (Running, Pending, Failed) |
| `kube_deployment_status_replicas_available` | Available replicas per deployment |
| `kube_pod_container_status_restarts_total` | Container restart count |
| `node_cpu_seconds_total` | Node CPU usage |
| `node_memory_MemAvailable_bytes` | Node available memory |
| `container_cpu_usage_seconds_total` | Per-container CPU usage |
| `container_memory_working_set_bytes` | Per-container memory usage |

### Backend Application Metrics (Spring Boot Actuator)

| Metric | Description |
|---|---|
| `http_server_requests_seconds_count` | Total HTTP request count |
| `http_server_requests_seconds_sum` | Total request latency (sum) |
| `jvm_memory_used_bytes` | JVM heap and non-heap memory |
| `jvm_gc_pause_seconds` | Garbage collection pause duration |
| `process_cpu_usage` | Backend CPU usage |
| `hikaricp_connections_active` | Active database connections |

### AI Service Metrics (prometheus_fastapi_instrumentator)

| Metric | Description |
|---|---|
| `http_requests_total` | Total requests to AI service |
| `http_request_duration_seconds` | Request latency histogram |
| `http_requests_in_progress` | Inflight requests |

---

## Grafana Dashboard

A pre-built Grafana dashboard is provided for CloudPulse AI:

```
monitoring/dashboards/cloudpulse-dashboard.json
```

### Dashboard Panels

| Panel | Query | Description |
|---|---|---|
| Pod Status | `kube_pod_status_phase` | Real-time pod health across namespaces |
| CPU Usage | `rate(container_cpu_usage_seconds_total[5m])` | Per-container CPU rate |
| Memory Usage | `container_memory_working_set_bytes` | Per-container memory consumption |
| Backend Request Rate | `rate(http_server_requests_seconds_count[5m])` | Backend API throughput |
| Backend P95 Latency | `histogram_quantile(0.95, rate(...))` | 95th percentile API response time |
| AI Service Request Rate | `rate(http_requests_total[5m])` | AI service throughput |
| Container Restarts | `kube_pod_container_status_restarts_total` | Restart count (signals instability) |
| HPA Replica Count | `kube_horizontalpodautoscaler_status_current_replicas` | Current scaled replicas |

---

## Accessing Grafana

```bash
# Port-forward Grafana to local machine
kubectl port-forward svc/kube-prometheus-stack-grafana \
  -n monitoring 3000:80

# Open in browser
# http://localhost:3000
# Default credentials: admin / prom-operator
```

---

## Accessing Prometheus

```bash
# Port-forward Prometheus
kubectl port-forward svc/kube-prometheus-stack-prometheus \
  -n monitoring 9090:9090

# Open in browser
# http://localhost:9090
```

### Verify Targets

In the Prometheus UI:
1. Navigate to **Status → Targets**
2. Confirm `cloudpulse-backend` and `cloudpulse-ai-service` are listed as **UP**

If targets show **DOWN**:
- Verify ServiceMonitor selectors match the service labels
- Check that Prometheus has permission to scrape the `dev` and `prod` namespaces
- Run `kubectl get servicemonitor -n monitoring` to confirm resources exist

---

## Kubernetes-Level Logging

AKS sends container logs and cluster events to **Azure Log Analytics Workspace** (`cloudpulse-law`).

```bash
# View application logs directly
kubectl logs -n dev deployment/cloudpulse-backend --tail=100
kubectl logs -n dev deployment/cloudpulse-ai-service --tail=100
kubectl logs -n dev deployment/cloudpulse-frontend --tail=100

# Follow logs in real-time
kubectl logs -n dev -f deployment/cloudpulse-backend

# View previous container logs (after a crash)
kubectl logs -n dev deployment/cloudpulse-backend --previous
```

---

## Alert Configuration

Key alert conditions to configure in Alertmanager:

| Alert | Condition | Severity |
|---|---|---|
| Pod not Running | `kube_pod_status_phase != Running` for >5min | Critical |
| High Restart Count | `kube_pod_container_status_restarts_total > 5` | Warning |
| High CPU | CPU > 80% for >10min | Warning |
| High Memory | Memory > 85% for >10min | Warning |
| Deployment Not Available | `kube_deployment_status_replicas_available == 0` | Critical |

---

## Screenshots to Add

> Add the following screenshots to `docs/images/`:
>
> - `prometheus-targets.png` — Prometheus Targets page showing `cloudpulse-backend` and `cloudpulse-ai-service` as UP
> - `grafana-dashboard.png` — Grafana showing the CloudPulse AI dashboard with live data
> - `monitoring-pods.png` — `kubectl get pods -n monitoring` output
> - `servicemonitor-yaml.png` — ServiceMonitor YAML in editor or applied to cluster

---

*Previous: [Deployment Guide](06-deployment-guide.md) | Next: [Rollback Strategy](08-rollback-strategy.md)*
