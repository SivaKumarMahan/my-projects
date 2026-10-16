# 06 — Security, DevSecOps and Compliance

## Read-only CVE remediation scanner

- Scans hosts **over SSH, read-only** — it never changes a host.
- Keeps **JSON scan history** and reports **what changed since the last scan** (new, fixed, still open).
- Has tests.
- **Angle:** safe-by-design security tooling; diff-based reporting so teams see progress, not a wall of findings.

## SAST finding triage: SSRF hardening story

A SAST tool flagged a GitHub sync script for possible **SSRF**.

| Step | Detail |
|---|---|
| Finding | Script makes HTTP requests to a configurable base URL |
| Triage | Mostly a false positive — the URL normally comes from trusted config |
| Real risks found | The base URL was configurable without validation, and HTTP redirects were followed implicitly |
| Fix | **Host allow-list** for the base URL; **redirects disabled / controlled** |
| Outcome | Finding closed with a written explanation of what was a false positive and what was fixed |

**Use it for:** "Tell me about a vulnerability you fixed." It shows you don't dismiss a finding as a false positive without checking for the real risk behind it.

```python
# Illustrative hardening
ALLOWED_HOSTS = {"api.github.com"}
host = urlparse(base_url).hostname
if host not in ALLOWED_HOSTS:
    raise ValueError(f"Host not allowed: {host}")
resp = session.get(url, allow_redirects=False, timeout=30)
```

## Access audits stored as code

- **Organization-wide IAM and SSO permissions audit:** risk summary, IAM-user access-key inventory, effective-access CSVs per principal.
- **GitHub repository ruleset** exported as JSON, plus the list of repos affected by a compliance-platform finding.
- **Angle:** least-privilege reviews, access-key hygiene (find and remove long-lived IAM user keys), branch protection as code, compliance evidence kept in git. 🟡

## AI review with security roles

The PR review prompts include `sensitive-data.md` and `privileged-automation.md` checklists (see [01](01-reusable-github-actions-and-ai-pr-review.md)), so security review happens on every PR.

## Container and host scanning

- **Trivy** on hosts through Ansible, with a managed ignore list for accepted risks ([05](05-ansible-patching-and-config-management.md)).
- Hard-coded environment variable gate on PR diffs ([01](01-reusable-github-actions-and-ai-pr-review.md)).

## Interview Q&A

**How do you handle a SAST false positive?**
Reproduce or reason about exploitability, look for the underlying weakness anyway, fix what's real, document why the rest is a false positive, and suppress it narrowly (not the whole rule).

**How do you find over-privileged access?**
Export effective permissions (IAM, SSO permission sets, GitHub), compare against role needs, list long-lived access keys and their last use, and track remediation in the audit summary.

**What's the risk with following redirects in server-side requests?**
An allow-listed host can redirect to an internal address (e.g. a metadata endpoint), bypassing the allow-list — classic SSRF.
