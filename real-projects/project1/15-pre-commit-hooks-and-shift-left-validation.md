# 15 — Pre-commit Hooks and Shift-Left Validation

Every commit to the IaC repo is checked locally before it reaches a pipeline. The hooks cover Bicep formatting and linting, Helm rendering, Kubernetes schema validation and security scanning.

**Files:** `hooks/pre-commit/.pre-commit-config.yaml`, `hooks/pre-commit/bicep_lint.py`, `hooks/pre-commit/helm_validate.py`, `helm/charts.yaml`, `Makefile`, `pyproject.toml`

## What runs on each commit

| Hook | Source | What it catches |
|---|---|---|
| `trailing-whitespace`, `end-of-file-fixer`, `check-merge-conflict`, `check-added-large-files` (1 MB), `check-yaml/json/toml/xml`, `pretty-format-json`, `check-symlinks`, `requirements-txt-fixer` | `pre-commit/pre-commit-hooks` v6.0.0 | Hygiene, broken YAML/JSON, merge markers, large binaries |
| **Checkov** | `bridgecrewio/checkov` 3.2.495 | IaC misconfigurations across Bicep, Kubernetes and Terraform |
| **bicep-lint** (custom Python) | `hooks/pre-commit/bicep_lint.py` | Checks the Azure CLI is present, installs Bicep if missing, runs `az bicep format` and `az bicep lint` on staged `.bicep` / `.bicepparam` files |
| **helm-validate** (custom Python) | `hooks/pre-commit/helm_validate.py` | For each **chart × environment × service** in `helm/charts.yaml`: `helm template` → **kubeconform** → **checkov --framework kubernetes** |

## How helm-validate works

```
helm/charts.yaml  (charts: wms, rbac × envs: sandbox/qlab/performance/production × services)
        │
        ▼
helm template <chart> -f values.yaml -f values/env/<env>/values.yaml -f values/<svc>/values.yaml
        │  rendered once, written to a temp file
        ├──► kubeconform  (rendered manifests vs Kubernetes OpenAPI schemas)
        └──► checkov --framework kubernetes  (security misconfigurations)
```

### Design details

- **Config-driven matrix.** `helm/charts.yaml` lists charts, environments and services. Adding an environment is a YAML edit, not a code change.
- **Render once, validate many times.** The `helm template` output feeds both kubeconform and Checkov.
- **Graceful degradation.** If kubeconform or Checkov is not installed, the hook warns and skips that step instead of blocking the commit.
- **`check-yaml` excludes `helm/*/templates/`**, because Go-templated files are not valid YAML until rendered.
- **Helm `values.schema.json`** for the `rbac` and `wms` charts, so Helm itself rejects bad values. For example, `verbs` is an enum (`get, list, watch, create, update, patch, delete, deletecollection`) and `environment` is an enum of the four environments.

```json
{
  "properties": {
    "environment": { "type": "string", "enum": ["sandbox", "qlab", "performance", "production"] },
    "rules": {
      "type": "array",
      "items": {
        "properties": {
          "verbs": {
            "type": "array",
            "items": { "enum": ["get","list","watch","create","update","patch","delete","deletecollection"] }
          }
        }
      }
    }
  }
}
```

## Pinned toolchain

| Tool | Version | Installed by |
|---|---|---|
| pre-commit | 4.5.0 | Poetry |
| azure-cli | 2.81.0 | Poetry |
| checkov | 3.2.495 | Poetry |
| pyyaml | 6.0.3 | Poetry |
| Helm | v4 | Makefile |
| kubeconform | v0.7.0 | Makefile |

## Developer tooling

- **Poetry** (`package-mode = false`, dependencies only), `poetry.toml` (in-project venv, cache dirs) and `.python-version` for a reproducible toolchain.
- **Makefile targets:**

```make
install-pyenv            # pyenv for the pinned Python
venv                     # in-project virtualenv
install                  # pip → Poetry → Helm v4 → kubeconform → poetry install → pre-commit install
pre-commit-install
pre-commit-all-files
pre-commit-staged-files
pre-commit-autoupdate
clean
```

- **`git-add-commit-push.sh`** wraps setup, hooks, commit and push. Flags: `--no-verify`, `--no-push`, `--setup-only`, `--skip-setup`.
- **`.editorconfig`** for consistent indentation; **`.gitignore`** for logs, `.venv`, Checkov `.sarif` output.
- **Git workflow:** short-lived feature branches and release branches, merged through Azure Repos pull requests with review.

## Interview Q&A

**Pre-commit vs CI checks?**
Pre-commit gives fast local feedback, but anyone can skip it with `--no-verify`. CI is the enforced gate. Run the same config in the pipeline with `pre-commit run --all-files`.

**kubeconform vs `helm lint`?**
`helm lint` checks chart structure. kubeconform validates the **rendered** manifests against Kubernetes API schemas, so it catches wrong fields and removed `apiVersion`s.

**What does Checkov catch in Kubernetes?**
Missing resource limits, privileged containers, `latest` image tags, missing probes, containers running as root, and similar problems.
