# Part 11 — SonarQube Cloud

[← Part 10 — First deployment by hand](10-first-deployment.md) | [Index](../README.md) | [Part 12 — GitHub Actions CI/CD →](12-github-actions-cicd.md)

---

We use **SonarQube Cloud** (formerly SonarCloud). It is free for public repositories and needs no server. The checks and quality gates are the same as a self-hosted SonarQube server, so everything you learn applies to both. (The self-hosted option is in Appendix A.)

> SonarSource changes its UI from time to time. If a button has a slightly different name, look for the same idea.

1. Go to <https://sonarcloud.io> → **Log in with GitHub**.
2. **+ → Analyze new project → Import an organization** → choose your GitHub account → choose the **Free** plan. Note the **organization key** (for example `yourname`).
3. Select the `three-tier-aks` repository. It is a monorepo (two apps), so choose **monorepo / set up multiple projects** and create two projects:
   - key `<org>_backend`, name `three-tier backend`
   - key `<org>_frontend`, name `three-tier frontend`
4. For **each** project: **Administration → Analysis Method → turn OFF Automatic Analysis**. We analyse from CI, and the two methods cannot run together.
5. For each project, **Administration → New Code**: set "Previous version" (or "Number of days: 30").
6. **My Account → Security → Generate token**. Then in GitHub:
   - secret `SONAR_TOKEN` = the token
   - variable `SONAR_ORG` = your organization key

**What SonarQube checks (interview answer):**

| Category | Meaning | Example |
|---|---|---|
| Bugs (Reliability) | code that is likely wrong | null dereference, resource leak |
| Vulnerabilities (Security) | code that is exploitable | SQL built by string concatenation |
| Security Hotspots | code a human must review | hard-coded IP, weak crypto, CORS `*` |
| Code Smells (Maintainability) | hard-to-maintain code | long methods, duplicated blocks, unused imports |
| Coverage | % of lines run by tests | from `jacoco.xml` |
| Duplications | copy-pasted code | |

The **Quality Gate** ("Sonar way") is a pass/fail rule on **new code**: for example coverage ≥ 80%, no new bugs or vulnerabilities, and every hotspot reviewed. `-Dsonar.qualitygate.wait=true` makes the CI step **fail** when the gate fails, which blocks the merge.

> Your backend has only 3 tests. If the gate fails on coverage, that is a real result: add a test, or adjust the gate in SonarQube Cloud. Talk about this trade-off in interviews.

---

[← Part 10 — First deployment by hand](10-first-deployment.md) | [Index](../README.md) | [Part 12 — GitHub Actions CI/CD →](12-github-actions-cicd.md)
