# 07 — Identity and Access Automation, and Agentic Onboarding

## Identity scripts (utilities repository)

| Area | Scripts |
|---|---|
| **Entra ID** | Guest invites, inactive-user reports, group membership export, remove users from groups, hide users from the address list |
| **FusionAuth** (customer identity) | ~15 scripts for users, tenants, entities, grants and logins, built on a small client library |
| **GitHub** | Sync org members into a team, grant the team read access; with tests |
| **Databricks** | Access-matrix generator, user listing, Genie space audit, job and warehouse cost scripts |
| **AWS** | IAM Identity Center user listing, S3 utilities, SSM environment fetch, assume-role helpers |

**Resume link:** Entra ID, RBAC, managed identities and automated role assignments are the same ideas as the Azure work in project1, applied across more systems.

## Agentic onboarding and offboarding 🟡

A multi-agent system that onboards and offboards people across **Entra ID, FusionAuth, Databricks, Atlassian, AWS, GitHub and Tableau**.

```
Trigger agent (new joiner / leaver request)
        │
        ▼
Orchestrator ──► per-system agents (one template, one agent per system)
        │            Entra · FusionAuth · Databricks · Atlassian · AWS · GitHub · Tableau
        ▼
Audit agent ──► checks final state matches the request, writes the audit record
```

- Built on an **MCP (Model Context Protocol)** architecture — each system is exposed to agents as tools.
- **Per-system agent template** keeps behaviour consistent.

### Documented safety rules

| Rule | Why |
|---|---|
| **Double approval for destructive actions** (deletes, removals) | A wrong offboarding is costly and hard to undo |
| **Agents never mint credentials** | Passwords, keys and tokens are issued by humans or the identity system's own flows |
| **Human vs agent task split** | Clear list of what agents may do alone vs what needs a person |
| **Scratch directory kept out of git** | Agent working files never leak into commits |

**Angle:** agentic automation with guardrails — a strong differentiator in current interviews. Be ready to discuss prompt injection, least-privilege tool access, auditability and how a human stays in the loop.

## Interview Q&A

**How do you make offboarding reliable?**
One request drives every system, each system reports its result, an audit step verifies nothing is left (sessions, keys, group memberships, licences), and failures are retried or escalated — never silently skipped.

**What guardrails do you put on an AI agent with admin access?**
Least-privilege tools per system, read-only by default, human approval for destructive actions, no credential creation, full audit log of every tool call, and inputs treated as untrusted data.
