---
name: openedge-guid
description: Use when generating OpenEdge GUID, GUID without hyphens, UUID hex, OID seed, or 32-character lowercase hexadecimal identifiers
---

# OpenEdge GUID

## Purpose

Generate OpenEdge GUID values (32 lowercase hex, no hyphens) via central helper. Never invent manually — always use helper to ensure validity.

## When to Use

- Generating OpenEdge GUID or UUID hex identifiers
- Creating OID seed or 32-character lowercase hex values
- Any artifact needing new OpenEdge identifier

---

## Format

- 32 lowercase hexadecimal characters
- No hyphens
- Example: `cc3cb3626ac171b1e81443e3385361d1`

## Command

```bash
~/.config/opencode/bin/openedge-guid [COUNT]
```

If `OPENCODE_CONFIG_DIR` is set, use:

```bash
"$OPENCODE_CONFIG_DIR/bin/openedge-guid" [COUNT]
```

`COUNT` defaults to `1`. One GUID is printed per line.

## Rules

- Use this helper whenever an OpenEdge artifact needs a new GUID value.
- Never hand-type, copy an old GUID, use counters, or use a hyphenated UUID.
- If the helper is missing, stop and ask the user to run `update-opencode.sh`.
