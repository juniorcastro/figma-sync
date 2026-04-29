---
name: figma-sync:push
description: "Push components or screens from the prototype to Figma. Handles Storybook story generation, browser-based style extraction, Figma component/screen creation, visual validation, and designer approval. Supports incremental and batch pushes."
disable-model-invocation: false
---

# figma-sync:push

Push components or screens from the React prototype to Figma with full fidelity.

## Arguments

```
figma-sync:push                         # push everything pending
figma-sync:push components              # push all components
figma-sync:push component button        # push a specific component
figma-sync:push screens                 # push all screens
figma-sync:push screen dashboard        # push a specific screen
```

Plural = all of that type. Singular + name = one specific item.

## Prerequisites Check

Before proceeding, verify the figma-sync:references skill is installed:

1. Check that `.claude/skills/figma-sync:references/figma-console-gotchas.md` exists using Read
2. If missing, stop and tell the user: "figma-sync references are missing. Run `npx skills add juniorcastro/figma-sync` to install all skills."

## Execution Model — READ THIS FIRST

You are the **orchestrator**. All heavy execution — scripts, browser automation, figma_execute calls, file writes — MUST be delegated to subagents.

The parent context:
- Reads manifest, story names, import statements (lightweight context gathering)
- Makes architectural decisions (variant axes, token bindings, layout)
- Writes agent prompts with explicit specs and reference doc paths
- Verifies agent output via screenshots
- Manages designer approval

The parent NEVER:
- Calls figma_execute, figma_take_screenshot, or any figma-console tool directly
- Writes story files, extraction scripts, or any code
- Reads reference docs into its own context (pass paths to agents instead)

If you catch yourself doing any of the above — STOP. Delegate it.

### Reference Docs for Agent Prompts

Do NOT read these into the orchestrator context. Pass the file paths to agents so they read them.

| Doc | Include when agent is... |
|-----|--------------------------|
| `.claude/skills/figma-sync:references/figma-console-gotchas.md` | Doing any figma-console work |
| `.claude/skills/figma-sync:references/workarounds-catalog.md` | Hitting instance sublayer or SLOT limitations |
| `.claude/skills/figma-sync:references/extraction-format.md` | Reading/writing extraction JSON |
| `.claude/skills/figma-sync:references/figma-creation-patterns.md` | Creating Figma components or screens |
| `.claude/skills/figma-sync:references/manifest-schema.md` | Reading/writing the manifest |
| `.claude/skills/figma-sync:references/token-mapping.md` | Mapping CSS/theme tokens to Figma variables |
| `.claude/skills/figma-sync:references/mcp-tools.md` | Choosing which figma-console MCP tools to use |

### Critical Rules to Include in Every Agent Prompt

- **Never assume the UI library** — scan actual import statements to determine component source (MUI, shadcn, custom, etc.)
- **Include figma-console-gotchas.md** in every agent that touches Figma — covers return values, font loading, color ranges, fills immutability, page context resets

## Screen/Flow Push Guard (MANDATORY)

Before pushing any screen or flow, check that **every component used by that screen** — both library AND custom — has been pushed and approved in the manifest. Screens must be composed from Figma component instances — never flat recreations.

**Enforcement:**
1. Read the screen's source file and identify ALL components it uses:
   - **Library components**: imported from external packages (`@mui/material`, `@/components/ui`, etc.)
   - **Custom components**: defined inline or imported from project files — detect via `data-name` attributes, variant props (`selected`, `disabled`), structural complexity (own layout/borders/backgrounds), and reuse across screens
   - **Nested sub-components**: walk the component tree recursively — if SelectionCard contains a custom Radio and IconBadge, those must also be in the manifest
2. For each component (library or custom), look it up in the manifest's `components` array
3. If ANY component is missing from the manifest or has `status` other than `"approved"`:
   - **STOP. Do not push the screen.**
   - Report ALL blocking components (library and custom) and their status
   - Example error:

```
ERROR: Cannot push screen "SurveyWizard"

The following components used by this screen are not yet synced and approved:

  Library:
    TimePicker      NOT IN MANIFEST    @mui/x-date-pickers    line 8

  Custom:
    SelectionCard   NOT IN MANIFEST    data-name="Selection Card"    line 511
    Radio           NOT IN MANIFEST    custom SVG radio              line 437
    IconBadge       NOT IN MANIFEST    sub-component of SelectionCard

Push and approve these components first:
  figma-sync:push component TimePicker
  figma-sync:push component SelectionCard
  figma-sync:push component Radio
  figma-sync:push component IconBadge

Screens must be composed from approved Figma component instances.
Pushing a screen without its components would produce flat frames
instead of proper instances — breaking the design system.
```

4. This check applies to `push screens`, `push screen <name>`, and `push` (all pending) when screens are in scope
5. There is no --force override for this guard — components must be approved first

## Internal Pipeline

