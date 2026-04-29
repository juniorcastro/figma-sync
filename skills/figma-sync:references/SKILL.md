---
name: figma-sync:references
description: "Reference documentation for figma-sync subagents — gotchas, creation patterns, extraction format, manifest schema, tool guide, token mapping, workarounds. Not user-invocable."
metadata:
  internal: true
---

# figma-sync:references

This skill directory contains reference documentation used by figma-sync subagents. These files are NOT meant to be read by the orchestrator — pass the file paths to agents instead.

| Doc | Purpose |
|-----|---------|
| figma-console-gotchas.md | Essential rules for figma_execute (return values, fonts, colors, etc.) |
| figma-creation-patterns.md | Tested snippets for creating components in Figma |
| extraction-format.md | JSON schema for Storybook extraction output |
| manifest-schema.md | Full schema for figma-sync.json |
| mcp-tools.md | Quick reference for figma-console MCP tools |
| token-mapping.md | CSS/theme token to Figma variable mapping |
| workarounds-catalog.md | Instance sublayer and SLOT limitation workarounds |
