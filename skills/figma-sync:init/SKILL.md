---
name: figma-sync:init
description: "One-time project setup. Detects tech stack, scans component + screen inventory, extracts design tokens, creates Figma file with variable collections, and writes the figma-sync.json manifest."
disable-model-invocation: false
---

# figma-sync:init

One-time project setup skill for figma-sync. Run once per project to bootstrap the Figma file and manifest.

## Prerequisites Check

Before proceeding, verify the figma-sync:references skill is installed:

1. Check that `.claude/skills/figma-sync:references/figma-console-gotchas.md` exists using Read
2. If missing, stop and tell the user: "figma-sync references are missing. Run `npx skills add juniorcastro/figma-sync` to install all skills."

## Execution Model — READ THIS FIRST

You are the **orchestrator**. All heavy execution — scripts, browser automation, figma_execute calls, file writes — MUST be delegated to subagents.

The parent context:
- Reads package.json, import statements (lightweight context gathering)
- Makes architectural decisions (token grouping, collection structure, page layout)
- Writes agent prompts with explicit specs and reference doc paths
- Verifies agent output via screenshots
- Manages user confirmation gates (e.g., "is the Figma file open?")

The parent NEVER:
- Calls figma_execute, figma_take_screenshot, or any figma-console tool directly
- Runs Storybook install commands or writes config files
- Reads reference docs into its own context (pass paths to agents instead)

If you catch yourself doing any of the above — STOP. Delegate it.

### Reference Docs for Agent Prompts

Do NOT read these into the orchestrator context. Pass the file paths to agents so they read them.

| Doc | Include when agent is... |
|-----|--------------------------|
| `.claude/skills/figma-sync:references/figma-console-gotchas.md` | Doing any figma-console work |
| `.claude/skills/figma-sync:references/figma-creation-patterns.md` | Creating Figma pages or structure |
| `.claude/skills/figma-sync:references/manifest-schema.md` | Writing the manifest |
| `.claude/skills/figma-sync:references/token-mapping.md` | Mapping CSS/theme tokens to Figma variables |
| `.claude/skills/figma-sync:references/mcp-tools.md` | Choosing which figma-console MCP tools to use |

### Critical Rules to Include in Every Agent Prompt

- **Never assume the UI library** — scan actual import statements to determine component source (MUI, shadcn, custom, etc.). Different files may use different libraries
- **Include figma-console-gotchas.md** in every agent that touches Figma — covers return values, font loading, color ranges, fills immutability, page context resets

## Step 1: Detect Stack (ORCHESTRATOR)

The orchestrator does this directly — lightweight codebase scanning.

Scan `package.json` and actual import statements in `src/` to detect:

- **Component library**: MUI, shadcn/ui, Chakra, Radix, Ant Design, custom, or mixed
- **Styling**: Tailwind CSS, CSS Modules, styled-components, plain CSS
- **Framework**: Next.js, Vite, CRA

Never assume which library is used. Always scan actual import statements in `src/` to confirm. Different files may use different libraries (mixed usage is common).

## Step 2: Scan Inventory (ORCHESTRATOR)

The orchestrator does this directly — lightweight grepping and code reading.

Build three inventories covering **all** visual components — library AND custom. Missing a component here means it won't get pushed to Figma, breaking screen composition later.

### 2a. Library Component Inventory

Grep all `.tsx` / `.ts` files in `src/` for import statements from external packages. For each:
- `name` — component name
- `sourceLibrary` — which package (e.g., `@mui/material`, `@mui/x-date-pickers`, `@/components/ui`)
- `importPath` — full import path
- `usedIn` — list of files that import it
- `usageCount` — how many files reference it
- `classification` — atomic (button, input, icon) or composite (date picker, menu)

### 2b. Custom Component Inventory

Scan prototype/view files (`src/imports/`, `src/components/`, etc.) for **project-defined components** that are visual building blocks — not library wrappers. Detection signals:

