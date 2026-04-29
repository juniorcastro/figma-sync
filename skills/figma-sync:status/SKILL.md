---
name: figma-sync:status
description: "Check what is in sync, pending, or drifted between the prototype and Figma. Read-only — makes no changes."
disable-model-invocation: false
---

# figma-sync:status

Check the sync status of all components, screens, and tokens between the codebase and Figma.

## Arguments

No arguments — always runs across the full project.

## Workflow

1. **Read `figma-sync.json` manifest** from the project root.
2. **For each component/screen in the inventory, check status:**
   - **APPROVED** — synced and approved by the designer.
   - **PENDING** — present in the inventory but not yet pushed to Figma.
   - **REJECTED** — pushed to Figma but the designer rejected it; needs rework.
   - **DRIFT** — code has changed since the last push (compare file modification times or content hashes stored in the manifest against current files).
3. **Optionally check that Figma nodes still exist** by calling `figma_execute` to verify the node IDs recorded in the manifest are still present in the Figma file.
4. **Report a summary table** to the user.

## Output Example

```
figma-sync status

Components:
  button          APPROVED    135 variants, pushed 2026-04-11
  textfield       APPROVED    10 variants, pushed 2026-04-11
  icon            APPROVED    3 variants, pushed 2026-04-11
  checkbox        PENDING     detected in inventory, not pushed

Screens:
  surveys-list    PENDING     detected in inventory, not pushed

Tokens:
  Button Type     OK          6 modes, 8 variables
  Design Tokens   OK          12 variables
```
