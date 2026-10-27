# Part 9 — The Helm chart

[← Part 8 — Terraform stack 2: platform (AKS, App Gateway, PostgreSQL)](08-terraform-platform.md) | [Index](../README.md) | [Part 10 — First deployment by hand →](10-first-deployment.md)

---

Create `helm/three-tier-app/` with these files.

## 9.1 `Chart.yaml` and `values.yaml`

```yaml
apiVersion: v2
name: three-tier-app
description: React frontend + Spring Boot backend on AKS (database is Azure PostgreSQL)
type: application
version: 0.1.0      # chart version: bump when templates change
appVersion: "1.0.0" # informational; the real image tag is passed with --set
```

```yaml
image:
  registry: acrttappsk01.azurecr.io # overridden by the pipeline
  tag: latest                       # overridden by the pipeline with the Git commit SHA
  pullPolicy: IfNotPresent

# Filled by the pipeline from Azure (az identity show / az keyvault show / az account show).
azure:
  tenantId: ""
  keyVaultName: ""
  backendClientId: ""

frontend:
  repository: ttapp-frontend
  replicas: 2
  port: 8080
  resources:
    requests: { cpu: 50m, memory: 64Mi }
    limits: { memory: 128Mi }

backend:
  repository: ttapp-backend
  replicas: 2
  port: 8080
  serviceAccountName: backend # must match the federated credential subject in Terraform
  javaToolOptions: "-XX:MaxRAMPercentage=75 -javaagent:/app/applicationinsights-agent.jar"
  resources:
    requests: { cpu: 250m, memory: 512Mi }
    limits: { memory: 768Mi }
  hpa:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
    targetCPUUtilizationPercentage: 70

# Key Vault secret name -> environment variable in the backend container.
keyVaultSecrets:
  db-url: DB_URL
  db-username: DB_USERNAME
  db-password: DB_PASSWORD
  appinsights-connection-string: APPLICATIONINSIGHTS_CONNECTION_STRING

ingress:
  enabled: true
  className: azure-application-gateway
  host: ""                 # empty = any host (use the App Gateway public IP)
  tls:
    enabled: false
    appgwSslCertificate: "" # name of the certificate uploaded to App Gateway (Part 10)

networkPolicy:
  enabled: true
  appGatewaySubnetCidr: 10.1.0.0/24
  privateEndpointSubnetCidr: 10.1.1.0/24
  monitoringNamespace: monitoring

serviceMonitor:
  enabled: false # turn on after kube-prometheus-stack is installed (Part 11)
```

## 9.2 Helpers — `templates/_helpers.tpl`

```text
{{- define "app.labels" -}}
app.kubernetes.io/part-of: {{ .Chart.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
{{- end }}

{{- define "app.backendName" -}}{{ .Release.Name }}-backend{{- end }}
{{- define "app.frontendName" -}}{{ .Release.Name }}-frontend{{- end }}

{{- define "app.podSecurityContext" -}}
runAsNonRoot: true
seccompProfile:
  type: RuntimeDefault
{{- end }}

{{- define "app.containerSecurityContext" -}}
allowPrivilegeEscalation: false
capabilities:
  drop: ["ALL"]
{{- end }}
```

## 9.3 Backend: service account, SecretProviderClass, Deployment, Service, HPA

`templates/backend-serviceaccount.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Values.backend.serviceAccountName }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
  annotations:
    # Workload Identity: tokens for this service account are exchanged for this Azure identity.
    azure.workload.identity/client-id: {{ required "azure.backendClientId is required" .Values.azure.backendClientId | quote }}
```

`templates/backend-secretproviderclass.yaml`:

