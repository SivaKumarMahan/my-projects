# Part 12 — GitHub Actions CI/CD

[← Part 11 — SonarQube Cloud](11-sonarqube-cloud.md) | [Index](../README.md) | [Part 13 — HTTPS and WAF →](13-https-and-waf.md)

---

## 12.1 Application pipeline — `.github/workflows/app-ci-cd.yml`

```yaml
name: app-ci-cd

on:
  pull_request:
    branches: [main]
    paths: ["backend/**", "frontend/**", "helm/**", ".github/workflows/app-ci-cd.yml"]
  push:
    branches: [main]
    paths: ["backend/**", "frontend/**", "helm/**", ".github/workflows/app-ci-cd.yml"]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: app-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }} # never cancel a running deploy

env:
  TRIVY_IMAGE: aquasec/trivy:0.76.1
  GITLEAKS_IMAGE: ghcr.io/gitleaks/gitleaks:v8.30.1
  RESOURCE_GROUP: ${{ vars.RESOURCE_GROUP }}
  ACR_NAME: ${{ vars.ACR_NAME }}
  AKS_NAME: ${{ vars.AKS_NAME }}
  RELEASE: ttapp
  NAMESPACE: app

jobs:
  # ---------------- CI: runs on every PR and every push ----------------
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0 # scan the whole Git history, not just the last commit
      - name: Gitleaks
        run: docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/repo" "$GITLEAKS_IMAGE" git /repo --redact --no-banner

  backend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    env:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0 # Sonar uses Git history for "new code"
      - uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - name: Build and unit test
        run: mvn -B verify
      - name: SonarQube Cloud scan + quality gate
        if: env.SONAR_TOKEN != ''
        run: >
          mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:sonar
          -Dsonar.host.url=https://sonarcloud.io
          -Dsonar.organization=${{ vars.SONAR_ORG }}
          -Dsonar.projectKey=${{ vars.SONAR_ORG }}_backend
          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
          -Dsonar.qualitygate.wait=true
      - name: Trivy dependency + secret scan
        # --offline-scan + the Maven cache: no extra calls to Maven Central (avoids HTTP 429).
        run: >
          docker run --rm -v "$PWD:/src" -v "$HOME/.m2:/root/.m2" "$TRIVY_IMAGE"
          fs --offline-scan --scanners vuln,secret --severity HIGH,CRITICAL --ignore-unfixed
          --exit-code 1 --skip-dirs /src/target /src

  frontend:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    env:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v7
        with:
          node-version: "24"
          cache: npm
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci --no-audit --no-fund
      - run: npm run build
      - name: SonarQube Cloud scan + quality gate
        if: env.SONAR_TOKEN != ''
        uses: SonarSource/sonarqube-scan-action@v8
        with:
          projectBaseDir: frontend
          args: >
            -Dsonar.host.url=https://sonarcloud.io
            -Dsonar.organization=${{ vars.SONAR_ORG }}
            -Dsonar.projectKey=${{ vars.SONAR_ORG }}_frontend
            -Dsonar.sources=src
            -Dsonar.qualitygate.wait=true
      - name: Trivy dependency + secret scan
        run: >
          docker run --rm -v "$PWD:/src" "$TRIVY_IMAGE"
          fs --scanners vuln,secret --severity HIGH,CRITICAL --ignore-unfixed
          --exit-code 1 --skip-dirs /src/node_modules /src

  helm-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: helm lint
        run: helm lint helm/three-tier-app --set azure.tenantId=x,azure.keyVaultName=x,azure.backendClientId=x
      - name: Trivy misconfiguration scan
        run: >
          docker run --rm -v "$PWD:/src" -w /src "$TRIVY_IMAGE"
          config --severity HIGH,CRITICAL --exit-code 1 --ignorefile /src/.trivyignore
          --helm-set azure.tenantId=x,azure.keyVaultName=x,azure.backendClientId=x
          /src/helm/three-tier-app

  # ---------------- Build, scan and push images (main branch only) ----------------
  build-push:
    needs: [secret-scan, backend, frontend, helm-scan]
    if: github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write # needed to request the OIDC token for azure/login
    strategy:
      matrix:
        app: [frontend, backend]
    env:
      IMAGE: ${{ vars.ACR_NAME }}.azurecr.io/ttapp-${{ matrix.app }}:${{ github.sha }}
    steps:
      - uses: actions/checkout@v7
      - uses: azure/login@v3
        with:
          client-id: ${{ vars.AZURE_APP_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - name: Log in to ACR
        run: az acr login --name "$ACR_NAME"
      - uses: docker/setup-buildx-action@v4
      - name: Build image (not pushed yet)
        uses: docker/build-push-action@v7
        with:
          context: ${{ matrix.app }}
          load: true # keep it in the local Docker daemon so Trivy can scan it
          push: false
          tags: ${{ env.IMAGE }}
          cache-from: type=gha,scope=${{ matrix.app }}
          cache-to: type=gha,mode=max,scope=${{ matrix.app }}
      - name: Trivy image scan (blocks the push on HIGH/CRITICAL)
        run: >
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock "$TRIVY_IMAGE"
          image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 "$IMAGE"
      - name: Push image
        run: docker push "$IMAGE"

  # ---------------- Deploy to the private AKS cluster ----------------
  deploy-dev:
    needs: build-push
    runs-on: [self-hosted, linux, aks-deployer] # the runner VM inside the hub VNet
    environment: dev
    permissions:
      contents: read
      id-token: write
    env:
      KUBECONFIG: ${{ github.workspace }}/.kubeconfig
    steps:
      - uses: actions/checkout@v7
      - uses: azure/login@v3
        with:
          client-id: ${{ vars.AZURE_APP_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - name: Read values from Azure
        run: |
          {
            echo "KV_NAME=$(az keyvault list -g "$RESOURCE_GROUP" --query '[0].name' -o tsv)"
            echo "BACKEND_CLIENT_ID=$(az identity show -g "$RESOURCE_GROUP" -n id-ttapp-dev-backend --query clientId -o tsv)"
            echo "TENANT_ID=$(az account show --query tenantId -o tsv)"
            echo "APPGW_IP=$(az network public-ip show -g "$RESOURCE_GROUP" -n pip-ttapp-dev-appgw --query ipAddress -o tsv)"
          } >> "$GITHUB_ENV"
      - name: Get kubeconfig (Entra ID auth)
        run: |
          az aks get-credentials -g "$RESOURCE_GROUP" -n "$AKS_NAME" --file "$KUBECONFIG" --overwrite-existing
          kubelogin convert-kubeconfig -l azurecli --kubeconfig "$KUBECONFIG"
      - name: Helm upgrade (auto-rollback on failure)
        run: |
          helm upgrade --install "$RELEASE" helm/three-tier-app \
            --namespace "$NAMESPACE" \
            --rollback-on-failure --wait --timeout 10m --history-max 10 \
            --set image.registry="$ACR_NAME.azurecr.io" \
            --set image.tag="${{ github.sha }}" \
            --set azure.tenantId="$TENANT_ID" \
            --set azure.keyVaultName="$KV_NAME" \
            --set azure.backendClientId="$BACKEND_CLIENT_ID" \
            --set serviceMonitor.enabled="${{ vars.SERVICE_MONITOR_ENABLED || 'false' }}" \
            --set ingress.tls.enabled="${{ vars.TLS_ENABLED || 'false' }}" \
            --set ingress.tls.appgwSslCertificate=ttapp-selfsigned
      - name: Smoke test through App Gateway
        run: |
          for i in $(seq 1 20); do
            if curl -fsS "http://$APPGW_IP/api/info"; then echo; echo "Smoke test passed"; exit 0; fi
            echo "waiting for App Gateway... ($i)"; sleep 15
          done
          echo "Smoke test failed"; exit 1
      - name: Debug info on failure
        if: failure()
        run: |
          kubectl -n "$NAMESPACE" get pods,svc,ingress -o wide || true
          kubectl -n "$NAMESPACE" get events --sort-by=.lastTimestamp | tail -30 || true
          helm -n "$NAMESPACE" history "$RELEASE" || true
```

