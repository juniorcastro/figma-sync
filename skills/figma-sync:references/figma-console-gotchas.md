# figma-console Gotchas

## Instance Sublayer Limitations

Nodes inside component instances have **compound IDs** like `I5002:20480;43:3782`. These are "instance sublayers" and most operations on them fail with:

```
Error: in <operation>: The node (instance sublayer or table cell) with id "I5002:20480;43:3782" does not exist
```

### What FAILS on instance sublayers

| Operation | Error |
|-----------|-------|
| `node.name` | "does not exist" |
| `node.visible = false` | "does not exist" |
| `node.opacity = 0` | "does not exist" |
| `node.remove()` | "does not exist" |
| `node.detachInstance()` | "does not exist" |
| `node.componentProperties` | "does not exist" |
| `figma.getNodeByIdAsync(compoundId)` | Returns `null` |

### What WORKS on instance sublayers

| Operation | Notes |
|-----------|-------|
| `node.type` | Returns correctly (e.g., "INSTANCE", "FRAME") |
| `node.id` | Returns the compound ID string |
| Accessing via `parent.children[index]` | The reference exists, but most properties are inaccessible |

### How to identify compound IDs

Compound IDs start with `I` and contain semicolons: `I{parentId};{childId}` or `I{parentId};{childId};{grandchildId}`. The deeper the nesting, the more semicolons.

**Rule of thumb**: If an ID contains `;`, treat it as potentially inaccessible and use workarounds.

## findAll() Crashes on Deep Instance Trees

**NEVER** use `findAll()` or `findOne()` on a component instance that has deeply nested sublayers:

```js
// WRONG — will crash
const texts = pageInstance.findAll(n => n.type === 'TEXT');

// CORRECT — manual traversal, one level at a time
function safeChildren(node, depth = 0, maxDepth = 2) {
  const results = [];
  if (depth >= maxDepth || !('children' in node)) return results;
  for (const child of node.children) {
    try {
      results.push({ id: child.id, name: child.name, type: child.type });
      results.push(...safeChildren(child, depth + 1, maxDepth));
    } catch (e) {
      // Instance sublayer — skip
      results.push({ id: child.id, type: child.type, inaccessible: true });
    }
  }
  return results;
}
```

## SLOT Default Content Cannot Be Evicted

SLOT frames inside component instances contain default content from the component definition. This default content:
- Cannot be removed (`.remove()` fails)
- Cannot be hidden (`.visible = false` fails)
- Cannot have opacity set (`.opacity = 0` fails)

If you `appendChild()` new content to a SLOT, both the default and new content exist. With auto-layout (HORIZONTAL/VERTICAL), they stack. Without auto-layout, they overlap.

**Workaround**: Use the clone approach — clone an existing reference that already has the correct slot content.

## Font Loading

Must call `figma.loadFontAsync()` before ANY text property change:

```js
// Common fonts in Inventor Portal
await figma.loadFontAsync({ family: "Toyota Type W02", style: "Regular" });
await figma.loadFontAsync({ family: "Toyota Type W02", style: "Semibold" });
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
```

Forgetting this causes silent failures — the text simply doesn't change, no error thrown.

## Page Context Resets Between Calls

Every `figma_execute` call starts with `figma.currentPage` pointing to the **first page** in the file. If your target page is not the first, switch at the start of every call:

```js
const targetPage = figma.root.children.find(p => p.name === "My Page");
await figma.setCurrentPageAsync(targetPage);
// Now work with targetPage.children
```

**WRONG**: `figma.currentPage = page` — throws "not supported"
**CORRECT**: `await figma.setCurrentPageAsync(page)`

## Timeout Management

| Operation | Recommended timeout |
|-----------|-------------------|
| Simple reads (list pages, get properties) | 5000ms (default) |
| Component search, tree traversal | 10000ms |
| Clone + position + validate | 15000ms |
| Heavy operations (multiple clones, bulk text changes) | 30000ms (max) |

Pass timeout in the `figma_execute` call: `{ "code": "...", "timeout": 15000 }`

## figma_execute Code Patterns

```js
// Code is auto-wrapped in async context. Use top-level await and return.

// WRONG
(async () => {
  const page = figma.currentPage;
  return page.name;
})();

// CORRECT
const page = figma.currentPage;
return page.name;
```

## Fills and Strokes Are Immutable

```js
// WRONG — mutating in place
node.fills[0].color = { r: 1, g: 0, b: 0 };

// CORRECT — clone, modify, reassign
const fills = [...node.fills];
fills[0] = { ...fills[0], color: { r: 1, g: 0, b: 0 } };
node.fills = fills;
```

## Property Name Typos Fail Silently

Figma node objects are **frozen** — setting a non-existent property throws `TypeError: object is not extensible` instead of silently succeeding. Common typos:

| Wrong | Correct |
|---|---|
| `clipContent` | `clipsContent` |
| `layoutPosition` | `layoutPositioning` |
| `strokeDash` | `dashPattern` |
| `autoLayout` | `layoutMode` |

Always verify property names against the [Figma Plugin API reference](https://www.figma.com/plugin-docs/api/properties/). If a property set appears to have no effect, check for a `TypeError` in the console.

## figma_set_instance_properties vs figma_execute

For simple property changes on instances, prefer `figma_set_instance_properties`:
- Automatically handles `#nodeId` suffix patterns
- Works with TEXT, BOOLEAN, VARIANT property types
- Simpler and less error-prone than `figma_execute`

Use `figma_execute` when you need:
- Multi-step logic
- Tree traversal
- Clone operations
- Conditional logic
- Operations on non-instance nodes
