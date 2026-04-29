# figma-sync Skill Roadmap

## Context

figma-sync is a set of Claude Code skills that sync React prototypes to Figma. The skills are being packaged for distribution via `npx skills add juniorcastro/figma-sync`. The repo structure is ready but shouldn't be published until the skill system is proven end-to-end.

**Current state of the skill system:**
- `figma-sync-init` — proven for fresh files (tested, works). Does NOT handle files with existing components.
- `figma-sync-push` for **library components** — proven (Button, TextField, Select, DatePicker, Menu, Icon all approved)
- `figma-sync-push` for **custom components** — partially proven (untested: containers with SLOTs for custom components, custom SVG-based components)
- `figma-sync-push` for **screens** — unproven (pipeline exists in the SKILL.md but never executed)
- `figma-sync-pull` — doesn't exist yet
- `figma-sync-status` / `figma-sync-log` — lightweight, low risk

All skills must be application/library agnostic. They detect the stack at runtime (MUI, shadcn, Chakra, etc.) and adapt.

---

## Milestone 0: Scan & adopt existing Figma components in init
**Status:** not started

**What:** Extend `figma-sync-init` to detect components that already exist in the Figma file and match them to React components in the codebase, instead of assuming a blank file.

**Why this is foundational:** Most designers running figma-sync will already have a Figma file with components — from a design system, a handoff file, or their own work. If init ignores what's already there, push creates duplicates. This makes figma-sync impractical for the majority of real-world projects.

**How it should work (added to init between current Step 5 and Step 6):**

1. **Scan Figma file for local components** — Walk all pages for COMPONENT_SET and COMPONENT nodes. For each, collect: name, node ID, variant axes (from variant names), child count, page location.

2. **Match against codebase inventory** — Compare the Figma component names against the library + custom component inventory from Step 2. Matching strategies:
   - **Exact name match** — Figma "Button" ↔ code Button (highest confidence)
   - **Fuzzy name match** — Figma "Primary Button" ↔ code Button with variant=primary
   - **Interactive confirmation** — For ambiguous matches, present the designer with a side-by-side: "Is Figma 'Input Field' the same as your MUI TextField?" with screenshots

3. **For matched pairs:**
   - Populate manifest `components[].figmaNodeId` with the existing node ID
   - Set status to `"adopted"` (new status value — means "existed before figma-sync, not pushed by us")
   - Add to `nodeMap`
   - Skip these components during push — they're already in Figma

4. **For unmatched code components:**
   - Set status to `"pending"` as before — these need to be pushed

5. **For unmatched Figma components:**
   - Report them: "These Figma components have no matching React component: [list]"
   - Don't delete or modify them — just inform

**Manifest schema change:** Add `"adopted"` as a valid status value alongside pending/approved/rejected. Adopted components are treated like approved for the Screen Push Guard (they exist in Figma, instances can be created from them).

**What to validate:**
- Scanning correctly identifies ComponentSets and standalone Components
- Name matching handles both exact and near-matches
- Interactive confirmation works for ambiguous cases
- Adopted components are used by the Screen Push Guard
- Push skips adopted components

**Done when:** Init run on a file with pre-existing components correctly maps them to code components and records them in the manifest.

---

## Milestone 1: Prove custom component push
**Status:** not started

**What:** Push custom (non-library) components through the full pipeline to validate the skill handles them correctly.

**Why the skill needs this:** The push SKILL.md has instructions for custom components (detect via `data-name`, variant props, structural complexity) but the detection, story generation, and extraction patterns haven't been battle-tested. Library components (MUI) have well-defined APIs and prop types. Custom components are freeform — the skill needs to handle:
- Components without TypeScript prop types (inline in view files)
- SVG-based components (custom Radio drawn with paths, not MUI)
- Composite components with sub-component dependencies
- Components that are functions with conditional rendering, not standalone files

**What to validate in the skill:**
- Does Phase 1 (state completeness audit) work for components without formal prop types?
- Does Phase 2 (story generation) produce valid stories for inline/freeform components?
- Does Phase 3 (extraction) capture SVG-based rendering correctly?
- Does Phase 4 (Figma creation) handle components that don't map to a library's token system?

**Skill changes expected:** Refinements to push SKILL.md based on what breaks — likely around story generation heuristics and extraction edge cases for custom components.

**Done when:** At least 3 custom components (atomic + composite) approved via the skill pipeline.

---

## Milestone 2: Prove screen push
**Status:** not started (blocked by Milestone 1)

**What:** Push at least 2 screens through the pipeline — one simple, one complex — to validate the screen composition workflow.

**Why the skill needs this:** Phase 4 for screens says "compose from synced component instances" but this has never been tested. Key unknowns:
- How does the skill instruct agents to find and instantiate approved components in Figma?
- How are component instances positioned and sized within the screen frame?
- How does the skill handle variable mode assignment (e.g., a Button with type=error)?
- What happens with static content that isn't a component (text blocks, dividers, spacing)?
- How does the Screen Push Guard perform in practice?