1. **`data-name` attributes** — These map directly to Figma layer names from the original design and indicate a designed component (e.g., `data-name="Selection Card"`, `data-name="App Bar"`, `data-name="Step"`)
2. **Variant props** — Functions accepting `selected`, `checked`, `active`, `disabled`, `state`, `variant` props indicate a component with visual states that needs Figma variants
3. **Reuse** — The same visual pattern appearing multiple times (e.g., `SelectionCard` and `TemplateCard` sharing a card+radio+icon structure)
4. **Structural complexity** — Components with their own layout, borders, backgrounds, shadows — not just text wrappers

For each custom component found:
- `name` — component name (prefer the `data-name` value, fallback to function name)
- `sourceLibrary` — `"custom"`
- `filePath` — where it's defined
- `usedIn` — which screens/views use it
- `classification` — atomic (radio, badge, stepper step) or pattern (selection card, app bar, action bar)
- `hasVariants` — true if it accepts state/variant props
- `subComponents` — list of child components it contains (e.g., SelectionCard contains Radio, IconBadge)

**Common custom components to look for:**
- Cards/tiles with selectable states
- Custom radio/checkbox implementations (drawn with SVG instead of MUI)
- Navigation elements (app bar, stepper, breadcrumbs)
- Action bars (footer buttons, toolbars)
- Section headers, dividers with labels
- Badge/chip/tag elements

### 2c. Screen Inventory

Identify route components or page-level views:
- `name` — screen/page name
- `filePath` — path to the component file
- `route` — associated route path (if router is used)
- `components` — list of ALL components used (both library and custom)

### Component Tree Diagram

After inventorying, produce a dependency tree showing which screens use which components, and which components contain sub-components. This makes the push order clear — leaves first, then composites, then screens.

Example:
```
Screen: SurveyWizard
  ├── AppBar (custom)
  │   └── Icon (library)
  ├── Stepper (custom)
  │   └── StepperStep (custom)
  ├── SelectionCard (custom)
  │   ├── IconBadge (custom)
  │   ├── Radio (custom)
  │   └── Icon (library)
  ├── TemplateCard (custom)
  │   ├── IconBadge (custom)
  │   └── Radio (custom)
  ├── TextField (library)
  └── Button (library)
```

## Step 3: Install Storybook (AGENT)

**Orchestrator decides:** whether Storybook is needed, which config options to set (aliases, styles, fonts).
**Agent executes:** runs install commands, writes config files, verifies startup.
**Orchestrator verifies:** Storybook is running on port 6006.

If Storybook is not already configured in the project:

1. Run `npx storybook@latest init --yes`
2. Configure `.storybook/main.ts` to inherit Vite aliases from `vite.config.ts` (or webpack aliases from the relevant config)
3. Configure `.storybook/preview.ts` to import app styles (CSS files, fonts, theme providers)
4. Verify Storybook starts successfully on port 6006

If Storybook is already present, validate the configuration is correct and styles are loaded.

## Step 4: Extract Design Tokens (AGENT)

**Orchestrator decides:** which files to scan, token grouping strategy, collection structure.
**Agent executes:** reads theme/config files, extracts all token values, writes structured output.
**Orchestrator verifies:** token list is complete and groupings make sense.

Scan the codebase for design tokens:

- CSS custom properties (from `:root`, `globals.css`, theme files)
- Tailwind config (`tailwind.config.ts` / `tailwind.config.js`) — colors, spacing, typography, etc.
- Theme files from known libraries (MUI `createTheme`, Chakra `extendTheme`, etc.)

For each token extract:
- **Colors** — all color values including semantic aliases (primary, secondary, destructive, etc.)
- **Typography** — font families, sizes, weights, line heights
- **Spacing** — spacing scale values
- **Border radius** — radius values
- **Shadows** — box shadow definitions

Group tokens by purpose (color theming, spacing scale, typography scale, etc.) and map them to Figma variable-compatible structures.

## Step 5: Detect Linked Libraries (AGENT)