## 12.2 Infrastructure pipeline — `.github/workflows/infra.yml`

````yaml
name: infra

on:
  pull_request:
    branches: [main]
    paths: ["infra/**", ".github/workflows/infra.yml"]
  workflow_dispatch:
    inputs:
      stack:
        description: Which Terraform stack
        type: choice
        options: [foundation, platform]
        default: platform
      action:
        description: What to do
        type: choice
        options: [plan, apply, destroy]
        default: plan

permissions:
  contents: read

concurrency:
  group: infra-${{ github.event.inputs.stack || 'pr' }} # never two applies on the same state
  cancel-in-progress: false

env:
  TF_VERSION: "1.17.0"
  TRIVY_IMAGE: aquasec/trivy:0.76.1
  # Terraform's azurerm provider + backend use these for OIDC login (no client secret).
  ARM_USE_OIDC: "true"
  ARM_USE_AZUREAD: "true"
  ARM_CLIENT_ID: ${{ vars.AZURE_INFRA_CLIENT_ID }}
  ARM_TENANT_ID: ${{ vars.AZURE_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}
  TF_VAR_my_ip: ${{ secrets.TF_VAR_MY_IP }}
  TF_VAR_alert_email: ${{ secrets.TF_VAR_ALERT_EMAIL }}
  TF_IN_AUTOMATION: "true"

jobs:
  # Static checks: no Azure access needed, safe on GitHub-hosted runners.
  validate:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        stack: [foundation, platform]
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}
      - run: terraform fmt -check -recursive
        working-directory: infra/${{ matrix.stack }}
      - run: terraform init -backend=false -input=false
        working-directory: infra/${{ matrix.stack }}
      - run: terraform validate
        working-directory: infra/${{ matrix.stack }}
      - name: Trivy IaC scan
        run: >
          docker run --rm -v "$PWD:/src" -w /src "$TRIVY_IMAGE"
          config --severity HIGH,CRITICAL --exit-code 1 --ignorefile /src/.trivyignore
          /src/infra/${{ matrix.stack }}

  # Plan on pull requests (same-repo PRs only: fork PRs must never reach the self-hosted runner).
  plan-pr:
    needs: validate
    if: github.event_name == 'pull_request' && github.event.pull_request.head.repo.full_name == github.repository
    runs-on: [self-hosted, linux, aks-deployer]
    permissions:
      contents: read
      id-token: write
    strategy:
      max-parallel: 1
      matrix:
        stack: [foundation, platform]
    defaults:
      run:
        working-directory: infra/${{ matrix.stack }}
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false
      - run: terraform init -input=false -backend-config=backend.hcl
      - name: terraform plan
        run: |
          terraform plan -input=false -lock-timeout=5m -var-file=dev.tfvars -no-color -out=tfplan | tee plan.txt
          {
            echo "### Plan for \`${{ matrix.stack }}\`"
            echo '```'
            grep -E '^(Plan:|No changes)' plan.txt || true
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"

  # Manual run: plan / apply / destroy one stack. "dev" environment can require your approval.
  run:
    needs: validate
    if: github.event_name == 'workflow_dispatch'
    runs-on: [self-hosted, linux, aks-deployer]
    environment: dev
    permissions:
      contents: read
      id-token: write
    defaults:
      run:
        working-directory: infra/${{ inputs.stack }}
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false
      - run: terraform init -input=false -backend-config=backend.hcl
      - name: terraform plan
        if: inputs.action != 'destroy'
        run: terraform plan -input=false -lock-timeout=5m -var-file=dev.tfvars -out=tfplan
      - name: terraform apply
        if: inputs.action == 'apply'
        run: terraform apply -input=false -auto-approve tfplan
      - name: terraform destroy
        if: inputs.action == 'destroy'
        run: terraform destroy -input=false -auto-approve -lock-timeout=5m -var-file=dev.tfvars
````

## 12.3 What happens, stage by stage

```text
Pull request ─► secret-scan ─┐
             ─► backend ─────┤ (build, unit test, JaCoCo, SonarQube gate, Trivy fs)
             ─► frontend ────┤ (npm ci, build, SonarQube gate, Trivy fs)
             ─► helm-scan ───┘ (helm lint, Trivy config)
                               all must pass to merge (branch ruleset)

Push to main ─► same CI jobs ─► build-push (matrix: frontend, backend)
                                  azure/login (OIDC) → buildx (GHA cache) → Trivy image → push :<git sha>
                               ─► deploy-dev  [waits for YOUR approval: environment "dev"]
                                  self-hosted runner in hub VNet → az login (OIDC) → kubeconfig (kubelogin)
                                  → helm upgrade --rollback-on-failure → smoke test through App Gateway
```

Design decisions to explain:

- **Image tag = Git commit SHA**, never `latest`. Every running pod points to an exact commit, and rollback means "deploy the previous SHA".
- **Scan before push.** A vulnerable image never reaches the registry.
- **Build once, deploy many.** QA and Prod would deploy the **same** image and digest that passed dev, promoted rather than rebuilt (Appendix B).
- **`concurrency`.** PR runs cancel older runs of the same PR; deploys are never cancelled halfway.
- **`permissions`** is read-only by default; `id-token: write` only on the jobs that log in to Azure.
- **Fork safety.** Fork PRs never reach the self-hosted runner (`head.repo.full_name == github.repository`), plus the repo setting from Part 2.4.
- **Pin versions** (`aquasec/trivy:0.76.1`, `actions/checkout@v7`). For stronger supply-chain security, pin actions to a **commit SHA**. Tag hijacking has happened in real attacks; the `tj-actions/changed-files` compromise in March 2025 is the well-known example.
- **Ways a GitHub workflow can be triggered:** `push`, `pull_request`, `workflow_dispatch` (manual button), `schedule` (cron), `workflow_run` (after another workflow), `repository_dispatch` (API call), `release`. This answers "in how many ways can a pipeline run?"

## 12.4 Run it

```bash
cd ~/three-tier-aks
git add .github && git commit -m "CI/CD workflows" && git push
```

1. **Actions → app-ci-cd**: the CI jobs run on GitHub-hosted runners, then `build-push`.
2. `deploy-dev` shows **"Waiting for review"**. Click **Review deployments → Approve**.
3. The job runs on `jumpbox-runner`. Watch the Helm output and the smoke test.
4. On the jumpbox: `helm history ttapp -n app` shows revision 2 with the Git SHA image.

Now practise the **PR flow**:

```bash
git checkout -b feature/title-length
# change @Size(max = 200) to @Size(max = 100) in TodoController.java
git commit -am "Limit title to 100 chars" && git push -u origin feature/title-length
# open a PR on GitHub: CI runs, SonarQube posts its result, merge only when green
```

After the first run, set up the **branch ruleset** from Part 2.4 (the check names now exist).

## 12.5 Run Terraform from the pipeline

**Actions → infra → Run workflow → stack: `platform`, action: `plan`**. It runs on the self-hosted runner. Approve the `dev` environment, then read the plan in the log. It should show **No changes** if nothing has drifted.

From now on you can create and destroy the platform stack **from GitHub** (`apply` / `destroy`) instead of from your laptop. Interview line: *"Terraform runs only in the pipeline: PRs get a plan, and apply needs approval through the environment protection rule. State is in Azure Storage with blob-lease locking, and the pipeline logs in with OIDC."*

**Interview question — "the whole company shares one state file; how do multiple people work on it?"** Remote backend + **state locking** (blob lease) lets only one plan or apply run at a time; the other waits (`-lock-timeout=5m`) or fails with the lock ID. Better still: **split the state** by component and environment (as we did: foundation and platform), so teams rarely touch the same state.

---

[← Part 11 — SonarQube Cloud](11-sonarqube-cloud.md) | [Index](../README.md) | [Part 13 — HTTPS and WAF →](13-https-and-waf.md)