**What to validate in the skill:**
- Screen Push Guard correctly blocks when components are missing
- Agent prompts for screen composition produce proper instances (not flat frames)
- Variable modes are set correctly on component instances
- Visual validation catches composition errors (wrong spacing, misaligned components)

**Skill changes expected:** Likely need to add concrete patterns/snippets to `figma-creation-patterns.md` for screen composition (instantiating components, positioning, setting modes). The push SKILL.md's Phase 4 screen section is sparse — it will need more detail.

**Done when:** At least 2 screens approved, composed from component instances (not flat frames).

---

## Milestone 3: Create figma-sync-pull skill
**Status:** not started

**What:** A new skill that detects changes designers made in Figma and produces a diff report for approval.

**Arguments:**
```
figma-sync-pull                         # pull everything that changed
figma-sync-pull tokens                  # pull token/variable changes
figma-sync-pull components              # pull all changed components
figma-sync-pull component Button        # pull a specific component
figma-sync-pull screens                 # pull all changed screens
figma-sync-pull screen Login            # pull a specific screen
```

**Pipeline:**

1. **Read baseline** — Load the manifest and the stored extraction JSON for each synced item. This is the "last known state" from the most recent push.

2. **Extract current Figma state** — For each item in scope:
   - **Tokens:** Read current Figma variable values via figma_execute, compare against manifest token values
   - **Components:** Screenshot the current Figma ComponentSet, extract styles from each variant via figma_execute (fills, strokes, padding, text properties, opacity), produce a current-state extraction JSON
   - **Screens:** Screenshot the current Figma screen frame, extract composition (which component instances, their positions, their variable modes)

3. **Diff** — Compare current Figma state against the stored extraction JSON:
   - Property-level diff: which fills, paddings, font sizes, colors changed
   - Structural diff: variants added/removed, components added/removed from screens
   - Visual diff: side-by-side screenshots (before from stored extraction, after from current Figma)

4. **Present diff report** — Show the designer:
   - Summary table of changes per item
   - Property-by-property breakdown (old value → new value)
   - Screenshots for visual comparison
   - For each change: Accept / Reject / Skip

5. **Apply decisions:**
   - **Accept** → Update the extraction JSON with the new values, update manifest status to reflect the pull, log the change. The accepted changes become the new baseline. Code changes to match are the designer's next step (manual or assisted).
   - **Reject** → Re-push the component/screen to Figma to restore it to match code (triggers figma-sync-push for that item)
   - **Skip** → No action, leave for later

6. **Manifest update** — Append pull operation to log with timestamp, direction (PULL), target, outcome, summary of accepted/rejected changes.

**Execution model:** Same orchestrator + subagent pattern as push. Orchestrator reads manifest and makes decisions. Agents do the Figma extraction and screenshot work.

**Reference docs needed:**
- `pull-diff-format.md` — Schema for the diff output (property name, old value, new value, change type)
- Existing `figma-console-gotchas.md` and `mcp-tools.md` apply to extraction agents

**Files to create:**
- `skills/figma-sync-pull/SKILL.md`
- `skills/figma-sync-references/pull-diff-format.md`

**Done when:** Diff report correctly detects a manual Figma change and accept/reject flow works.

---

## Milestone 4: Publish to repo
**Status:** not started (blocked by Milestones 0-3)

**What:** Copy all proven skills to the `juniorcastro/figma-sync` repo, clean up project-specific content, tag v1.0.0, push.

**Steps:**
1. Copy all SKILL.md files from the test project to the repo's `skills/` directories
2. Add the new `figma-sync-pull/` skill directory to the repo
3. Copy any new/updated reference docs (including `pull-diff-format.md`)
4. Audit all files for project-specific content and generalize:
   - Remove references to specific app names, fonts, Figma file keys
   - Ensure all instructions reference "the project's font" or "detected font" instead of hardcoded values
   - Ensure manifest path references use `.figma-sync/` consistently
5. Update README.md to include the pull skill in the usage section and skill table
6. Commit, tag `v1.0.0`, push to GitHub
7. Test: run `npx skills add juniorcastro/figma-sync` in a clean project directory, verify all 6 skills install and `/figma-sync-init` is invocable

**Done when:** Fresh `npx skills add` installs all 6 skills; init runs successfully in a new project.

---

## Changelog

| Date | Update |
|------|--------|
| 2026-04-13 | Roadmap created. Repo structure ready at `/Users/junior.castro/Documents/Development/figma-sync/`. Init + library component push proven. |
| 2026-04-29 | Added Milestone 0: scan & adopt existing Figma components. This is foundational — most real-world files already have components. |
