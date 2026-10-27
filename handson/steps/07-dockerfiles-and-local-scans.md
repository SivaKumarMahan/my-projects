# Part 7 — Multistage Dockerfiles and local scans

[← Part 6 — The application code](06-application-code.md) | [Index](../README.md) | [Part 8 — Terraform stack 2: platform (AKS, App Gateway, PostgreSQL) →](08-terraform-platform.md)

---

## 7.1 Backend — `backend/Dockerfile`

```dockerfile
# ---------- Stage 1: build the JAR ----------
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /src

# Copy only pom.xml first: dependencies are cached until pom.xml changes.
COPY pom.xml .
RUN mvn -B -q dependency:go-offline

COPY src ./src
RUN mvn -B -q package -DskipTests

# Azure Monitor Application Insights Java agent (auto-collects requests, SQL calls, exceptions).
# --chmod matters: a remote ADD is saved as mode 600 (root only), so the non-root user could not read it.
ADD --chmod=644 https://github.com/microsoft/ApplicationInsights-Java/releases/download/3.7.10/applicationinsights-agent-3.7.10.jar /src/applicationinsights-agent.jar

# ---------- Stage 2: small runtime image ----------
# Distroless: no shell, no package manager, runs as non-root (uid 65532).
FROM gcr.io/distroless/java21-debian13:nonroot
WORKDIR /app
COPY --from=build /src/target/backend-0.0.1-SNAPSHOT.jar /app/app.jar
COPY --from=build /src/applicationinsights-agent.jar /app/applicationinsights-agent.jar

EXPOSE 8080
USER nonroot
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

`backend/.dockerignore`:

```text
target/
.mvn/
mvnw
mvnw.cmd
.git
*.md
```

## 7.2 Frontend — `frontend/Dockerfile`

```dockerfile
# ---------- Stage 1: build static files ----------
FROM node:24-alpine AS build
WORKDIR /app

# Install dependencies first so this layer is cached until package*.json changes.
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund

COPY . .
RUN npm run build

# ---------- Stage 2: serve with nginx as non-root ----------
# nginx-unprivileged runs as uid 101 and listens on 8080 (not 80).
FROM nginxinc/nginx-unprivileged:1.31-alpine

# Patch OS packages that have published fixes (keeps Trivy green between base-image releases).
USER root
RUN apk upgrade --no-cache
USER 101

COPY nginx/default.conf.template /etc/nginx/templates/default.conf.template
COPY --from=build /app/dist /usr/share/nginx/html

ENV BACKEND_URL=http://backend:8080
EXPOSE 8080
```

`frontend/.dockerignore`:

```text
node_modules
dist
.git
*.md
```

## 7.3 What to say about these Dockerfiles

| Technique | Where | Why |
|---|---|---|
| **Multistage build** | both | Maven/JDK and Node exist only in the build stage. The final image has only the JRE + JAR, or nginx + static files. Measured: backend final image ≈ 300 MB (the Maven + JDK build image alone is ≈ 540 MB, before dependencies); frontend ≈ 65 MB (the Node build image alone is ≈ 170 MB, plus `node_modules`) |
| **Layer caching** | `COPY pom.xml` / `package*.json` first | dependencies download again only when those files change, so code changes rebuild in seconds. Your answer to "how do you reduce Docker build time?" |
| **Non-root** | `distroless:nonroot`, `nginx-unprivileged` | a container escape lands as an unprivileged user; required by `runAsNonRoot: true` in Kubernetes |
| **Distroless** | backend runtime | no shell, no package manager, so a much smaller attack surface and fewer CVEs |
| **`.dockerignore`** | both | smaller build context, no `target/` or `node_modules/` leaking in, faster builds |
| **`ENTRYPOINT` vs `CMD`** | backend | `ENTRYPOINT` is the fixed executable; `CMD` gives default arguments that `docker run image <args>` can override. JVM options come from `JAVA_TOOL_OPTIONS`, so the entrypoint never changes |
| **`ADD --chmod`** | backend | a real bug found while testing: a remote `ADD` is saved as mode 600 (root only), so the non-root JVM failed with "Error opening zip file". `--chmod=644` fixes it |
| **`apk upgrade`** | frontend | patches OS packages that have fixes not yet in the base image |

## 7.4 Build and run the images locally

```bash
cd ~/three-tier-aks
docker build -t ttapp-backend:local backend
docker build -t ttapp-frontend:local frontend
docker images | grep ttapp