### Phase 1: Scope (ORCHESTRATOR)

The orchestrator does this directly — it's lightweight reading.

- Read figma-sync.json manifest
- Determine what needs pushing (new, changed, or explicitly requested)
- For components: check if Storybook stories exist, note which need generation
- For screens: identify route/view components and their states

#### Full Component Tree Audit (MANDATORY)

Before any push, build a complete dependency tree of what's needed. This covers BOTH library components (MUI, shadcn, etc.) AND custom/project-specific components.

1. **For a single component push**: identify its sub-components (e.g., DatePicker uses Icon → must be pushed first)
2. **For a screen push**: walk the screen's source file and enumerate every visual component:
   - Library imports (from `@mui/*`, `@/components/ui/*`, etc.)
   - Custom components (functions with `data-name`, variant props, own visual structure)
   - Nested sub-components (e.g., SelectionCard → Radio + IconBadge)
3. **Cross-check against manifest**: every component in the tree must exist with `status: "approved"`
4. **Report gaps**: list any missing or non-approved components before proceeding

This audit prevents partial pushes where screens reference components that don't exist in Figma yet.

#### State Completeness Audit (MANDATORY for components)

Before finalizing variant axes, enumerate ALL visual states the component supports — not just the "obvious" ones. Missing states cause expensive rework (stories, extraction, and Figma creation must all be redone).

Use a **matrix approach** — identify independent axes, then enumerate valid cross-products:

| Axis | Values | Examples |
|---|---|---|
| **Content** | empty, filled/populated | Placeholder text vs selected value |
| **Interaction** | rest, hover, focused, active/pressed | Default, mouse over, keyboard focus, click |
| **Validation** | none, error, warning, success | Normal, red border + helper text |
| **Availability** | enabled, disabled, readonly, loading | Interactive, grayed out, spinner |

Interaction states like hover and active are **modifiers** that multiply across content and validation states (e.g., empty+hover, filled+hover, error+hover, error-filled+hover). Disabled states have no hover/focused cross — exclude invalid combinations.

Also identify **distinct text roles**: if the component displays different text depending on state (e.g., placeholder when empty vs value when filled), each role needs its own TEXT component property — named after the component's prop/API name. Scan the component's types/props to find these roles.

#### Variable-Length Container Detection

If the component's children are variable-length (menus, lists, tables, card grids), plan a **container + sub-component architecture**:
- Sub-components (e.g., Menu/Item, Menu/Subheader, Menu/Divider) are created as standalone Components or ComponentSets
- The container is a single Component (not ComponentSet) with a **SLOT** inside (`component.createSlot('name')`)
- The SLOT's `preferredValues` points to the sub-components, constraining what designers can insert
- Create sub-components first — they become the slot's preferredValues

**Present the proposed state matrix and text roles to the designer for confirmation before proceeding to Phase 2.** It is far cheaper to add a state now than to redo the entire pipeline later.

### Phase 2: Story Generation (AGENT)

**Orchestrator decides:** variant axes, story names, decorator style, which stories are missing.
**Agent executes:** writes the .stories.tsx file, verifies it compiles.

**IMPORTANT — Interactive states (hover, active) do NOT get stories.** CSS pseudo-class states (`:hover`, `:active`) cannot be captured as static Storybook stories — `play` functions using JS events (`pointerenter`, `dispatchEvent`) do NOT trigger CSS pseudo-class matching. Only generate stories for states achievable via **props or DOM attributes** (disabled, error, focused via prop, filled via value, etc.). Hover/active variants are created during Phase 3 extraction by Playwright simulation on the base story.

- **Every component that gets a Figma node must have stories and extraction** — including containers, wrappers, and layout components. Source code inspection is not a substitute for measuring actual rendered output. Never skip the pipeline for "simple" components.
- For components without stories: generate .stories.tsx
  - Detect variant axes from the component's props/API (prop types, CVA definitions, TypeScript types)
  - Cross variant axes for full matrix, **excluding interactive pseudo-class states**
  - Each story wrapped in decorator with **white bg (`#ffffff`)** + padding. Only use gray (`#f0f0f0`) for borderless sub-components that would be invisible against white (e.g., standalone menu items, subheaders, dividers shown outside their container). Containers with their own surface (background, shadow, border) always use white.
  - **Use realistic, distinct content** in each story variant. For icon stories, pick the correct icon for each action (e.g., ContentCut for Cut, ContentCopy for Copy). For text, use realistic labels of varying length. Stories are reference material — duplicated placeholder content produces misleading extraction screenshots.
- For screens: generate stories for each state (empty, populated, error, loading)
- Verify stories render in Storybook

### Phase 3: Browser Extraction (AGENT)

**Orchestrator decides:** which story URLs to extract, expected variant count (stories + hover/active variants derived from base stories).
**Agent executes:** starts Storybook if needed, runs browser extraction, writes JSON.
**Orchestrator verifies:** extraction JSON has all expected variants.

