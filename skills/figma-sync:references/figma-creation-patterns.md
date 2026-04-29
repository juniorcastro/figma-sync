# Figma Creation Patterns

Tested `figma_execute` snippets for creating components. All patterns follow the figma-console rules from the parent skill.

## Create a Component with Auto Layout

```js
// Always load fonts first
await figma.loadFontAsync({ family: "Inter", style: "Medium" });

const comp = figma.createComponent();
comp.name = "variant=contained, size=medium";

// Auto layout
comp.layoutMode = "HORIZONTAL";
comp.primaryAxisAlignItems = "CENTER";
comp.counterAxisAlignItems = "CENTER";
comp.paddingLeft = 16;
comp.paddingRight = 16;
comp.paddingTop = 6;
comp.paddingBottom = 6;
comp.itemSpacing = 8;
comp.layoutSizingHorizontal = "HUG";
comp.layoutSizingVertical = "FIXED";

// Size (set after layout mode)
comp.resize(100, 36);

// Background
comp.fills = [{ type: "SOLID", color: { r: 0.098, g: 0.463, b: 0.824 } }];
comp.cornerRadius = 4;

// Text child
const label = figma.createText();
label.characters = "BUTTON";
label.fontSize = 14;
label.fontName = { family: "Inter", style: "Medium" };
label.fills = [{ type: "SOLID", color: { r: 1, g: 1, b: 1 } }];
label.letterSpacing = { value: 0.4, unit: "PIXELS" };
label.textCase = "UPPER";
comp.appendChild(label);

// TEXT component property
const labelKey = comp.addComponentProperty("Label", "TEXT", "BUTTON");
label.componentPropertyReferences = { characters: labelKey };

return { id: comp.id, name: comp.name };
```

## Combine Components as Variants

```js
// Collect all component nodes by ID
const ids = ["1:2", "1:3", "1:4"]; // from previous creation steps
const comps = await Promise.all(ids.map(id => figma.getNodeByIdAsync(id)));

// Must be on the correct page
const page = figma.root.children.find(p => p.name === "Synced Components");
await figma.setCurrentPageAsync(page);

// Find the section
const section = page.findOne(n => n.type === "SECTION" && n.name === "Button");

// Combine
const cs = figma.combineAsVariants(comps, section || page);
cs.name = "Button";
cs.description = "Synced from React: src/components/ui/button.tsx\nLast synced: 2026-04-11\nfigma-sync-id: Button";

return { componentSetId: cs.id, name: cs.name, childCount: cs.children.length };
```

## Layout Variants in Grid

```js
const cs = await figma.getNodeByIdAsync("{componentSetId}");

// Define axis ordering
const variantOrder = ["contained", "outlined", "text"];
const sizeOrder = ["small", "medium", "large"];
const colWidth = 180;
const rowHeight = 60;

for (const child of cs.children) {
  // Parse variant name: "variant=contained, size=medium"
  const props = {};
  child.name.split(', ').forEach(pair => {
    const [key, val] = pair.split('=');
    props[key.trim()] = val.trim();
  });

  const col = sizeOrder.indexOf(props.size);
  const row = variantOrder.indexOf(props.variant);
  if (col >= 0 && row >= 0) {
    child.x = col * colWidth;
    child.y = row * rowHeight;
  }
}

return { arranged: cs.children.length };
```

## Create or Find Page + Section

```js
const pageName = "Synced Components";
let page = figma.root.children.find(p => p.name === pageName);
if (!page) {
  page = figma.createPage();
  page.name = pageName;
}
await figma.setCurrentPageAsync(page);

// Check for existing section
let section = page.findOne(n => n.type === "SECTION" && n.name === "{ComponentName}");
if (!section) {
  section = figma.createSection();
  section.name = "{ComponentName}";
  section.x = 0;
  section.y = 0;
}

return { pageId: page.id, sectionId: section.id, sectionName: section.name };
```

## Border/Stroke Pattern

For components with visible borders (e.g., outlined variant):
```js
comp.strokes = [{ type: "SOLID", color: { r: 0.098, g: 0.463, b: 0.824 } }];
comp.strokeWeight = 1;
comp.strokeAlign = "INSIDE"; // CSS borders are inside
```

## Transparent Background

For ghost/text buttons with no background:
```js
comp.fills = []; // No fill = transparent
```

## Opacity for Disabled State

```js
comp.opacity = 0.5; // matches CSS disabled:opacity-50
```

## Presentation Standard (MANDATORY)

Every ComponentSet and its enclosing Section must follow this visual standard:

### ComponentSet
- **Dashed stroke**: `#8A38F5` (purple), weight 1, dashPattern `[8, 4]`
- **Internal padding**: 24px between variants and the component outline
- **Grid layout**: `layoutMode: "NONE"`, variants positioned manually in a grid with 24px gaps
- **Grid organization**: primary axis (columns) = states left to right, secondary axis (rows) = sizes top to bottom
- **Fixed variant width**: set `layoutSizingHorizontal = "FIXED"` on each variant to prevent content-hugging

### Section
- **White background**: `fills: [{ type: "SOLID", color: { r: 1, g: 1, b: 1 } }]`
- **50px padding** between section edges and the ComponentSet
- **Sized to fit**: section width = ComponentSet width + 100, section height = ComponentSet height + 100
- **100px gap** between sections on the page — always recheck after adding new sections to ensure no overlaps or inconsistent gaps
- **All sections left-aligned** at x=0

### Text conventions
- Use `{PropertyName}` for variable text (e.g., `{Label}`, `{Value}`, `{Placeholder}`)
- This distinguishes component property text from fixed/decorative text

### Icons
- Always use Icon ComponentSet instances (with INSTANCE_SWAP) instead of raw vectors
- Import icons from the MUI Icons library and swap via the Icon component's property
- Match icon size variant to the component size (small → small, medium → medium)
- Icon ComponentSet sizes must match MUI's exact `SvgIcon` fontSize values: **small=20×20, medium=24×24, large=35×35**. Approximations (18/20/22) cause library icons to get scaled when swapped in, producing visually smaller icons than the browser renders.

```js
// Example: applying presentation standard to a ComponentSet
cs.strokes = [{ type: "SOLID", color: { r: 0.541, g: 0.22, b: 0.961 } }];
cs.strokeWeight = 1;
cs.dashPattern = [8, 4];

// Section
section.fills = [{ type: "SOLID", color: { r: 1, g: 1, b: 1 } }];
const sectionPad = 50;
section.resizeWithoutConstraints(cs.width + sectionPad * 2, cs.height + sectionPad * 2);
```

## Common Gotchas

1. **Set layoutMode BEFORE setting padding** — padding is ignored if layoutMode is not set
2. **Set resize BEFORE layoutSizingVertical = "FIXED"** — order matters
3. **Append children BEFORE setting their layout sizing** — `layoutSizingHorizontal = "FILL"` only works after node is in a parent with auto layout
4. **Colors are 0-1** — `rgb(25, 118, 210)` becomes `{ r: 0.098, g: 0.463, b: 0.824 }`, NOT `{ r: 25, g: 118, b: 210 }`
5. **Fills array is read-only** — always clone: `comp.fills = [...newFills]`
