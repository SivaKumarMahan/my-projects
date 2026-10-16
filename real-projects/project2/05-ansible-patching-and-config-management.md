# 05 — Ansible Patching and Configuration Management

A maintenance-ops repository manages Linux and Windows hosts with Ansible, one folder per concern.

## Layout

| Folder | Purpose |
|---|---|
| `computeUpdate/` | Monthly OS patching: isolated inventory, runbook, saved `.check` and `.run` output files, WinRM setup for Windows hosts, calendar reminder (`.ics`) |
| `trivy/` | Installs Trivy, generates vulnerability reports, manages an ignore list |
| `etckeeper/` | Tracks `/etc` in Git on each host, so config changes are diffable and auditable |
| `reporting/` | Disk, network and machine-summary reports |
| `splunk/` | Splunk deployment server and universal forwarder clients |
| `status_page/` | Status page integration for maintenance windows |
| `ec2Maintenance/patchManager` | AWS Systems Manager Patch Manager (also defined in OpenTofu) |

## Secrets: 1Password, not vault files

- Credentials and SSH keys come from **1Password** — the 1Password SSH agent for keys, the CLI for secrets.
- **No Ansible Vault files and no key files** in the repo or on disk.
- Loader scripts set up the environment for bash, zsh and fish.

## Makefile UX

```make
make ping                                 # connectivity check
make check PLAY=<playbook> LIMIT=<hosts>  # ansible-playbook --check --diff
make apply PLAY=<playbook> LIMIT=<hosts>  # real run
make computeupdate                        # monthly patch flow
# FORKS=<n> controls parallelism
```

## Safe change flow

```
check (--check --diff) ──► review saved .check output ──► apply in batches (LIMIT / serial) ──► save .run output
```

- **Dry run first**, with diffs saved to files for review and audit.
- **Batches** (`LIMIT`, `serial`) so not every host restarts at once.
- Status page updated for the maintenance window.

## Illustrative patch play

```yaml
- hosts: linux
  become: true
  serial: "25%"
  tasks:
    - name: Apply updates
      ansible.builtin.package:
        name: "*"
        state: latest
    - name: Check if reboot is required
      ansible.builtin.stat: { path: /var/run/reboot-required }
      register: reboot
    - name: Reboot when needed
      ansible.builtin.reboot: { reboot_timeout: 900 }
      when: reboot.stat.exists
```

## Interview Q&A

**How do you patch Windows and Linux fleets without downtime?**
Group hosts by role, patch in rolling batches (`serial`), drain or fail over load-balanced members first, dry-run with `--check --diff`, reboot only when required, verify health before the next batch, and keep a status page and calendar for the window. For cloud instances, Patch Manager with maintenance windows and patch baselines.

**Why etckeeper?**
Every `/etc` change becomes a git commit on the host — you can see what changed, when, and roll back.

**Why 1Password instead of Ansible Vault?**
No shared vault password to distribute or rotate, per-user access with audit, and keys never land on disk.
