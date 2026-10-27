# Part 15 — Monitoring with Prometheus and Grafana

[← Part 14 — Monitoring with Azure Monitor](14-azure-monitor.md) | [Index](../README.md) | [Part 16 — Kubernetes exercises →](16-kubernetes-exercises.md)

---

We install the open-source **kube-prometheus-stack** with Helm: Prometheus Operator, Prometheus, Alertmanager, Grafana, kube-state-metrics and node-exporter. The managed Azure version (Azure Monitor managed Prometheus + Azure Managed Grafana) is in Appendix C.

## 15.1 Values and alert rules

`k8s/monitoring/kube-prometheus-stack-values.yaml`:

```yaml
# AKS hides the control plane, so Prometheus cannot scrape these. Disabling them avoids "target down" noise.
kubeControllerManager:
  enabled: false
kubeScheduler:
  enabled: false
kubeEtcd:
  enabled: false
kubeProxy:
  enabled: false

prometheus:
  prometheusSpec:
    # Pick up ServiceMonitors / PrometheusRules from ALL namespaces, not only ones labelled for this release.
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
    retention: 2d
    resources:
      requests: { cpu: 200m, memory: 512Mi }
      limits: { memory: 1536Mi }

alertmanager:
  alertmanagerSpec:
    resources:
      requests: { cpu: 20m, memory: 64Mi }
      limits: { memory: 128Mi }

grafana:
  persistence:
    enabled: false # lab only: dashboards you create in the UI are lost on restart
  resources:
    requests: { cpu: 50m, memory: 128Mi }
    limits: { memory: 256Mi }
```

`k8s/monitoring/backend-alerts.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: backend-alerts
  namespace: monitoring
spec:
  groups:
    - name: backend
      rules:
        - alert: BackendHigh5xxRate
          expr: |
            sum(rate(http_server_requests_seconds_count{application="backend", status=~"5.."}[5m]))
              /
            sum(rate(http_server_requests_seconds_count{application="backend"}[5m])) > 0.05
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "More than 5% of backend requests return 5xx"

        - alert: BackendTargetDown
          expr: up{namespace="app", service=~".*backend.*"} == 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Prometheus cannot scrape backend pod {{ $labels.pod }}"

        - alert: AppPodRestarting
          expr: increase(kube_pod_container_status_restarts_total{namespace="app"}[15m]) > 2
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.pod }} restarted more than twice in 15 minutes"
```

## 15.2 Install (from the jumpbox)

```bash
cd ~/three-tier-aks && git pull
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo prometheus-community/kube-prometheus-stack     # note the CHART VERSION (92.x when this was written)

kubectl create namespace monitoring
helm upgrade --install kps prometheus-community/kube-prometheus-stack -n monitoring \
  --version <CHART VERSION> -f k8s/monitoring/kube-prometheus-stack-values.yaml --wait --timeout 15m
kubectl get pods -n monitoring
kubectl apply -f k8s/monitoring/backend-alerts.yaml
```

Always pin `--version` for third-party charts, so an upstream release never changes your cluster by surprise.

## 15.3 Let Prometheus scrape the backend

Set the GitHub variable **`SERVICE_MONITOR_ENABLED` = `true`** and re-run **app-ci-cd**. The chart now creates a `ServiceMonitor`. The Prometheus Operator turns it into scrape config for `/actuator/prometheus` on every backend pod. The NetworkPolicy already allows the `monitoring` namespace to reach the backend on 8080.

## 15.4 Open the UIs through an SSH tunnel

```bash
# laptop: one SSH session that forwards two ports
ssh -L 3000:localhost:3000 -L 9090:localhost:9090 azureuser@$JUMPBOX_IP

# on the jumpbox (inside that session)
kubectl -n monitoring port-forward svc/kps-grafana 3000:80 >/dev/null &
kubectl -n monitoring port-forward svc/kps-kube-prometheus-stack-prometheus 9090:9090 >/dev/null &
kubectl -n monitoring get secret kps-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Open <http://localhost:9090> (Prometheus) and <http://localhost:3000> (Grafana, user `admin` + the password above) on your laptop.

## 15.5 Prometheus: targets and PromQL

**Status → Targets**: `serviceMonitor/app/ttapp-backend/0` shows 2 targets **UP**. If it says DOWN or the target is missing, check the ServiceMonitor labels, the Service port name (`http`), and the NetworkPolicy.

Run these queries in the Prometheus UI (generate traffic first):

```promql
# Requests per second by endpoint and status
sum by (uri, status) (rate(http_server_requests_seconds_count{application="backend"}[1m]))

# p95 latency (works because percentiles-histogram is enabled)
histogram_quantile(0.95, sum by (le) (rate(http_server_requests_seconds_bucket{application="backend"}[5m])))

# Error ratio (what the BackendHigh5xxRate alert uses)
sum(rate(http_server_requests_seconds_count{application="backend",status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count{application="backend"}[5m]))

# JVM heap and DB connection pool
sum by (pod) (jvm_memory_used_bytes{application="backend", area="heap"})
hikaricp_connections_active{application="backend"}

# Kubernetes state (kube-state-metrics)
kube_deployment_status_replicas_available{namespace="app"}
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="app", container!=""}[5m]))
```

**Alerts** tab: your three rules from `backend-alerts.yaml` are listed (green = not firing).

## 15.6 Grafana dashboards

1. **Dashboards → Browse**: the built-in "Kubernetes / Compute Resources / Namespace (Pods)" → namespace `app`.
2. **Dashboards → New → Import** → ID **4701** ("JVM (Micrometer)") → choose the Prometheus data source.
3. Build your own dashboard with 4 panels using the PromQL above: RPS, p95 latency, error %, and pods available. This is the **RED** method (Rate, Errors, Duration) for services. The **USE** method (Utilization, Saturation, Errors) is for nodes and resources.

## 15.7 When to use which (interview answer)

| Need | Tool |
|---|---|
| Pod and container logs, search, retention, audit | Log Analytics (Container Insights) |
| Request traces, dependency map, exceptions | Application Insights |
| Real-time metrics, custom app metrics, flexible PromQL, cheap long-running dashboards | Prometheus + Grafana |
| Alerts on Kubernetes or app metrics | Prometheus Alertmanager (or managed Prometheus alert rules) |
| Alerts on Azure resources (App Gateway, PostgreSQL, node CPU), log queries | Azure Monitor alerts + action groups |

Investigation order for "10% of requests are failing": App Gateway access logs and backend health (is it the edge?) → Prometheus error rate by `uri` and `pod` (one pod or all? one endpoint?) → App Insights failures (exception + SQL dependency) → pod logs and events (restarts, OOM, probes) → PostgreSQL metrics (connections, CPU).

---

[← Part 14 — Monitoring with Azure Monitor](14-azure-monitor.md) | [Index](../README.md) | [Part 16 — Kubernetes exercises →](16-kubernetes-exercises.md)
