# Workarounds Catalog

Tested solutions for common problems when customizing component instances in Figma via figma-console.

## 1. Clone-and-Use (Preferred)

**Problem**: Need a fully composed page but can't build it from scratch due to instance sublayer limitations.

**Solution**: Find an existing reference design and clone it.

```js
// Find the reference
const page = figma.root.children.find(p => p.name === "Reference Page");
await figma.setCurrentPageAsync(page);
const reference = page.children.find(c => c.name === "Legal Agreements — Populated");

// Clone it
const clone = reference.clone();

// Move to target page
const targetPage = figma.root.children.find(p => p.name === "Target Page");
targetPage.appendChild(clone);
clone.x = 0;
clone.y = 0;
clone.name = "Legal Agreements — Populated (Clone)";

return { cloneId: clone.id, type: clone.type }; // Should be INSTANCE
```

**When to use**: Always try this first. Only fall back to other approaches when no suitable reference exists.

**Limitations**: You get an exact copy — any modifications need additional workarounds below.

## 2. Component Properties Override

**Problem**: Need to change text, variant, or boolean on a cloned instance.

**Solution**: Use `componentProperties` on the instance (or `figma_set_instance_properties` tool).

```js
const instance = await figma.getNodeByIdAsync("5002:20958");

// Check what properties are available
const props = instance.componentProperties;
// Returns: { "Title#12345": { type: "TEXT", value: "..." }, "variant": { type: "VARIANT", value: "default" } }

// Set properties
instance.setProperties({
  "Title#12345": "New Title",
  "variant": "populated"
});
```

**When to use**: When the target instance directly exposes the property you want to change. Check `componentProperties` first.

**Limitations**: Only works for properties exposed at the instance level. Deeply nested sub-instance properties are NOT exposed on the parent.

## 3. Pre-Insertion Text Setting

**Problem**: Need text on a component that will be inserted into a deep slot, but text nodes are inaccessible once inside the slot.

**Solution**: Create and configure the instance at page level (where it's fully accessible), THEN insert it into the target slot.

```js
// 1. Get the component
const component = await figma.getNodeByIdAsync("43:3751"); // Page/Header component
const headerInstance = component.createInstance();

// 2. Set properties while it's at page level (fully accessible)
headerInstance.setProperties({
  "Page Title#66306:0": "Legal Agreements",
  "Page Description#66306:3": "Policies participants must accept."
});

// 3. Also set text directly if needed
await figma.loadFontAsync({ family: "Toyota Type W02", style: "Semibold" });
// Find text nodes while accessible
function findTexts(node) {
  const results = [];
  if (node.type === 'TEXT') results.push(node);
  if ('children' in node) {
    for (const child of node.children) results.push(...findTexts(child));
  }
  return results;
}
const texts = findTexts(headerInstance);
// Set text on found nodes...

// 4. NOW insert into the slot
const targetSlot = await figma.getNodeByIdAsync("I5002:20480;43:3783");
targetSlot.appendChild(headerInstance);
```

**When to use**: When you need custom content in a SLOT and the default content doesn't have the right text.

**Limitations**: 
- The SLOT's default content remains (cannot be removed), causing overlap or stacking
- The slot's auto-layout determines how multiple children are arranged
- If the slot has HORIZONTAL layout, both default and inserted content appear side by side

## 4. Standalone-Then-Insert

**Problem**: Need to build a complex component (e.g., a table with custom data) and place it inside an instance's SLOT.

**Solution**: Build the entire structure as a standalone frame on the page, then move it into the slot.

```js
// 1. Build a table frame at page level
const tableFrame = figma.createFrame();
tableFrame.name = "Custom Table";
tableFrame.layoutMode = "VERTICAL";
tableFrame.resize(1084, 400);

// 2. Add header row, data rows, etc.
// ... (full access to all nodes while at page level)

// 3. Insert into the content slot
const contentSlot = await figma.getNodeByIdAsync("I5002:20480;43:3783");
contentSlot.appendChild(tableFrame);
```

**When to use**: When you need a fully custom structure (not an instance) inside a slot.

**Limitations**: Same SLOT default content issue — the existing default remains.

## 5. Selective Detach

**Problem**: Need to modify a specific nested instance that's inaccessible, but don't want to detach the entire page.

**Solution**: Navigate to the specific nested instance and detach only that one.

```js
// Navigate safely to the target
const wrapper = await figma.getNodeByIdAsync("I5002:20480;43:3780");
const slot = wrapper.children[1]; // Content slot
const table = slot.children[0]; // Table instance inside slot

// Detach ONLY the table (not the whole page)
try {
  const detached = table.detachInstance();
  // Now detached is a regular FRAME — fully modifiable
  return { detachedId: detached.id, type: detached.type };
} catch (e) {
  return { error: e.message };
}
```

**When to use**: Last resort — when you absolutely must modify deeply nested content and no other workaround works.

**Limitations**: 
- May fail on instance sublayers (compound ID error)
- Detaching breaks the instance link — no more component updates
- Only detach the minimum necessary scope
- Node IDs may change after detaching

## 6. Clone-and-Swap

**Problem**: Need a different variant or configured version of a nested component.

**Solution**: Find another instance elsewhere in the file that has the configuration you want, clone it, and swap it in.

```js
// Find a reference that already has the right configuration
const sourcePage = figma.root.children.find(p => p.name === "Reference Page");
await figma.setCurrentPageAsync(sourcePage);

// Find the specific configured component
const source = sourcePage.findOne(n => n.name === "Status Chip — Active" && n.type === "INSTANCE");

// Clone it
const clone = source.clone();

// Move to target page and position
const targetPage = figma.root.children.find(p => p.name === "Target Page");
await figma.setCurrentPageAsync(targetPage);
targetPage.appendChild(clone);

// Position or insert where needed
```

**When to use**: When you need a pre-configured variant that exists somewhere in the file but isn't in your current context.

**Limitations**: Must find a suitable source instance first.

## Decision Tree

```
Need to recreate a full page?
  → Clone-and-Use (#1)

Need to change text/variant on the clone?
  → Check componentProperties → Component Properties Override (#2)
  → Property not exposed? → Is it in a SLOT?
    → Yes → Pre-Insertion Text Setting (#3)
    → No → Navigate safely → Selective Detach (#5) as last resort

Need custom structure in a SLOT?
  → Standalone-Then-Insert (#4)

Need a differently configured nested component?
  → Clone-and-Swap (#6)
```