# Run the backend like Kubernetes will: read-only root filesystem, writable /tmp only
docker network create ttnet
docker run -d --name pg --network ttnet -e POSTGRES_DB=appdb -e POSTGRES_PASSWORD=postgres postgres:16-alpine
docker run -d --name be --network ttnet --read-only --tmpfs /tmp -p 8080:8080 \
  -e DB_URL=jdbc:postgresql://pg:5432/appdb ttapp-backend:local
sleep 25 && curl -s localhost:8080/actuator/health/readiness     # {"status":"UP"}

docker exec be sh     # fails: "executable file not found" -> distroless has no shell (on purpose)
docker rm -f be pg && docker network rm ttnet
```

How to debug a distroless container is a good interview question. In Kubernetes you use `kubectl debug` to attach an **ephemeral debug container** that shares the process namespace (Part 10.5).

## 7.5 Scan locally with Trivy and Gitleaks

You don't need to install these tools; run them as containers:

```bash
cd ~/three-tier-aks
TRIVY=aquasec/trivy:0.76.1

# 1) Image scan: OS packages + Java/npm libraries inside the image
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock $TRIVY \
  image --severity HIGH,CRITICAL --ignore-unfixed ttapp-backend:local
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock $TRIVY \
  image --severity HIGH,CRITICAL --ignore-unfixed ttapp-frontend:local

# 2) Filesystem scan: pom.xml and package-lock.json dependencies + hard-coded secrets
docker run --rm -v "$PWD":/src -v "$HOME/.m2":/root/.m2 $TRIVY \
  fs --offline-scan --scanners vuln,secret --severity HIGH,CRITICAL --ignore-unfixed \
  --skip-dirs /src/frontend/node_modules --skip-dirs /src/backend/target /src

# 3) Misconfiguration scan: Terraform (Helm is added in Part 9). One target per run.
docker run --rm -v "$PWD":/src -w /src $TRIVY config --severity HIGH,CRITICAL \
  --ignorefile /src/.trivyignore /src/infra

# 4) Secrets in the whole Git history
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD":/repo ghcr.io/gitleaks/gitleaks:v8.30.1 git /repo --redact
```

**Real findings from testing this guide (use them as interview stories):**

1. **Base images had fixable HIGH CVEs.** `distroless/java21-debian12` and `nginx-unprivileged:1.29-alpine` failed the gate (expat, brotli, curl…). The fix was moving to `java21-debian13` and `1.31-alpine`, plus `apk upgrade`. Then the scan was clean. *"How do you fix Trivy findings?"*: update the base image first, then the dependency; use `--ignore-unfixed` so you only block on things you *can* fix; record a justified exception in `.trivyignore` only when the risk is accepted.
2. **Maven Central returned HTTP 429** when Trivy looked up the parent POM. The fix is to run Trivy *after* the Maven build, mount `~/.m2`, and use `--offline-scan`.
3. **Trivy config flagged AZU-0047** ("NSG allows ingress from any IP") on the App Gateway subnet. That rule is the whole point of a public web entry point, and WAF protects it, so it is suppressed in `.trivyignore` **with a comment explaining why**. Never suppress silently.
4. **Trivy config flagged KSV-0014** (frontend without a read-only root filesystem). It was fixed in the chart (Part 9), not ignored.

Test that Gitleaks really works by committing a fake token on a throwaway branch:

```bash
git checkout -b leak-test
echo "token=ghp_$(head -c 200 /dev/urandom | tr -dc 'A-Za-z0-9' | head -c 36)" > leak.txt
git add leak.txt && git commit -m "test leak"
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD":/repo ghcr.io/gitleaks/gitleaks:v8.30.1 git /repo --redact
# "leaks found: 1", exit code 1
git checkout main && git branch -D leak-test     # never push that branch
```

```bash
git add . && git commit -m "Multistage Dockerfiles" && git push
```

---

[← Part 6 — The application code](06-application-code.md) | [Index](../README.md) | [Part 8 — Terraform stack 2: platform (AKS, App Gateway, PostgreSQL) →](08-terraform-platform.md)
