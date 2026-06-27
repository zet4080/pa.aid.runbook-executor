---
name: ansible-role-writing
description: Use when writing Ansible roles, playbooks, or custom modules for deploy.proalpha.ansiblegalaxy or deploy.proalpha.playbooks, especially argument_specs, tests/test.yml, FQCN, idempotency, or win_shell/win_command review
---

# Ansible Role Writing

## Purpose

Write Ansible roles/playbooks/modules with required structure, specs, tests, FQCN, idempotency. Enforces argument_specs, FQCN, win_shell/win_command compliance.

## When to Use

- Writing Ansible roles or playbooks for deploy.proalpha.* repos
- Writing custom modules for proALPHA Ansible deployments
- Reviewing role structure, argument_specs, or idempotency

---

## 1. Required Files (HIGH if missing)
|------|---------|
| `meta/main.yml` | Galaxy metadata — see § 2 |
| `meta/argument_specs.yml` | Parameter schema — see § 3 |
| `tasks/main.yml` | Role entry point |
| `tests/test.yml` | Test — `hosts: os_windows` |
| `README.md` | Dependencies, config options, operation/side-effects — **HIGH** per ansible.md § 1 |

## 2. meta/main.yml Required Fields

```yaml
galaxy_info:
  author: [Name]
  description: [Role description]
  company: Proalpha          # capital P — exact
  license: Proalpha
  min_ansible_version: "2.1"
  galaxy_tags: []
dependencies: []
```

> See ansible.md § 10 — missing `license`, `company`, `galaxy_tags`, or `dependencies` = MEDIUM.

## 3. argument_specs.yml Rules (HIGH violations)

- Top-level key: `{role_name}_settings`, type: dict, required: true
- Every param: `type`, `required`, `description`
- Order: lexicographic within each nesting level
- `required: false` needs justification — inline comment referencing ticket OR PR comment with rationale; no justification = HIGH

```yaml
argument_specs:
  main:
    short_description: Entry point for [role]
    options:
      pasoe_settings:
        type: dict
        required: true
        description: Settings dictionary for pasoe
        options:
          assemblies_sys:
            type: dict
            required: true
            description: settings dictionary for assemblies
```

## 4. tests/test.yml Rules

```yaml
- hosts: os_windows        # NOT 'all'
  gather_facts: false      # MUST be false
  remote_user: root
  vars_files:
    - "{{ playbook_dir }}/../../../../test_values.yml"
    - "{{ playbook_dir }}/../../../../openedge_test_values.yml"
```

- Use `ansible.builtin.assert` for verification
- Paths on D: drive — NOT C:
- Parameter blocks: `_test` suffix, e.g. `role_settings: "{{ role_settings_test }}"`

## 5. Idempotency Verification

Run role twice; second run must show zero changes.

Watch for: `register→when→changed` patterns, random values (GUIDs), `win_shell`/`win_command` usage.

> See ansible.md § 5 — erp.configure, erp.erp_components, erp.install, erp.system, erp.utils MUST be idempotent. erp.app is one-shot only.

## 6. Custom Modules (`{role_name}/plugins/modules/`)

Two files required: `{name}.ps1` + `{name}.py`

**PS1 minimal structure:**
```ps1
$spec = @{
  options = @{
    param_name = @{ type = "str"; required = $true }
  }
  supports_check_mode = $true
}
$module = [Ansible.Basic.AnsibleModule]::Create($args, $spec)
```

**Idempotency pattern — check state first:**
```ps1
$current = Get-Service -Name $module.Params.service_name | Select -ExpandProperty StartType
if ($current -ne $module.Params.startup_type) {
  Set-Service -Name $module.Params.service_name -StartupType $module.Params.startup_type
  $module.Result.changed = $true
}
```

- Error via `$module.FailJson(...)` — not raw exceptions

**Python wrapper** must declare:
- `DOCUMENTATION`: module name, short_description, options
- `EXAMPLES`: at least one working example
- `RETURN`: `changed` + any outputs described

> Missing any of these = MEDIUM per ansible.md § 7.

## 7. Quick Patterns

**Tags** (MEDIUM if missing on ERP component plays):
```yaml
tags: component_[name]   # e.g. component_db_server, component_pasoe
```

**FQCN — always** (MEDIUM if missing):
```yaml
ansible.windows.win_service    # NOT win_service
erp.install.openjdk            # NOT openjdk
```

**Role invocation** (> See ansible.md § 9):
| Pattern | When |
|---------|------|
| `roles:` | Static, executes before tasks |
| `ansible.builtin.include_role` | Dynamic, runtime |
| `ansible.builtin.import_role` | Static, parse-time |

**Settings merge:**
```yaml
{role}_values: "{{ {role}_defaults | ansible.builtin.combine({role}_custom, recursive=true) }}"
```
Always `recursive=true` — shallow combine loses nested keys.

**File copy** (> See ansible.md § 11):
Use `erp.install.copy_from_share` role; check `copy_from_share_changed | bool` for conditional restarts.

## 8. RED FLAG / STOP Table

| Excuse | Reality |
|--------|---------|
| "README and defaults suffice" | argument_specs.yml required — HIGH |
| "tests can be written later" | tests/test.yml required now — HIGH |
| "win_shell is standard" | NEVER — always changed=true — HIGH |
| "idempotency is best-effort" | 5 namespaces MUST be idempotent — HIGH |
| "any top-level name works" | MUST be `{role_name}_settings` — HIGH |
| "required: false is fine" | Needs ticket/PR justification — HIGH |
| "no README needed" | HIGH; add dependencies + config + operation |
| "short names resolve fine" | FQCN always — MEDIUM |
| "tags are decorative" | component_* required on ERP plays — MEDIUM |
| "used playbook without component tags" | MEDIUM; add `tags: component_[name]` |
| "shallow combine works" | Must use `recursive=true` — MEDIUM |
| "use $args for PS params" | Use Ansible.Basic.AnsibleModule — HIGH |
