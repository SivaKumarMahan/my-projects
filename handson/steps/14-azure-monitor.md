# Part 14 — Monitoring with Azure Monitor

[← Part 13 — HTTPS and WAF](13-https-and-waf.md) | [Index](../README.md) | [Part 15 — Monitoring with Prometheus and Grafana →](15-prometheus-and-grafana.md)

---

## 14.1 How the pieces fit (interview answer)

```text
AKS nodes/pods ──(ama-logs agent, DCR)──► Log Analytics: ContainerLogV2, KubePodInventory, KubeEvents, Perf...
AKS control plane ──(diagnostic setting)──► Log Analytics: AKSAuditAdmin, AKSControlPlane
App Gateway ──(diagnostic setting)──► Log Analytics: AGWAccessLogs, AGWFirewallLogs
Spring Boot ──(App Insights Java agent)──► Application Insights ──► same workspace: AppRequests, AppDependencies, AppExceptions
                                                    │
Metric alert (node CPU) / Log alert (CrashLoopBackOff KQL) ──► Action group ──► email
```

- **Azure Monitor** is the umbrella service.
- **Log Analytics** stores logs; you query them with **KQL**.
- **Container Insights** collects Kubernetes logs and inventory.
- **Application Insights** collects application telemetry: requests, dependencies (SQL calls), exceptions, the application map, and live metrics. The Java agent does this **without code changes**.

## 14.2 Generate some traffic first

```bash
APPGW_IP=$(az network public-ip show -g $RG -n pip-ttapp-dev-appgw --query ipAddress -o tsv)
for i in $(seq 1 300); do curl -s -o /dev/null http://$APPGW_IP/api/todos; curl -s -o /dev/null http://$APPGW_IP/api/nope; done
```

## 14.3 Container Insights

Portal → AKS cluster → **Monitoring → Insights**: Cluster, Nodes, Controllers and Containers tabs. Then **Logs**:

```kusto
// Errors in application logs (last hour)
ContainerLogV2
| where TimeGenerated > ago(1h) and PodNamespace == "app"
| where LogMessage has_any ("ERROR", "Exception")
| project TimeGenerated, PodName, ContainerName, LogMessage
| order by TimeGenerated desc
```

```kusto
// Latest status and restart count of every pod in "app"
KubePodInventory
| where TimeGenerated > ago(30m) and Namespace == "app"
| summarize arg_max(TimeGenerated, PodStatus, ContainerRestartCount, Node = Computer) by Name
```

```kusto
// Kubernetes warning events (FailedScheduling, BackOff, FailedMount, Unhealthy...)
KubeEvents
| where TimeGenerated > ago(1h) and Namespace == "app" and KubeEventType == "Warning"
| project TimeGenerated, Name, Reason, Message
| order by TimeGenerated desc
```

```kusto
// Who changed what in the cluster (audit, no read calls)
AKSAuditAdmin
| where TimeGenerated > ago(1d)
| project TimeGenerated, User = tostring(User.username), Verb, ObjectRef = strcat(ObjectRef.resource, "/", ObjectRef.name)
| order by TimeGenerated desc
```

## 14.4 Application Insights

Portal → **appi-ttapp-dev**:

- **Application map**: `backend` → PostgreSQL (`appdb`), with call counts and latency.
- **Failures**: your `/api/nope` calls show as 404s; drill into one request.
- **Performance**: slowest operations and their SQL dependencies.
- **Live metrics**: real-time requests while you run the loop above.

The same data in KQL (workspace tables):

```kusto
AppRequests
| where TimeGenerated > ago(1h)
| summarize count(), avg(DurationMs), percentile(DurationMs, 95) by Name, ResultCode
| order by count_ desc
```

## 14.5 Alerts

Terraform created two alerts: **node CPU > 80%** (metric alert) and **CrashLoopBackOff** (log alert, KQL every 5 minutes). Look at them in Portal → Monitor → **Alerts → Alert rules**. You trigger the CrashLoop alert for real in [Lab 2](17-break-and-fix-lab.md#lab-2--crashloopbackoff) and get an email.

**Metric vs log alerts:** metric alerts are near real-time and cheap, but only cover numbers Azure already collects. Log alerts run any KQL query, which is more flexible, but have a delay (evaluation frequency + ingestion latency) and cost per rule.

---

[← Part 13 — HTTPS and WAF](13-https-and-waf.md) | [Index](../README.md) | [Part 15 — Monitoring with Prometheus and Grafana →](15-prometheus-and-grafana.md)
