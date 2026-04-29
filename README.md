# figma-sync

Automatically produce a pixel-perfect, handoff-ready Figma file from a running React prototype. The app is the source of truth. figma-sync extracts real rendered styles via Playwright and creates proper Figma components with variants, design tokens, and variable bindings.

## Install

```
npx skills add juniorcastro/figma-sync
```

This installs 5 skills into `.claude/skills/`:

| Skill | Invoke | Purpose |
|-------|--------|---------|
| figma-sync:init | `/figma-sync:init` | One-time setup: detect stack, scan inventory, extract tokens, create Figma file + variable collections, write manifest |
| figma-sync:push | `/figma-sync:push [target]` | Push components or screens to Figma. Full pipeline: story gen, extraction, Figma creation, validation, approval |
| figma-sync:status | `/figma-sync:status` | Read-only: what's synced, pending, drifted |
| figma-sync:log | `/figma-sync:log [target]` | History of all sync operations |
| figma-sync:references | (internal) | Reference docs for subagents: not user-invocable |

## Prerequisites

- **Figma Desktop** with the [Bridge plugin](https://www.figma.com/community/plugin/1500272763498261498/figma-console-bridge) installed and running
- **Node.js project** with `npm run dev` working
- **Two MCP servers** connected to Claude Code (see setup below)

Storybook and Playwright are installed automatically during `figma-sync:init` if not already present.

### MCP server setup

figma-sync requires two MCP servers connected to Claude Code:

**1. figma-console-mcp (primary):** Runs Plugin API code directly in Figma Desktop. Used for all component creation, variable management, screenshots, and file manipulation.

```json
{
  "mcpServers": {
    "figma-console": {
      "command": "npx",
      "args": ["-y", "figma-console-mcp"]
    }
  }
}
```

Add this to your Claude Code MCP settings. See [figma-console-mcp](https://github.com/southleft/figma-console-mcp) for full setup instructions.

**2. Figma MCP (secondary):** Official Figma MCP server. Used only for design system library search during init, which figma-console-mcp does not support. Enable it in Claude Code via `/mcp` and selecting the Figma plugin.

## Usage

```
# 1. Initialize the project (run once)
/figma-sync:init

# 2. Push components to Figma
/figma-sync:push component Button
/figma-sync:push component TextField
/figma-sync:push components              # push all pending

# 3. Push screens (after all components are approved)
/figma-sync:push screen Login
/figma-sync:push screens                 # push all pending

# 4. Check sync status
/figma-sync:status

# 5. View sync history
/figma-sync:log
/figma-sync:log component Button
```

## How it works

1. **Init** scans your codebase for components (library + custom), extracts design tokens, creates a Figma file with variable collections, and writes a `figma-sync.json` manifest.

2. **Push** runs a 6-phase pipeline for each component:
   - **Scope**: audit states, build variant matrix, confirm with designer
   - **Story generation**: create Storybook stories for each variant
   - **Browser extraction**: use Playwright to capture computed styles from the running Storybook
   - **Figma creation**: build the component in Figma with auto layout, fills, text styles, icon instances, and variable bindings
   - **Visual validation**: screenshot comparison + designer approval
   - **Manifest update**: record the push result

3. **Screens** are composed from approved component instances: never flat recreations. A mandatory guard checks that all components a screen uses are approved before allowing the push.

## Architecture

figma-sync uses an **orchestrator + subagent** pattern. The main skill orchestrates decisions and verification. Subagents handle heavy execution: Storybook scripts, Playwright extraction, figma_execute calls. Reference docs in `figma-sync:references/` are passed to agents by file path: never loaded into the orchestrator's context.

## Pin to a version

```
npx skills add juniorcastro/figma-sync#v1.0.0
```

## Update

```
npx skills update figma-sync
```
