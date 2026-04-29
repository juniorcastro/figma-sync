# figma-sync.json Manifest Schema (v1)

The manifest file (`figma-sync.json`) lives at the project root and tracks all sync state between the codebase and Figma.

## Full Schema

```json
{
  "version": "1.0.0",
  "project": "string — project name",
  "stack": {
    "library": "mui | shadcn | chakra | radix | antd | custom | mixed",
    "styling": "tailwind | css-modules | styled-components | css",
    "framework": "next | vite | cra"
  },
  "figma": {
    "fileId": "string — Figma file key",
    "fileName": "string — Figma file name",
    "libraries": {
      "inUse": [
        {
          "name": "string — library display name",
          "fileKey": "string — Figma file key of the library",
          "components": {
            "[componentName]": "string — component key for importComponentByKeyAsync"
          }
        }
      ],
      "available": [
        {
          "name": "string — library display name",
          "libraryKey": "string — library key from search_design_system"
        }
      ]
    }
  },
  "tokens": {
    "collections": {
      "[collectionName]": {
        "collectionId": "string — Figma variable collection ID",
        "modes": { "[modeName]": "string — mode ID" },
        "variables": {
          "[variableName]": {
            "id": "string — Figma variable ID",
            "usage": "string — what this variable controls"
          }
        }
      }
    }
  },
  "components": [
    {
      "name": "string",
      "sourcePath": "string — import path",
      "library": "string — source library",
      "figmaNodeId": "string — ComponentSet node ID",
      "variantAxes": { "[axis]": ["values"] },
      "totalVariants": "number",
      "componentProperties": {},
      "colorTheming": {
        "collection": "string — variable collection name",
        "boundVariables": ["variable names"],
        "sharedVariables": ["variable names"]
      },
      "status": "pending | approved | rejected",
      "lastPushed": "ISO date string or null"
    }
  ],
  "screens": [
    {
      "name": "string",
      "sourcePath": "string — file path",
      "figmaNodeId": "string — frame node ID or null",
      "states": ["empty", "populated", "error", "loading"],
      "usesComponents": ["component names referenced"],
      "status": "pending | approved | rejected",
      "lastPushed": "ISO date string or null"
    }
  ],
  "nodeMap": {
    "[manifestItemKey]": "string — Figma node ID"
  },
  "log": [
    {
      "timestamp": "ISO date string",
      "direction": "push",
      "target": "string — e.g. component/button, screen/dashboard",
      "outcome": "approved | rejected | error",
      "summary": "string — what changed"
    }
  ]
}
```

## Key Notes

### nodeMap enables smart updates instead of full rebuilds

The `nodeMap` maps manifest keys (like `component/Button` or `screen/Dashboard`) to their Figma node IDs. When a component needs updating, the sync process can look up the existing node and modify it in place rather than deleting and recreating everything. This preserves designer annotations, comments, and positioning.

### log is append-only

Never delete or modify existing log entries. Each sync operation appends a new entry. This provides a full audit trail of what was pushed, when, and whether it was accepted.

### Status values

- **pending** — Detected in the codebase but not yet pushed to Figma. The component/screen exists in the manifest as a tracking placeholder.
- **approved** — Pushed to Figma and the designer has approved it. No further action needed until the source code changes.
- **rejected** — Pushed to Figma but the designer rejected it (visual mismatch, wrong structure, etc.). Needs investigation and a re-push.

### Libraries track external assets available to push agents

`figma.libraries` has two sub-sections:

- **inUse** — Libraries with components already placed in the file. Each entry includes a `fileKey` (for cross-referencing) and a `components` map of name → component key. Push agents use these keys with `figma.importComponentByKeyAsync(key)` to create instances of library components (e.g., icons).

- **available** — All team libraries accessible to the file, discovered via `search_design_system`. These are libraries the file *could* use but hasn't yet. Useful for knowing what assets exist before creating raw alternatives.

Both lists are populated during `figma-sync:init` and updated incrementally during pushes (when new library components are used for the first time, add them to `inUse`).

### Variable collection IDs and mode IDs enable fast lookups

Storing Figma variable collection IDs and mode IDs in the manifest avoids repeated calls to the Figma API to resolve them. When creating or updating components that use design tokens, the sync process can bind variables directly using these cached IDs. Only refresh them if a Figma API call fails with a stale ID error.