```yaml
# Tells the Secrets Store CSI driver which Key Vault secrets to fetch, and with which identity.
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: {{ include "app.backendName" . }}-kv
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: {{ .Values.azure.backendClientId | quote }} # Workload Identity
    keyvaultName: {{ required "azure.keyVaultName is required" .Values.azure.keyVaultName | quote }}
    tenantId: {{ required "azure.tenantId is required" .Values.azure.tenantId | quote }}
    objects: |
      array:
      {{- range $secret, $env := .Values.keyVaultSecrets }}
        - |
          objectName: {{ $secret }}
          objectType: secret
      {{- end }}
  # Also copy the values into a normal Kubernetes Secret so we can use them as env vars.
  # The Secret only exists while at least one pod mounts the CSI volume.
  secretObjects:
    - secretName: {{ include "app.backendName" . }}-secrets
      type: Opaque
      data:
      {{- range $secret, $env := .Values.keyVaultSecrets }}
        - objectName: {{ $secret }}
          key: {{ $env }}
      {{- end }}
```

`templates/backend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.backendName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
    app: backend
spec:
  {{- if not .Values.backend.hpa.enabled }}
  replicas: {{ .Values.backend.replicas }}
  {{- end }}
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1       # one extra pod during rollout
      maxUnavailable: 0 # never go below the desired count -> zero downtime
  template:
    metadata:
      labels:
        app: backend
        azure.workload.identity/use: "true" # webhook injects the federated token into the pod
    spec:
      serviceAccountName: {{ .Values.backend.serviceAccountName }}
      terminationGracePeriodSeconds: 45
      securityContext:
        {{- include "app.podSecurityContext" . | nindent 8 }}
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: backend
      containers:
        - name: backend
          image: "{{ .Values.image.registry }}/{{ .Values.backend.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.backend.port }}
          env:
            - name: APP_VERSION
              value: {{ .Values.image.tag | quote }}
            - name: JAVA_TOOL_OPTIONS
              value: {{ .Values.backend.javaToolOptions | quote }}
            - name: APPLICATIONINSIGHTS_ROLE_NAME # name shown in the App Insights application map
              value: backend
            {{- range $secret, $env := .Values.keyVaultSecrets }}
            - name: {{ $env }}
              valueFrom:
                secretKeyRef:
                  name: {{ include "app.backendName" $ }}-secrets
                  key: {{ $env }}
            {{- end }}
          # Slow JVM start: startup probe gives up to 150s before liveness kicks in.
          startupProbe:
            httpGet: { path: /actuator/health/liveness, port: http }
            periodSeconds: 5
            failureThreshold: 30
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: http }
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: http }
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              sleep:
                seconds: 10 # let Service endpoints / App Gateway stop sending traffic first
          resources:
            {{- toYaml .Values.backend.resources | nindent 12 }}
          securityContext:
            {{- include "app.containerSecurityContext" . | nindent 12 }}
            readOnlyRootFilesystem: true
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: kv-secrets
              mountPath: /mnt/secrets
              readOnly: true
      volumes:
        - name: tmp
          emptyDir: {}
        - name: kv-secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: {{ include "app.backendName" . }}-kv
```

`templates/backend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "app.backendName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
    app: backend
spec:
  type: ClusterIP # internal only: reachable from the frontend pods, never from the internet
  selector:
    app: backend
  ports:
    - name: http
      port: 8080
      targetPort: http
```

`templates/backend-hpa.yaml`:

```yaml
{{- if .Values.backend.hpa.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "app.backendName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "app.backendName" . }}
  minReplicas: {{ .Values.backend.hpa.minReplicas }}
  maxReplicas: {{ .Values.backend.hpa.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization # percent of the CPU *request*
          averageUtilization: {{ .Values.backend.hpa.targetCPUUtilizationPercentage }}
{{- end }}
```

## 9.4 Frontend: Deployment and Service

