---
name: ansible-role-testing
description: Use when editing proALPHA Ansible repos in WSL under /repos, testing Ansible roles/playbooks, pushing deploy.proalpha.ansiblegalaxy/deploy.proalpha.playbooks changes, or explicitly deploying/testing with AWS RTU/RoleTestingUtility.
---

# Ansible Role Testing

## Purpose

Test Ansible roles/playbooks in proALPHA WSL environment. Manages edit workflow across `/repos/` (Ubuntu-Dev), Windows D: drive (AnsibleWSL), and AWS RTU (remote testing).

## When to Use

- Testing Ansible role/playbook changes via RTU
- Editing Ansible repos in WSL under /repos
- Deploying or testing with AWS Role Testing Utility

---

## Operating Model

All three repos live under `/repos`.

Agents edit only `/repos/deploy.proalpha.ansiblegalaxy` and `/repos/deploy.proalpha.playbooks`.

RTU lives at `/repos/role-testing-utility.ansible.dev`. RTU runs WSL-native (Python venv at `/opt/rtu-venv`); it mounts no containers and requires no Docker.

Agent-edited files are the same files RTU tests.

| Repo | Purpose |
|---|---|
| `/repos/deploy.proalpha.ansiblegalaxy` | Collection roles and tests |
| `/repos/deploy.proalpha.playbooks` | ERP playbooks and values |
| `/repos/role-testing-utility.ansible.dev` | AWS RTU runner/profile only |

Do normal code work in `/repos/...`, commit there, push there. Do not edit Windows D: clones.

## WSL Setup

**What `06-setup-ansible-rtu.sh` does (automated):**

Run once after SSH works:

```bash
cd ~/wsl-setup
./06-setup-ansible-rtu.sh
```

Automates:
- clones or updates all three repos under `/repos`
- calls `ansible/rtu/scripts/setup-venv.sh` to create Python venv at `/opt/rtu-venv`
- installs `/usr/local/bin/rtu`
- symlinks `/usr/local/bin/rtu-wsl → rtu` for backward compatibility

**What you must do manually — one time only:**

Initialize the RTU profile. Creates `rtu_definitions/deploy_proalpha/` including `aws_creds.conf` and patches generated paths to `/repos/...`.

```bash
rtu init deploy_proalpha
```

**What you must do manually — before every AWS test session:**

AWS SSO credentials expire after 12 hours. Refresh them:

1. Open `https://d-9967282068.awsapps.com/start` in a browser.
2. Select `SharedServices-Ansible-dev`.
3. Click `Access keys`.
4. Copy the block from `Option 2` (starts with `[597350916081_prj-engineer]`).
5. In WSL:
   ```bash
   nano /repos/role-testing-utility.ansible.dev/rtu_definitions/deploy_proalpha/aws_creds.conf
   ```
6. Replace entire file content with the copied block. Save.

The file must look like:

```ini
[597350916081_prj-engineer]
aws_access_key_id=ASIA...
aws_secret_access_key=...
aws_session_token=...
```

RTU is the Python tool at `/repos/role-testing-utility.ansible.dev/tools/rtu/rtu.py`. The `rtu` wrapper activates `/opt/rtu-venv` and delegates to it. No Docker or devcontainer required.

Do not install RTU separately.

## Agent Workflow

1. Edit only `/repos/deploy.proalpha.ansiblegalaxy` or `/repos/deploy.proalpha.playbooks`.
2. Commit and push the changed repo branch.
3. Do not deploy to AWS unless user explicitly asks.
4. Use `rtu`, not VS Code, for headless agent-triggered AWS tests.

## AWS RTU Test

### AWS Credentials

If no current credentials are available, tell the user:

1. Open `https://d-9967282068.awsapps.com/start`.
2. Select `SharedServices-Ansible-dev`.
3. Click `Access keys`.
4. Copy the credentials block from `Option 2`.
5. In WSL, paste the copied block into `/repos/role-testing-utility.ansible.dev/rtu_definitions/deploy_proalpha/aws_creds.conf`.

Agents must not ask for credentials in chat, store credentials elsewhere, or try to generate them. Credentials expire after 12 hours.

Check setup:

```bash
test -d /repos/deploy.proalpha.ansiblegalaxy/.git
test -d /repos/deploy.proalpha.playbooks/.git
test -d /repos/role-testing-utility.ansible.dev/.git
rtu list_instances
```

Complete install test:

```bash
rtu provision
rtu test
rtu deprovision
```

If inspection needed:

```bash
rtu list_instances
rtu rdp rtu_instance
rtu shell          # drops into bash with /opt/rtu-venv activated
```

Default cleanup: always `rtu deprovision` after test. Leave AWS VM running only when user explicitly asks.

## Test Definition

Complete install path lives in `/repos/role-testing-utility.ansible.dev/rtu_definitions/deploy_proalpha/test_definitions.yml` and must import playbooks through the `/repos` path:

```yaml
- name: Run proALPHA Installation
  ansible.builtin.import_playbook: /repos/deploy.proalpha.playbooks/erp/install.yml
```

Local collections in `requirements_local.yml` must use `/repos/deploy.proalpha.ansiblegalaxy/...` paths. `rtu init deploy_proalpha` patches generated `/mnt/home/ansible/...` defaults to `/repos/...`.

## Red Flags

| Thought | Reality |
|---|---|
| "Edit `/mnt/d/proalpha/ansible/...`" | Wrong model. Edit WSL repos under `/repos/...`. |
| "Clone into `~/ansible` for RTU" | Wrong model. All working repos live under `/repos`. |
| "Use VS Code Dev Containers to run RTU" | Use `rtu`; WSL-native venv, no Docker needed. |
| "Run `rtu init deploy_proalpha` every time" | One-time profile init. Check existing `rtu_definitions/conf.yml` first. |
| "RTU should use `/mnt/home/ansible/...` paths" | Wrong for WSL agent workflow. `rtu init` patches to `/repos/...`. |
| "Agent can fetch AWS creds" | No. User must paste 12h SSO creds. |
| "Deploy while testing code change by default" | No. Commit/push only unless user explicitly requests AWS deploy/test. |
| "Leave test VM running" | Deprovision unless user explicitly requests keeping it. |
