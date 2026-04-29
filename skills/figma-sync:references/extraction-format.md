# Extraction JSON Format

Schema for the output of `figma-sync:extract`. Each component produces one JSON file.

## Top-Level Structure

```json
{
  "component": {
    "name": "Button",
    "sourcePath": "src/components/ui/button.tsx",
    "library": "@mui/material",
    "extractedAt": "2026-04-11T12:00:00Z",
    "storybookUrl": "http://localhost:6006"
  },
  "variantAxes": {
    "variant": ["contained", "outlined", "text"],
    "size": ["small", "medium", "large"]
  },
  "defaults": {
    "variant": "contained",
    "size": "medium"
  },
  "variants": [ /* see Variant Object below */ ],
  "fontMapping": {
    "Toyota Type W02:500": {
      "family": "Inter",
      "style": "Medium",
      "reason": "Toyota Type W02 not available in Figma"
    }
  }
}
```

## Variant Object

```json
{
  "id": "variant-contained--size-medium",
  "props": { "variant": "contained", "size": "medium" },
  "isDuplicate": false,
  "duplicateOf": null,
  "screenshotPath": ".figma-sync/screenshots/button/variant-contained--size-medium.png",
  "styles": {
    "box": {
      "width": 75,
      "height": 36,
      "borderRadius": { "topLeft": 4, "topRight": 4, "bottomLeft": 4, "bottomRight": 4 }
    },
    "layout": {
      "display": "inline-flex",
      "flexDirection": "row",
      "alignItems": "center",
      "justifyContent": "center",
      "gap": 8,
      "paddingTop": 6,
      "paddingRight": 16,
      "paddingBottom": 6,
      "paddingLeft": 16
    },
    "background": {
      "type": "solid",
      "color": { "r": 0.098, "g": 0.463, "b": 0.824, "a": 1 }
    },
    "border": {
      "width": 0,
      "style": "none",
      "color": null
    },
    "text": {
      "content": "Button",
      "fontFamily": "Roboto",
      "fontWeight": 500,
      "fontSize": 14,
      "lineHeight": 24,
      "letterSpacing": 0.4,
      "color": { "r": 1, "g": 1, "b": 1, "a": 1 },
      "textTransform": "uppercase"
    },
    "opacity": 1,
    "boxShadow": "none"
  },
  "figma": {
    "fills": [{ "type": "SOLID", "color": { "r": 0.098, "g": 0.463, "b": 0.824 } }],
    "strokes": [],
    "cornerRadius": 4,
    "layoutMode": "HORIZONTAL",
    "primaryAxisAlignItems": "CENTER",
    "counterAxisAlignItems": "CENTER",
    "paddingLeft": 16,
    "paddingRight": 16,
    "paddingTop": 6,
    "paddingBottom": 6,
    "itemSpacing": 8,
    "textFills": [{ "type": "SOLID", "color": { "r": 1, "g": 1, "b": 1 } }],
    "fontSize": 14,
    "fontName": { "family": "Roboto", "style": "Medium" },
    "letterSpacing": { "value": 0.4, "unit": "PIXELS" },
    "textCase": "UPPER"
  }
}
```

## Color Conversion

CSS `rgb(R, G, B)` to Figma `{ r, g, b }`:
- Divide each channel by 255
- Example: `rgb(25, 118, 210)` → `{ r: 0.098, g: 0.463, b: 0.824 }`

CSS `rgba(R, G, B, A)` — alpha goes in `opacity` on the paint, not on the color.

## Font Weight to Style Mapping

| CSS font-weight | Figma style |
|----------------|-------------|
| 100 | "Thin" |
| 200 | "ExtraLight" |
| 300 | "Light" |
| 400 | "Regular" |
| 500 | "Medium" |
| 600 | "SemiBold" |
| 700 | "Bold" |
| 800 | "ExtraBold" |
| 900 | "Black" |

## Text Transform Mapping

| CSS text-transform | Figma textCase |
|-------------------|---------------|
| "uppercase" | "UPPER" |
| "lowercase" | "LOWER" |
| "capitalize" | "TITLE" |
| "none" | "ORIGINAL" |

## Hover State Extraction

CSS `:hover` pseudo-class styles **cannot** be triggered by JavaScript events (`mouseenter`, `pointerenter`, `dispatchEvent`). These events fire JS handlers but do not activate CSS pseudo-class matching.

The **only** reliable way to extract hover styles is to use Playwright's physical mouse simulation:

```js
// WRONG — does NOT trigger CSS :hover
element.dispatchEvent(new PointerEvent('pointerenter', { bubbles: true }));

// CORRECT — physically moves cursor, triggers CSS :hover
await page.hover('.MuiOutlinedInput-root');
await page.waitForTimeout(300); // wait for CSS transitions
// Now getComputedStyle() returns hover-state values
```

When creating Storybook stories for hover states, the story itself just needs to exist as an extraction target. The actual hover simulation happens during Playwright extraction.

## Extraction Persistence

Always persist extraction JSON as an artifact in `.figma-sync/` at the project root. Future re-pushes can diff against it to detect what actually changed, avoiding unnecessary full re-extraction.