`templates/frontend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.frontendName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
    app: frontend
spec:
  replicas: {{ .Values.frontend.replicas }}
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: frontend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: frontend
    spec:
      terminationGracePeriodSeconds: 30
      securityContext:
        {{- include "app.podSecurityContext" . | nindent 8 }}
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: frontend
      containers:
        - name: frontend
          image: "{{ .Values.image.registry }}/{{ .Values.frontend.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.frontend.port }}
          env:
            - name: BACKEND_URL # used by nginx proxy_pass
              value: "http://{{ include "app.backendName" . }}:8080"
          # AGIC copies this readiness probe into the App Gateway health probe.
          readinessProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: /healthz, port: http }
            periodSeconds: 10
          lifecycle:
            preStop:
              sleep:
                seconds: 10
          resources:
            {{- toYaml .Values.frontend.resources | nindent 12 }}
          securityContext:
            {{- include "app.containerSecurityContext" . | nindent 12 }}
            readOnlyRootFilesystem: true
          # nginx needs two writable folders: its temp/pid files and the rendered config.
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: nginx-conf
              mountPath: /etc/nginx/conf.d
      volumes:
        - name: tmp
          emptyDir: {}
        - name: nginx-conf
          emptyDir: {}
```

`templates/frontend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "app.frontendName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
    app: frontend
spec:
  type: ClusterIP # AGIC sends traffic straight to pod IPs; the Service is used to find them
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: http
```

## 9.5 Ingress, PDB, NetworkPolicy, ServiceMonitor

`templates/ingress.yaml`:

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
  annotations:
    appgw.ingress.kubernetes.io/health-probe-path: /healthz
    appgw.ingress.kubernetes.io/connection-draining: "true"
    appgw.ingress.kubernetes.io/connection-draining-timeout: "30"
    {{- if .Values.ingress.tls.enabled }}
    appgw.ingress.kubernetes.io/appgw-ssl-certificate: {{ .Values.ingress.tls.appgwSslCertificate | quote }}
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - {{- with .Values.ingress.host }}
      host: {{ . }}
      {{- end }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "app.frontendName" . }}
                port:
                  name: http
{{- end }}
```

`templates/pdb.yaml`:

```yaml
# Keeps at least one pod of each tier running during node drains (upgrades, autoscaler scale-down).
{{- range $tier := list "frontend" "backend" }}
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ $.Release.Name }}-{{ $tier }}
  labels:
    {{- include "app.labels" $ | nindent 4 }}
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: {{ $tier }}
{{- end }}
```

`templates/networkpolicy.yaml`:

```yaml
{{- if .Values.networkPolicy.enabled }}
# 1) Deny everything in this namespace by default (zero trust).
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
# 2) Every pod may resolve DNS through CoreDNS.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes: ["Egress"]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - { protocol: UDP, port: 53 }
        - { protocol: TCP, port: 53 }