- Ensure Storybook is running on port 6006
- Use Playwright for extraction (preferred over agent-browser):
  - Navigate to each story's iframe URL
  - Screenshot the rendered output
  - Extract computed styles via `page.evaluate()` + `getComputedStyle`: dimensions, colors, padding, gap, typography, border, opacity
  - **For hover/active variants:** navigate to the base story, then use `await page.hover(selector)` or `await page.click(selector)` to physically trigger CSS pseudo-class states before extracting. Wait 300ms for CSS transitions.
  - Normalize: px to numbers, rgb to Figma 0-1 range, flex to auto layout props
- Write extraction JSON
- Total variant count = stories + hover variants (e.g., 20 stories + 8 hover extractions = 28 variants)

### Phase 4: Figma Creation (AGENT)

**Orchestrator decides:** section placement, variant grid layout, component properties, variable bindings, presentation standard specs. Include all of this in the agent prompt along with reference doc paths.
**Agent executes:** all figma_execute calls — creating nodes, setting properties, combining variants.
**Orchestrator verifies:** screenshots after agent completes, checks node counts and names.

- Read extraction JSON
- **MUST follow presentation standard** from `.claude/skills/figma-sync:references/figma-creation-patterns.md` (see Presentation Standard section)
- For components:
  - Find or create section on Components page (white bg)
  - Load fonts
  - Create variant Components with auto layout, fills, strokes, text
  - Use Icon ComponentSet instances (with INSTANCE_SWAP) instead of raw vectors for any icons. Look up icon component keys from `figma.libraries.inUse` in the manifest and import via `figma.importComponentByKeyAsync(key)`. Never draw SVG paths manually when a library icon exists. Check the component's actual source package (`node_modules/@mui/<package>/icons/`) to identify the correct icon name — sub-packages often use different icons than @mui/icons-material.
  - Apply Figma text styles to all text nodes — never hardcode font size/weight. Check existing styles first, create new ones only if needed.
  - Combine as ComponentSet via figma.combineAsVariants()
  - Apply dashed #8A38F5 stroke to ComponentSet, dashPattern [8, 4]
  - Add component properties (TEXT with {PropertyName} convention, INSTANCE_SWAP via isExposedInstance)
  - Bind fills/strokes/text to variable collections
  - Arrange variants in neat grid: 24px internal padding, 24px gaps, states left to right, sizes top to bottom
  - Set fixed width on all variants (layoutSizingHorizontal = "FIXED")
  - Size section to fit with 50px padding around the ComponentSet
- For screens:
  - Create frame on Screens page matching viewport dimensions
  - Compose from synced component instances where possible
  - Set variable modes for component instances (e.g., Button Type = error)
  - Add annotation layers if needed

### Phase 5: Visual Validation + Designer Approval (ORCHESTRATOR + AGENT)

**Agent executes:** takes screenshots, applies corrections per orchestrator feedback.
**Orchestrator does:** vision comparison, decides what needs fixing, presents result to designer, manages approval gate.

- Screenshot each created Figma element
- Claude vision compares against Storybook screenshots
- Self-correction loop: up to 3 iterations for automatic fixes (color, spacing, alignment)
- Present result to designer for approval
- Designer can: approve, reject with feedback, or skip
- Feedback loop: Claude applies corrections, re-validates, re-presents
- Only approved items are marked "approved" in manifest

### Phase 5b: Orphan Cleanup (AGENT)

**Orchestrator decides:** run cleanup now.
**Agent executes:** scans page, deletes orphans, takes verification screenshot.
**Orchestrator verifies:** screenshot shows only Sections remain.

After every push — including failed/retried attempts — scan the target page for orphan elements:

1. Iterate all direct children of the page
2. Any node that is NOT a Section is an orphan (failed creates, detached text, loose components)
3. Remove all orphans
4. Screenshot to verify clean page — only Sections should remain
5. Log any orphans found and removed

This step runs automatically after Phase 5 and before Phase 6. Never skip it.

### Phase 6: Manifest Update (ORCHESTRATOR)

The orchestrator does this directly — it's a lightweight file write based on agent-reported node IDs.

- Update figma-sync.json:
  - Add/update component/screen entries with Figma node IDs
  - Set status: approved/rejected/pending
  - Append to sync log: timestamp, direction (push), target, outcome

## Agent Usage Inside Push

- Use subagents for extraction (one per component/screen)
- Use subagents for Figma creation (batch variant creation)
- Use subagents for color/variable binding (repetitive across variants)
- ALWAYS verify agent output with screenshot before proceeding
- ALWAYS check for orphan artifacts after agent writes
- Keep the approval gate in the main conversation (never delegate approval to an agent)
- Remember: the parent orchestrates and verifies — agents execute. No exceptions

## Conflict Handling

- If Figma node was manually modified since last sync, flag and ask before overwriting
- --force flag to override
