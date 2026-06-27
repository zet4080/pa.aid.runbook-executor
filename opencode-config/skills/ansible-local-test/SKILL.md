---
name: ansible-local-test
description: Use when asked to test ansible role changes, run ansible locally, test a role, test changes, run ansible test, or validate ansible playbook against local Windows machine
---

# Ansible Local Test

## Purpose

Test Ansible role/playbook changes locally via AnsibleWSL distro. Bridges Ubuntu-Dev (`/repos/`) edits to Windows D: drive copy for AnsibleWSL execution.

## When to Use

- Testing Ansible role changes locally
- Running Ansible locally or validating playbooks
- Testing against local Windows machine via AnsibleWSL

---

## Overview

Test Ansible role/playbook changes locally via AnsibleWSL distro. Changes made in Ubuntu-Dev (`/repos/`) must be pushed to git and pulled into the Windows D: drive copy before AnsibleWSL can see them.

## Quick Reference

| Item | Path |
|---|---|
| Edit roles here | `/repos/deploy.proalpha.ansiblegalaxy/` |
| Edit playbooks here | `/repos/deploy.proalpha.playbooks/` |
| AnsibleWSL roles (D: drive) | `/mnt/d/proalpha/ansible/deploy.proalpha.ansiblegalaxy/` |
| AnsibleWSL playbooks (D: drive) | `/mnt/d/proalpha/ansible/deploy.proalpha.playbooks/` |
| Inventory | `/mnt/d/proalpha/ansible/inventory.yml` |
| Dev values | `/mnt/d/proalpha/ansible/deploy.proalpha.playbooks/erp/dev.yml` |
| Ansible binary | `/home/ansible/.venv/bin/ansible-playbook` |
| Run in AnsibleWSL | `wsl.exe -d AnsibleWSL -u ansible -- bash -c "..."` |

## Workflow

### 1 — Push changes (Ubuntu-Dev)
```bash
cd /repos/deploy.proalpha.ansiblegalaxy   # or deploy.proalpha.playbooks
git add .
git commit -m "feat: describe change"
git push
```

### 2 — Pull in AnsibleWSL D: copy
```bash
wsl.exe -d AnsibleWSL -u ansible -- bash -c "cd /mnt/d/proalpha/ansible/deploy.proalpha.ansiblegalaxy && git pull"
# if playbooks changed too:
wsl.exe -d AnsibleWSL -u ansible -- bash -c "cd /mnt/d/proalpha/ansible/deploy.proalpha.playbooks && git pull"
```

### 3 — Find the component tag
Tags follow pattern `component_<name>` e.g. `component_pasoe`, `component_broker`, `component_db`.

```bash
grep -r "tags:" /mnt/d/proalpha/ansible/deploy.proalpha.playbooks/erp/install.yml | head -20
```

### 4 — Run targeted test
```bash
wsl.exe -d AnsibleWSL -u ansible -- bash -c "
  source /home/ansible/.venv/bin/activate && \
  ansible-playbook \
    -i /mnt/d/proalpha/ansible/inventory.yml \
    /mnt/d/proalpha/ansible/deploy.proalpha.playbooks/erp/install.yml \
    -e @/mnt/d/proalpha/ansible/deploy.proalpha.playbooks/erp/dev.yml \
    -e runtime_environment=dev \
    --tags <component_tag> \
    -v
"
```

Full install (no tag):
```bash
wsl.exe -d AnsibleWSL -u ansible -- bash -c "
  source /home/ansible/.venv/bin/activate && \
  ansible-playbook \
    -i /mnt/d/proalpha/ansible/inventory.yml \
    /mnt/d/proalpha/ansible/deploy.proalpha.playbooks/erp/install.yml \
    -e @/mnt/d/proalpha/ansible/deploy.proalpha.playbooks/erp/dev.yml \
    -e runtime_environment=dev \
    -v
"
```

### 5 — Check idempotency (recommended)
Run step 4 again. All tasks must show `ok`, zero `changed`. Any `changed` = idempotency bug.

### 6 — Verify SSH if connection fails
```bash
wsl.exe -d AnsibleWSL -u ansible -- bash -c "
  source /home/ansible/.venv/bin/activate && \
  ansible -i /mnt/d/proalpha/ansible/inventory.yml all -m win_ping
"
```
Expected: `win_server | SUCCESS => { "ping": "pong" }`

## Red Flags — STOP

| Thought | Reality |
|---|---|
| "I'll edit `/mnt/d/` directly" | Edit in `/repos/`, push, pull. Never edit D: directly. |
| "Changes are in `/repos/`, just run it" | `/repos/` is invisible from AnsibleWSL. Must push+pull first. |
| "I'll skip `dev.yml`" | Missing `-e @dev.yml` breaks port/path values for local setup. |
| "The tag is just `pasoe`" | Tags are `component_pasoe`, not bare role names. |
| "No need to activate venv" | `ansible-playbook` not on PATH without venv. Use full path or activate. |
