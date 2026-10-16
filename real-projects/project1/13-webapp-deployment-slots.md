# 13 — App Service Web Apps: Deployment Slots and Slot Swap

A separate containerized **Conveyor** web application runs on Azure App Service (Web App for Containers) in two production instances, each with a **staging** slot.

## Azure DevOps setup

| Item | Details |
|---|---|
| Repo | Source, `pipelines/azure-pipeline.yml`, release script `releases/scripts/conveyer/global/manage-webapp.sh` (start/stop, config updates, slot management) |
| Build pipeline | Builds code, packages and publishes the artifact |
| Release pipelines | One per production web app |
| Artifact | From the build pipeline, default version **Latest** |
| Variable groups | One per web app |
| Service connections | One ARM connection per web app resource group |

### Release stages (per web app)

1. **Deploy to staging slot** — Azure CLI (shell) task runs `manage-webapp.sh` with app name, resource group and slot.
2. **Swap slots (staging → production)** — Azure CLI task, same script, after approval.

```bash
az webapp deployment slot swap \
  --name <webapp> --resource-group <rg> \
  --slot staging --target-slot production
```

Service connections need **Contributor** on the App Service.

## Azure configuration

- **Production slot** — 100% of traffic. **Staging slot** — 0% traffic, used to validate the new release.
- Both slots share one **App Service plan** (no extra cost for the staging environment).
- **Deployment Center** — container from ACR; image pulls authenticated with a **managed identity** (`AcrPull` on the registry).

### Setting up slots from scratch

1. Create a `staging` slot on the web app.
2. Release pipeline artifact: Azure Container Registry → image → default tag `release`.
3. Stage 1 — *Azure App Service deploy*: app type *Web App for Containers*, deploy to slot ✅, slot `staging`, image `<acr>.azurecr.io/<image>:release` (optionally use the managed identity for ACR).
4. Stage 2 — manual approval, then Azure CLI slot swap.

## Investigation — why environment variables changed after a swap

### Test environment

```bash
# Registry + test images
az acr login --name <testacr>
docker pull nginx && docker tag nginx <testacr>.azurecr.io/nginx:1 && docker push <testacr>.azurecr.io/nginx:1
docker pull httpd && docker tag httpd <testacr>.azurecr.io/httpd:1 && docker push <testacr>.azurecr.io/httpd:1
```

- User-assigned managed identity with **AcrPull** on the registry.
- Web app (Premium v3 P0V3) + `staging` slot; identity attached to the staging slot.

```bash
az webapp config container set \
  --name <webapp> --resource-group <rg> --slot staging \
  --docker-custom-image-name <testacr>.azurecr.io/httpd:1 \
  --docker-registry-server-url https://<testacr>.azurecr.io

az webapp deployment slot swap --name <webapp> --resource-group <rg> \
  --slot staging --target-slot production
```

Production and staging URLs were checked before and after the swap (nginx ↔ httpd moved as expected).

### Test case 1 — deploying to the staging slot

Pushed `httpd:2` and deployed it to staging. **Result:** environment variables are untouched by a slot deployment.

### Test case 2 — swapping with sticky and non-sticky settings

| | Production (before) | Staging (before) |
|---|---|---|
| Sticky | `PROD_SLOT_STICKY=prodsticky` | `STAGE_SLOT_STICKY=stagesticky` |
| Non-sticky | `PROD_SLOT=prod`, `SLOT=prod` | `STAGE_SLOT=stages`, `SLOT=staging` |

After swap:

| | Production (after) | Staging (after) |
|---|---|---|
| Sticky | `PROD_SLOT_STICKY=prodsticky` (stayed) | `STAGE_SLOT_STICKY=stagesticky` (stayed) |
| Non-sticky | `STAGE_SLOT=stages`, `SLOT=staging` (moved in) | `PROD_SLOT=prod`, `SLOT=prod` (moved in) |

### Conclusion

1. Deploying to staging does **not** change environment variables.
2. Swapping **does** swap environment variables between slots.
3. A setting with **"Deployment slot setting"** checked is **sticky** — it stays with the slot.
4. Unchecked settings are **non-sticky** — they travel with the app during the swap.
5. Slot-specific values (`ENVIRONMENT`, `SLOT`, `DB_USERNAME`, `DB_PASSWORD`, connection strings) **must be sticky**.
6. Values common to both slots (`ACR_SERVER_URL`, registry credentials) can be non-sticky.

The production issue was caused by slot-specific settings that were not marked as deployment slot settings; marking them sticky fixed it.
