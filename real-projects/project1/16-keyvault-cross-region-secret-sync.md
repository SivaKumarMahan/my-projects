# 16 — Key Vault Cross-Region Secret Sync and Alerting as Code

## Problem

DR and multi-region setups need the same secrets in a secondary Key Vault (for example West US 2 → a secondary region). Copying them by hand drifts and gets forgotten.

**Files:** `azure/bicep/stacks/wms/keyvaultSync/` (`main.bicep`, `function_app.py`, `function.sh`, `README.md`) and the modules `eventGrid`, `functionApp`, `diagnosticSettings`, `logMetricAlert`.

## Architecture (event-driven)

```
Primary Key Vault ──(Event Grid system topic)──► Event subscription ──► Azure Function (Python, Flex Consumption)
   SecretNewVersionCreated / NearExpiry / Expired / Deleted        │              │
                                                       dead-letter ▼              ▼ set_secret()
                                                    Storage blob (DLQ)      Secondary Key Vault
```

## The function

- Python, `@app.event_grid_trigger`.
- Reads the secret version from the primary vault with `DefaultAzureCredential` (managed identity) and writes it to the secondary.
- **Preserves metadata:** `content_type`, `tags`, `enabled`, `expires_on`, `not_before`.
- Handles `403` (RBAC problem) and `404` (secret gone) separately, and logs a warning on near-expiry events.
- **Re-raises on failure** so Event Grid retries with backoff and finally dead-letters the event.

```python
import azure.functions as func
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
from azure.core.exceptions import HttpResponseError, ResourceNotFoundError

app = func.FunctionApp()
cred = DefaultAzureCredential()

@app.event_grid_trigger(arg_name="event")
def KeyVaultSyncFunction(event: func.EventGridEvent):
    data = event.get_json()
    name, version = data["ObjectName"], data.get("Version")
    if event.event_type.endswith("SecretNearExpiry"):
        logging.warning("Secret %s is near expiry", name)
    try:
        src = SecretClient(PRIMARY_URL, cred).get_secret(name, version)
        SecretClient(SECONDARY_URL, cred).set_secret(
            name, src.value,
            content_type=src.properties.content_type,
            tags=src.properties.tags,
            enabled=src.properties.enabled,
            expires_on=src.properties.expires_on,
            not_before=src.properties.not_before,
        )
    except ResourceNotFoundError:
        logging.error("Secret %s not found", name); raise
    except HttpResponseError as e:
        if e.status_code == 403:
            logging.error("RBAC denied for %s", name)
        raise   # let Event Grid retry, then dead-letter
```

## Least privilege

| Identity | Vault | Role |
|---|---|---|
| Function system-assigned identity | Primary | **Key Vault Secrets User** (read) |
| Function system-assigned identity | Secondary | **Key Vault Secrets Officer** (write) |

## Infra and code ship in one stack deploy

- The `.bicepparam` file uses `loadTextContent()` on `function_app.py`, `host.json` and `requirements.txt` and passes them as environment variables to a **Deployment Script**.
- The script rebuilds the files, zips them, uploads the zip to blob storage with `--auth-mode login` (no storage keys), and runs `az functionapp deployment source config-zip`.
- **Flex Consumption** plan (`FC1`) with blob-container deployment storage and identity-based auth.
- **Dead-letter** destination on the Event Grid subscription is a storage container, so no event is lost silently.

## Monitoring and alerting as code

**File:** `azure/bicep/modules/logMetricAlert/logMetricAlert.bicep`

- **Action Group** with an email receiver and the common alert schema.
- **Scheduled Query Rule** (log alert) over Log Analytics:

```kusto
AppRequests
| where Name == "KeyVaultSyncFunction" and Success == false
| summarize Failures = count() by bin(TimeGenerated, 5m)
```

| Setting | Value |
|---|---|
| Severity | 2 |
| Evaluation frequency / window | `PT5M` / `PT5M` |
| `autoMitigate` | true |
| Failing periods | configured to avoid one-off noise |

- **Diagnostic settings** send `FunctionAppLogs` and `AllMetrics` to Log Analytics.
- Across the platform the same pattern is used for the Application Insights and Log Analytics stacks, Container Insights on AKS and Defender for Containers.

## Interview Q&A

**Why Event Grid instead of a timer?**
Push-based and near real-time, with no polling cost and no window where secrets are out of sync.

**What happens if the function fails?**
Event Grid retries with exponential backoff, then dead-letters to blob, and the KQL alert fires.

**What are Event Grid's delivery guarantees?**
At-least-once. That is why the sync is idempotent: `set_secret` just creates a new version with the same value if an event is delivered twice.