---
# 3) Frontend: only App Gateway may call it; it may only call the backend.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes: ["Ingress", "Egress"]
  ingress:
    - from:
        - ipBlock:
            cidr: {{ .Values.networkPolicy.appGatewaySubnetCidr }}
      ports:
        - { protocol: TCP, port: 8080 }
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - { protocol: TCP, port: 8080 }
---
# 4) Backend: called by frontend and Prometheus; may call PostgreSQL and HTTPS (App Insights).
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes: ["Ingress", "Egress"]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: {{ .Values.networkPolicy.monitoringNamespace }}
      ports:
        - { protocol: TCP, port: 8080 }
  egress:
    - to:
        - ipBlock:
            cidr: {{ .Values.networkPolicy.privateEndpointSubnetCidr }}
      ports:
        - { protocol: TCP, port: 5432 }
    - ports:
        - { protocol: TCP, port: 443 }
{{- end }}
```

`templates/servicemonitor.yaml`:

```yaml
{{- if .Values.serviceMonitor.enabled }}
# Tells Prometheus (kube-prometheus-stack) to scrape Spring Boot's /actuator/prometheus.
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "app.backendName" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      app: backend
  endpoints:
    - port: http
      path: /actuator/prometheus
      interval: 30s
{{- end }}
```

## 9.6 Kubernetes concepts in this chart (know each one)

| Concept | Where | Explain it like this |
|---|---|---|
| **startupProbe** | backend | "JVM needs up to ~60 s to start. The startup probe gives it up to 150 s; liveness and readiness don't run until it passes, so a slow start is never killed." |
| **livenessProbe** | both | "If it fails 3 times, kubelet **restarts** the container (deadlock, hung JVM). It must not check the database, or a DB outage would restart every pod." |
| **readinessProbe** | both | "If it fails, the pod is removed from Service endpoints and from the App Gateway pool, but **not restarted**. Our readiness includes the DB." |
| **App Gateway health probe** | Ingress annotation | "AGIC builds the App Gateway probe from the readiness probe / `health-probe-path`. If it fails: **502 Bad Gateway**." |
| **RollingUpdate `maxSurge: 1, maxUnavailable: 0`** | both | "A new pod must be Ready before an old one is removed: zero downtime." |
| **preStop sleep + graceful shutdown** | both | "On delete, the endpoint removal and SIGTERM happen in parallel. Sleeping 10 s first lets App Gateway and kube-proxy stop sending traffic; then Spring finishes in-flight requests." |
| **requests / limits** | both | "Requests are for scheduling (and HPA %), limits are the cap. Memory over the limit = **OOMKilled**. No CPU limit, so no CPU throttling." |
| **HPA** | backend | "Scales pods 2–5 on CPU above 70% of the *request*. Needs metrics-server (built into AKS)." |
| **PDB `minAvailable: 1`** | both | "A node drain (upgrade, autoscaler) can never evict the last pod of a tier." |
| **topologySpreadConstraints** | both | "Spread replicas across nodes, so one node failure doesn't take both." |
| **securityContext** | both | "Non-root, no privilege escalation, all Linux capabilities dropped, seccomp RuntimeDefault, read-only root filesystem (writable `emptyDir` only where needed)." |
| **ServiceAccount + label `azure.workload.identity/use`** | backend | "The webhook injects `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` and the projected token file into the pod." |
| **SecretProviderClass + `secretObjects`** | backend | "The CSI driver mounts Key Vault secrets as files **and** syncs a Kubernetes Secret, which we use as env vars. The synced Secret exists only while a pod mounts the volume." |
| **NetworkPolicy** | namespace | "Default deny. Then: App Gateway subnet → frontend:8080, frontend → backend:8080, monitoring → backend:8080, backend → PE subnet:5432, everyone → CoreDNS:53." |
| **ClusterIP Services** | both | "Stable virtual IP + DNS name in front of changing pod IPs. Not reachable from outside the cluster." |

## 9.7 Lint and test the chart locally

```bash
cd ~/three-tier-aks
SET="azure.tenantId=x,azure.keyVaultName=x,azure.backendClientId=x"
helm lint helm/three-tier-app --set $SET
helm template ttapp helm/three-tier-app -n app --set $SET > /tmp/rendered.yaml
less /tmp/rendered.yaml       # read what Helm really sends to Kubernetes

# Schema check against the Kubernetes API (CRDs like SecretProviderClass are skipped)
docker run --rm -v /tmp:/t ghcr.io/yannh/kubeconform:latest -strict -summary -ignore-missing-schemas /t/rendered.yaml

# Security misconfiguration check
docker run --rm -v "$PWD":/src -w /src aquasec/trivy:0.76.1 config --severity HIGH,CRITICAL \
  --ignorefile /src/.trivyignore --helm-set $SET /src/helm/three-tier-app
```

```bash
git add helm && git commit -m "Helm chart" && git push
```

---

[← Part 8 — Terraform stack 2: platform (AKS, App Gateway, PostgreSQL)](08-terraform-platform.md) | [Index](../README.md) | [Part 10 — First deployment by hand →](10-first-deployment.md)
