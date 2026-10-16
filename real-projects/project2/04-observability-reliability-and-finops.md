# 04 — Observability, Reliability and FinOps

A monitoring repository holds several small services and tools, plus Lambda-based checkers in a separate functions repository.

## Overview

| Project | What it does | Angle |
|---|---|---|
| Ingest monitor | Reads per-feed YAML cadence files, lists S3, flags feeds that are `missing_past_due`, dedupes alerts with an SSM re-arm policy. Runs as a scheduled ECS task. | Schedule-aware SLA monitoring, alert-fatigue control |
| Alerting | Splunk alerts defined in YAML. A CLI renders `savedsearches.conf`, generates a markdown catalog and syncs through the Splunk REST API. Runbook included. | Alerts as code |
| Cost observability | Daily Databricks cost report as a PDF email on a cron workflow, with anomaly config, suppressions, savings tracking and SQL. | FinOps with real findings |
| Credential inventory | CLI listing Databricks and AWS credentials that are about to expire. | Secrets lifecycle |
| Email intake service | Python poller (Microsoft Graph → issue tracker) on ECS Fargate. Uses create-metadata to send only allowed fields; moves each mail to Success or Failure. | Event-driven integration, Dockerized service |
| Observability foundation | Notebooks and SQL for telemetry. | Telemetry design |
| Reporting | Reports built on that telemetry. | Reporting |
| AI skills | Claude skills for Splunk and Databricks alert-noise cleanup and daily alert reports. | Reducing alert fatigue with AI |

Each service has its own deploy workflow (ingest monitor sweep, intake service, vendor list sync).

## Ingest SLA monitor

```
feeds/*.yaml  (feed, S3 prefix, expected cadence, grace period)
       │
Scheduled ECS task ──► list S3 objects per feed ──► latest arrival vs expected schedule
       │
       ├─ on time                → clear state
       └─ missing_past_due       → alert once
                                     │
                         SSM parameter stores "alerted" state
                         re-arm only after the feed recovers (or after a re-arm window)
```

- **Schedule-aware:** a daily feed isn't "late" at 09:00 if it's due at 10:00 plus grace.
- **Dedupe / re-arm:** one alert per incident instead of one per run — prevents alert storms.

```yaml
# Illustrative feed definition
feed: daily-viewership
s3_prefix: s3://<bucket>/incoming/viewership/
cadence: daily
expected_by: "10:00"
timezone: America/New_York
grace_minutes: 60
```

## Alerts as code (Splunk)

- Each alert is a YAML file: search, schedule, threshold, severity, recipients, runbook link.
- CLI renders `savedsearches.conf`, regenerates a **markdown catalog** of every alert, and **syncs via REST** so Splunk matches git.
- Changes go through pull requests like any other code.

## Credential and certificate expiry monitor 🟡

A daily Lambda with **six collectors** — ACM certificates, IAM access keys, SSM-stored secrets, Databricks service principal secrets and PATs, and others.

- Emits **one event per credential per run**, plus a **heartbeat** event.
- A credential that **can't be evaluated emits `UNKNOWN`**, never silence.
- Unit-test file per collector; `deploy.sh` for deployment.

**Key principle:** *absence of data must itself be an alert.* If the heartbeat stops, or a collector returns `UNKNOWN`, that's visible — the monitor can't fail silently. This is the monitoring-of-monitoring answer SRE interviews look for.

```python
# Illustrative event shape
{"credential": "acm:api-cert", "status": "OK|WARN|EXPIRED|UNKNOWN",
 "days_left": 21, "run_id": "...", "collector": "acm"}
```

## FinOps: daily Databricks cost report

- Scheduled GitHub Actions workflow builds a PDF and emails it daily.
- Anomaly configuration, suppressions for known spikes, savings tracking over time.
- **Root-cause finding:** about **65% of spend was idle SQL warehouses** — fixed with auto-stop and right-sizing.
- Workflow hygiene: **concurrency group** (no overlapping runs), **job timeout**, and an **OIDC role trusted only for `refs/heads/main`**.

```yaml
on:
  schedule: [{ cron: "0 12 * * *" }]
concurrency: { group: daily-cost-report, cancel-in-progress: false }
jobs:
  report:
    timeout-minutes: 30
    permissions: { id-token: write, contents: read }
```

## AI tool usage reporting

Lambdas report usage and cost for AI tools (Claude and OpenAI) with tests, plus a license usage audit — FinOps applied to LLM spend.

## Incident / RCA write-ups

Post-fix write-ups exist for a telemetry checkpoint bug and a node-timeline CPU issue — use them as incident stories (symptom → investigation → root cause → fix → prevention).

## Interview Q&A

**How do you reduce alert fatigue?**
Schedule-aware thresholds, dedupe with explicit re-arm, alert catalog reviews, routing by severity, and deleting alerts nobody acts on.

**How do you know your monitoring is working?**
Heartbeats, `UNKNOWN` states that alert, and synthetic checks — silence is treated as a failure.

**Give a FinOps example.**
The daily report showed idle warehouses were ~65% of spend; auto-stop and sizing changes addressed it, and the report tracks savings since. *(Only quote the number if you can explain how it was measured.)*
