# 03 — Jenkins Pipelines and SonarQube Quality Gates

Static analysis runs on all three codebases — **backend APIs**, **admin UI** and **mobile app** — through Jenkins declarative pipelines and a self-hosted SonarQube server.

## First version: one pipeline, everything inline

```groovy
pipeline {
  agent any
  environment {
    SCANNER_HOME = tool 'sonarqube'
  }
  stages {
    stage('Checkout Code') {
      steps {
        checkout scmGit(branches: [[name: 'main']],
                        userRemoteConfigs: [[credentialsId: '<git-credentials-id>', url: '<repo-url>']])
      }
    }
    stage('Install Dependencies') {
      steps {
        nvm(version: 'lts/*') { sh 'npm install' }      // Jenkins nvm-wrapper plugin
      }
    }
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonarqube') {
          sh """
            ${SCANNER_HOME}/bin/sonar-scanner \
              -Dsonar.projectKey=<project-key> \
              -Dsonar.projectName=<project-name> \
              -Dsonar.sources=.
          """
        }
      }
    }
    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          script {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
              error "Pipeline failed due to SonarQube quality gate failure: ${qg.status}"
            }
          }
        }
      }
    }
  }
  post { always { cleanWs() } }
}
```

- **`withSonarQubeEnv`** injects the server URL and token configured in Jenkins.
- **`waitForQualityGate()`** waits for the SonarQube webhook and **fails the build** if the gate is not `OK` — the quality gate is enforced, not advisory.
- **`timeout(5 min)`** so a missing webhook can't hang the build.
- **`cleanWs()`** in `post { always }` keeps agents clean.

> The first version had the SonarQube token and server URL hard-coded in the Jenkinsfile. That was removed in the refactor below (see [06](06-security-review-and-lessons-learned.md)).

## Refactored version: thin Jenkinsfile + scripts in the repo

Same pipeline shape for backend, admin UI and mobile app — only the project key changes.

```groovy
pipeline {
  agent { label 'master' }
  environment {
    SONARQUBE_SERVER   = 'LocalSonar'
    SONAR_PROJECT_KEY  = 'NGL-Backend-api'          // NGL-Admin-ui, NGL-Mobile-app
    SONAR_PROJECT_NAME = 'NGL-Backend-api'
    SCANNER_HOME       = tool 'SonarLocal'
    JAVA_HOME          = tool name: 'jdk17'
    PATH               = "${JAVA_HOME}/bin:${SCANNER_HOME}/bin:${env.PATH}"
  }
  stages {
    stage('Installing Dependencies') {
      steps { sh 'bash script/build.sh' }
    }
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv(SONARQUBE_SERVER) {
          withCredentials([string(credentialsId: 'admin-sonarqube-token', variable: 'SONARQUBE_AUTH_TOKEN')]) {
            sh 'bash script/sonar.sh -h "$SCANNER_HOME" -k "$SONAR_PROJECT_KEY" -n "$SONAR_PROJECT_NAME" -u "$SONAR_HOST_URL" -t "$SONARQUBE_AUTH_TOKEN"'
          }
        }
      }
    }
  }
  post { always { cleanWs() } }
}
```

### `script/build.sh`

```bash
#!/bin/bash
set -euo pipefail
. /etc/profile.d/nvm.sh
nvm install          # uses the version in .nvmrc
npm install          # npm ci is better for reproducible CI installs
```

### `script/sonar.sh`

```bash
#!/bin/bash
set -euo pipefail

while getopts "h:k:n:u:t:" opt; do
  case "$opt" in
    h) SCANNER_HOME=$OPTARG ;;
    k) SONAR_PROJECT_KEY=$OPTARG ;;
    n) SONAR_PROJECT_NAME=$OPTARG ;;
    u) SONAR_HOST_URL=$OPTARG ;;
    t) SONAR_TOKEN=$OPTARG ;;
    *) echo "usage: $0 -h scanner_home -k key -n name -u url -t token"; exit 1 ;;
  esac
done

export SONAR_TOKEN     # scanner reads the token from the environment
"${SCANNER_HOME}/bin/sonar-scanner" \
  -Dsonar.projectKey="${SONAR_PROJECT_KEY}" \
  -Dsonar.projectName="${SONAR_PROJECT_NAME}" \
  -Dsonar.sources=. \
  -Dsonar.host.url="${SONAR_HOST_URL}"
```

### What the refactor improved

| Before | After |
|---|---|
| Token and server URL hard-coded in the Jenkinsfile | Token from Jenkins **credentials** (`withCredentials`), URL from `withSonarQubeEnv` |
| Logic duplicated in each Jenkinsfile | Shared `build.sh` / `sonar.sh` scripts versioned with each repo |
| Node version from the plugin (`lts/*`) | `nvm install` reads `.nvmrc` — same version locally and in CI |
| JDK implicit | `jdk17` tool pinned (current scanners need Java 17) |

### Bugs found and fixed while reviewing the scripts

- **`getopts "k:n:u:t:"` was missing `h:`**, so `-h` was rejected and `SCANNER_HOME` only worked because Jenkins already exported it. Adding `h:` fixes it.
- **No `usage` function** existed for the `*)` branch.
- **Token on the command line** (`-Dsonar.login=…`) shows up in process lists and logs; the scanner reads `SONAR_TOKEN` from the environment instead (`sonar.login` is deprecated).
- **No quality gate** in the refactored pipeline — add the `waitForQualityGate()` stage back, or pass `-Dsonar.qualitygate.wait=true`, so failing code still breaks the build.

## Production approval gate

Requirement from the client: production deploys must wait for named approvers.

```groovy
stage('Approve production') {
  when { branch 'main' }
  steps {
    timeout(time: 24, unit: 'HOURS') {
      input message: 'Deploy to production?', submitter: '<approver-group>'
    }
  }
}
```

(In Azure DevOps the same control is an **environment with approvals and checks**.)

## Interview Q&A

**How do you enforce code quality in CI?**
SonarQube analysis on every build plus `waitForQualityGate()`, which fails the pipeline if the gate (coverage, new bugs, vulnerabilities, duplication) isn't met.

**How does Jenkins know the analysis finished?**
SonarQube calls a Jenkins webhook when the background task completes; `waitForQualityGate` waits for it, with a timeout.

**How do you keep secrets out of Jenkinsfiles?**
Jenkins credentials with `withCredentials` (masked in logs), server config via `withSonarQubeEnv`, and tokens passed through environment variables, not arguments.

**Why move logic into scripts?**
The Jenkinsfile stays a thin orchestrator; scripts are testable locally, versioned with the code, and reused across three repos.