**Orchestrator decides:** which search queries to run, how to categorize results.
**Agent executes:** runs search_design_system queries, scans file for remote instances, writes structured output.
**Orchestrator verifies:** library list is complete and categorized.

Detect all external libraries linked to the Figma file. There is no single "list all libraries" API — use two complementary approaches:

### 5a: Scan instances already in the file (Plugin API)

Scan all pages for INSTANCE nodes with `remote === true` to find which library components are already placed:

```js
await figma.loadAllPagesAsync();
const remoteComponents = new Map(); // key -> { name, count }
for (const page of figma.root.children) {
  const instances = page.findAll(n => n.type === 'INSTANCE');
  for (const inst of instances) {
    const mc = await inst.getMainComponentAsync();
    if (mc && mc.remote) {
      const existing = remoteComponents.get(mc.key) || { name: mc.name, count: 0 };
      existing.count++;
      remoteComponents.set(mc.key, existing);
    }
  }
}
```

This gives component keys and names but NOT the source library file key. Cross-reference with step 5b.

### 5b: Discover available team libraries (REST API)

Use `mcp__figma__search_design_system` with broad queries to discover all accessible libraries:

- Search with diverse terms: `"icon"`, `"button"`, `"input"`, `"color"`, `"text"`, `"layout"`
- Each result includes `libraryName` and `libraryKey`
- Deduplicate by `libraryKey`
- Use `includeLibraryKeys` to restrict subsequent searches to specific libraries

### 5c: Cross-reference and categorize

Match the component keys from 5a against the libraries from 5b (by searching component names within specific libraries). Produce:

- **inUse**: Libraries with components already in the file — include `fileKey` and `components` map (name → componentKey)
- **available**: All other team libraries — include `libraryKey` and `name`

Icon libraries are especially important to identify, as push agents need them for INSTANCE_SWAP.

## Step 6: Scan & Adopt Existing Components (AGENT)

**Orchestrator decides:** whether the file has existing components worth adopting, matching strategy, confirmation of ambiguous matches.
**Agent executes:** figma_execute calls to scan pages, collect component metadata, take screenshots for confirmation.
**Orchestrator verifies:** match quality, presents ambiguous matches to user for confirmation.

If the Figma file already has components (most real-world files do), detect them and match them to the codebase inventory from Step 2 instead of treating the file as blank.

### 6a: Scan Figma file for local components

Walk ALL pages (not just a "Components" page — files often organize components across sub-pages like "↳ Table", "↳ App Bar", etc.):

```js
const components = [];
for (const page of figma.root.children) {
  await page.loadAsync();
  const nodes = page.findAll(n => n.type === 'COMPONENT_SET' || (n.type === 'COMPONENT' && n.parent.type !== 'COMPONENT_SET'));
  for (const node of nodes) {
    components.push({
      name: node.name,
      nodeId: node.id,
      type: node.type,
      page: page.name,
      variantAxes: node.type === 'COMPONENT_SET'
        ? Object.keys(node.componentPropertyDefinitions).filter(k => node.componentPropertyDefinitions[k].type === 'VARIANT')
        : [],
      childCount: node.children?.length || 0
    });
  }
}
```

For each component collect: name, node ID, type (COMPONENT_SET or standalone COMPONENT), page location, variant axes, child count.

### 6b: Match against codebase inventory

Compare Figma component names against the library + custom component inventory from Step 2. Apply matching strategies in order of confidence:

1. **Exact name match** (highest confidence) — Figma "Button" ↔ code `Button`. Case-insensitive, ignore spaces vs camelCase ("File Uploader" ↔ `FileUploader`).

2. **Partial/fuzzy name match** (medium confidence) — Figma "Primary Button" ↔ code `Button` with `variant=primary`. Figma "Project Navigation" ↔ code `Sidebar` (semantic overlap). Only propose if name overlap > 50% or clear semantic equivalence.

3. **Interactive confirmation** (for ambiguous matches) — Present the user with:
   - Figma component name + page location + variant count
   - Candidate code component name + file path + prop types
   - Ask: "Is Figma '[name]' the same as your code component '[name]'?"
   - Take a screenshot of the Figma component for visual reference if needed

