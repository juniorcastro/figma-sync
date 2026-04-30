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

### 1. Read Manifest

Read `.figma-sync/figma-sync.json` from the project root. If no manifest exists, report "No manifest found. Run `/figma-sync:init` to set up." and stop.

### 2. Validate Source Paths

For each component and screen in the manifest that has a `sourcePath`:
- Resolve the path relative to the manifest's parent directory (the project root)
- Check if the file exists on disk
- Mark as **STALE** if the file is missing

### 3. Scan for Untracked Components

Scan `src/` for component files (`.tsx` / `.ts` that export React components). Compare against the manifest's component list. Report any components found in the codebase that are not tracked in the manifest as **UNTRACKED**.

Detection: look for files in component directories (`src/components/`, etc.) that export a function returning JSX and are not already listed in the manifest's `components[].sourcePath`.

### 4. Check Component/Screen Status

For each component/screen in the manifest:
- **ADOPTED** — matched to existing Figma component during init, not yet pushed from code
- **APPROVED** — pushed and approved by the designer
- **PENDING** — in the manifest but not yet pushed to Figma
- **REJECTED** — pushed but designer rejected it; needs rework
- **STALE** — sourcePath no longer exists on disk (from step 2)

### 5. Verify Figma Nodes (optional)

If the Figma bridge is connected, call `figma_execute` to verify that node IDs in the manifest's `nodeMap` still exist in the Figma file. Report any **MISSING** nodes (deleted from Figma but still referenced in manifest).

Skip this step if the bridge is not connected — report "Figma verification skipped (bridge not connected)."

### 6. Report Summary

Print a structured summary table:

```
figma-sync status
Manifest: .figma-sync/figma-sync.json (42 entries)

Components:
  App Bar         ADOPTED     2 variants, node 43:9713
  Table           ADOPTED     2 variants, node 43:5655
  EmptyState      PENDING     not pushed
  OldComponent    STALE       src/components/OldComponent.tsx not found

Screens:
  ServiceCatalog  PENDING     not pushed

Untracked:
  src/components/NewThing.tsx  (not in manifest)

Tokens:
  Localization    OK          2 modes (EN, JP)

Figma Nodes:
  36 verified, 0 missing (or "skipped — bridge not connected")

Summary:
  37 adopted, 5 pending, 0 stale, 0 untracked
```
