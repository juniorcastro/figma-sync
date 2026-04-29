---
name: figma-sync:log
description: "View the history of all sync operations. Shows what was pushed, when, and the outcome."
disable-model-invocation: false
---

# figma-sync:log

View the chronological history of all sync operations recorded in the manifest.

## Arguments

```
figma-sync:log                          # full history
figma-sync:log components               # history of all component syncs
figma-sync:log component button         # history of a specific component
figma-sync:log screens                  # history of all screen syncs
figma-sync:log screen dashboard         # history of a specific screen
```

Plural form (`components`, `screens`) returns all entries of that type. Singular form + name (`component button`, `screen dashboard`) returns entries for one specific item.

## Workflow

1. **Read `figma-sync.json`** and locate the `log` array.
2. **Filter by target** if an argument was provided (by type, or by type + name).
3. **Display entries chronologically** (oldest first).

## Output Example

```
figma-sync log

2026-04-11 14:30  PUSH  component/button      APPROVED   135 variants
2026-04-11 15:45  PUSH  component/textfield   APPROVED   10 variants
2026-04-11 16:00  PUSH  component/icon        APPROVED   3 variants
2026-04-12 09:15  PUSH  component/button      APPROVED   updated: +end icon variants, +variable modes
```

## Log Entry Format

Log entries are **append-only** in the manifest. Each entry records:

- **timestamp** — when the operation occurred
- **direction** — the operation type (e.g., `PUSH`)
- **target** — what was synced (e.g., `component/button`, `screen/dashboard`)
- **outcome** — the result (`APPROVED`, `REJECTED`, `PENDING`)
- **summary** — a brief description of what changed
