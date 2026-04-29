# figma-console MCP Tool Reference

Which tool to use for each operation, with recommended timeouts.

## Tool Selection Guide

| Operation | Tool | Timeout |
|---|---|---|
| Read file structure, pages | `figma_execute` | 5s |
| Search components in library | `figma_search_components` | 10s |
| Import library component | `figma_execute` (`figma.importComponentByKeyAsync`) | 10s |
| Create component with auto layout | `figma_execute` | 15-30s |
| Set instance properties | `figma_set_instance_properties` | 5s |
| Create/manage variables | `figma_execute` | 10s |
| Set variable modes | `figma_execute` | 5s |
| Screenshot for validation | `figma_take_screenshot` | 5s |
| Set text content | `figma_set_text` | 5s |
| Set fills/strokes | `figma_set_fills` / `figma_set_strokes` | 5s |
| Rename node | `figma_rename_node` | 5s |
| Delete node | `figma_delete_node` | 5s |

## Notes

### figma_execute is the swiss army knife

Use `figma_execute` for anything that doesn't have a dedicated tool: creating nodes, setting auto layout properties, managing variables, importing components, complex multi-step operations. It runs arbitrary Plugin API code in the Figma context.

### Prefer dedicated tools for simple operations

For straightforward single-property changes, the dedicated tools (`figma_set_text`, `figma_set_fills`, `figma_set_strokes`, `figma_rename_node`, `figma_delete_node`, `figma_set_instance_properties`) are safer than `figma_execute` because:

- They validate inputs before execution
- They handle error cases consistently
- They produce predictable return values
- Less risk of accidentally modifying unrelated nodes

### Batch operations: max 6-8 variants per figma_execute call

When creating multiple variants in a component set, limit each `figma_execute` call to 6-8 variants. Larger batches risk timeouts (the default Plugin API timeout is ~30s). If a component has 20+ variants, split across multiple calls.

### Always use async node access

- Use `getNodeByIdAsync()` (not `getNodeById()`) for dynamic-page access. The sync version only works for nodes on the currently loaded page; the async version loads the page automatically.
- Use `getMainComponentAsync()` (not `.mainComponent`) for instances. The sync property returns `null` if the main component's page hasn't been loaded yet.

### Common patterns

**Find or create a page:**
```js
await figma.loadAllPagesAsync();
let page = figma.root.children.find(p => p.name === 'Components');
if (!page) {
  page = figma.createPage();
  page.name = 'Components';
}
await figma.setCurrentPageAsync(page);
```

**Import and instantiate a library component:**
```js
const component = await figma.importComponentByKeyAsync('component_key_here');
const instance = component.createInstance();
```

**Create a variable collection with modes:**
```js
const collection = figma.variables.createVariableCollection('Colors');
// First mode is created automatically, rename it
collection.renameMode(collection.modes[0].modeId, 'primary');
// Add additional modes
const secondaryModeId = collection.addMode('secondary');
```