### 6c: Record matches in manifest

For each confirmed match:
- Set `status` to `"adopted"` in the manifest components array
- Populate `figmaNodeId` with the existing Figma node ID
- Add to `nodeMap` as `component/[Name]` → node ID
- Record `variantAxes` from the Figma component's property definitions

For unmatched code components:
- Set `status` to `"pending"` as normal — these need to be pushed later

For unmatched Figma components:
- Report them to the user: "These Figma components have no matching React component: [list]"
- Do NOT delete or modify them

### 6d: Adopt implications

Adopted components:
- Are treated identically to `"approved"` for the Screen Push Guard (they exist in Figma, instances can be created from them)
- Are SKIPPED during `figma-sync:push components` — they already exist
- Can be overwritten with `figma-sync:push component [Name] --force` if the designer wants to rebuild them from code
- Their `figmaNodeId` is used by screen composition to instantiate them

### Skip condition

If the Figma file has zero local COMPONENT_SET or COMPONENT nodes (truly blank file), skip this step entirely and proceed to Step 7.

## Step 7: Set Up Figma File (AGENT)

**Orchestrator decides:** page names, collection structure, variable names and modes. Asks user to confirm file is open.
**Agent executes:** all figma_execute calls — creating pages, variable collections, variables, modes.
**Orchestrator verifies:** screenshots show correct page structure and variable collections.

**Prerequisite**: The user must create a new Figma file (or open an existing one) in Figma Desktop with the Bridge plugin running. figma-console cannot create files — it operates on the currently open file.

1. Ask the user to confirm the file is open in Figma Desktop with Bridge plugin connected
2. Verify connection via `figma_get_status` or `figma_list_open_files`
3. Record the file key and name (from `figma_get_file_data` or the Figma URL the user provides)
4. Create page structure:
   - **"Components"** — will hold all component sets (populated later by `figma-sync:push`)
   - **"Screens"** — will hold full-page screen compositions (populated later)
5. Build variable collections from the extracted tokens:
   - Create collections grouped by purpose (e.g., "Colors", "Spacing", "Typography")
   - Create variables within each collection matching the code token names
   - Set up modes if applicable (light/dark themes, component states, etc.)
6. Take a screenshot to verify the file structure is correct

**Note:** If components were adopted in Step 6, the file structure setup still proceeds (variable collections, pages) but the adopted components are left untouched. Do not recreate or modify any component nodes that were matched and marked as `"adopted"`.

## Step 8: Write Manifest (ORCHESTRATOR)

The orchestrator does this directly — lightweight file write based on agent-reported node IDs.

Create `figma-sync.json` at `.figma-sync/figma-sync.json` in the project root, following the v1 schema defined in `.claude/skills/figma-sync:references/manifest-schema.md`. Create the `.figma-sync/` directory if it doesn't exist.

Key fields init must populate:
- `version` — `"1.0.0"`
- `project` — from `package.json` name field
- `stack` — detected library, styling, and framework
- `figma.fileId` and `figma.fileName` — from the open Figma file
- `figma.libraries` — detected linked libraries from Step 5 (`inUse` with component keys, `available` with library keys)
- `tokens.collections` — variable collection IDs, mode IDs, and variable IDs created in Step 7
- `components` — all detected components with `status: "pending"` (not yet pushed), except adopted components which get `status: "adopted"` and their `figmaNodeId` populated from Step 6
- `screens` — all detected screens with `status: "pending"`
- `nodeMap` — pre-populated with `component/[Name]` → node ID entries for adopted components from Step 6; empty for all others (populated later by `figma-sync:push`)
- `log` — single entry recording the init operation

## Outputs

- Figma file with page structure and variable collections set up
- `figma-sync.json` manifest at the project root

No components or screens are pushed to Figma yet -- that is handled by `figma-sync:push`.

## Approval

No designer approval is needed for init. This step is infrastructure only.
