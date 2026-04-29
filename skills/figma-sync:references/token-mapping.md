# Design Token Mapping: Code to Figma Variables

How design tokens from known UI libraries map to Figma variable collections and modes.

## MUI (Material UI)

### Color tokens

| MUI Token | Figma Variable | Notes |
|---|---|---|
| `palette.primary.main` | `type/main` | Primary brand color |
| `palette.primary.dark` | `type/main-hover` | Contained button hover background |
| `palette.primary.contrastText` | `type/on-main` | Text/icon color on primary surfaces |
| `palette.action.disabledBackground` | `disabled/bg` | `rgba(0,0,0,0.12)` |
| `palette.action.disabled` | `disabled/fg` | `rgba(0,0,0,0.26)` |

### Interaction overlays

| State | Derivation |
|---|---|
| Hover overlay | `main` @ 4% opacity |
| Active/pressed overlay | `main` @ 12% opacity |
| Border (outlined variant) | `main` @ 50% opacity |

### Color types as variable modes

MUI color types map to Figma variable modes within a single collection:

- primary
- secondary
- error
- warning
- info
- success

Each mode sets the `type/main`, `type/main-hover`, `type/on-main` etc. values to the corresponding palette colors. This allows a single Button component set in Figma to switch color via mode rather than duplicating variants.

## shadcn/ui + Tailwind CSS

### Token source

CSS custom properties defined in `globals.css` (or equivalent theme file):

```css
:root {
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  --primary: 222.2 47.4% 11.2%;
  --primary-foreground: 210 40% 98%;
  --secondary: 210 40% 96.1%;
  --secondary-foreground: 222.2 47.4% 11.2%;
  --destructive: 0 84.2% 60.2%;
  --destructive-foreground: 210 40% 98%;
  --muted: 210 40% 96.1%;
  --muted-foreground: 215.4 16.3% 46.9%;
  --accent: 210 40% 96.1%;
  --accent-foreground: 222.2 47.4% 11.2%;
  --border: 214.3 31.8% 91.4%;
  --input: 214.3 31.8% 91.4%;
  --ring: 222.2 84% 4.9%;
}
```

### Mapping to Figma

| CSS Custom Property | Figma Variable |
|---|---|
| `--primary` | `primary` |
| `--primary-foreground` | `primary-foreground` |
| `--secondary` | `secondary` |
| `--secondary-foreground` | `secondary-foreground` |
| `--destructive` | `destructive` |
| `--destructive-foreground` | `destructive-foreground` |
| `--muted` | `muted` |
| `--muted-foreground` | `muted-foreground` |
| `--accent` | `accent` |
| `--accent-foreground` | `accent-foreground` |
| `--background` | `background` |
| `--foreground` | `foreground` |
| `--border` | `border` |
| `--input` | `input` |
| `--ring` | `ring` |

### Modes

Light and dark themes (`:root` vs `.dark`) map directly to Figma variable modes within the collection.

## General Approach

### Step 1: Scan for theme configuration

Look for theme files in this priority order:

1. `tailwind.config.ts` / `tailwind.config.js` — extended theme colors
2. `globals.css` / `global.css` / `index.css` — CSS custom properties
3. `theme.ts` / `theme.js` / `createTheme()` calls — MUI theme objects
4. `chakra-theme.ts` / `extendTheme()` calls — Chakra UI
5. Component-level `styled()` or `sx` overrides — last resort

### Step 2: Map to Figma variable collections

- Create one collection per token domain (e.g., "Colors", "Spacing", "Typography")
- For color tokens, group by semantic role: primary, secondary, error, etc.

### Step 3: Use modes for color schemes and component themes

- **Color scheme modes**: light, dark (maps to `:root` / `.dark` or MUI `mode`)
- **Component theme modes**: primary, secondary, error, warning, info, success (maps to MUI palette types or shadcn color variants)
- Choose the mode strategy based on how the library organizes its themes

### Step 4: Disabled state tokens

Disabled tokens (`disabled/bg`, `disabled/fg`) are typically shared across all themes and do not vary by color type or light/dark mode. Store them as single-mode variables or in a separate "State" collection.

### Step 5: MUI outlined component border states

For MUI components using the `outlined` variant (TextField, Select, Autocomplete), the border color changes across states in a specific pattern:

| State | Border opacity/color | Notes |
|---|---|---|
| Empty/rest | `rgba(0,0,0,0.23)` | Light gray |
| **Hover** | `rgba(0,0,0,0.87)` | Much darker — almost black |
| **Filled** | `rgba(0,0,0,0.87)` | **Same as hover** |
| Focused | Primary color, 2px width | e.g. `rgb(25,118,210)` |
| Error | Error color | e.g. `rgb(211,47,47)` |
| Disabled | `rgba(0,0,0,0.26)` | Dimmed |

**Key insight**: Hover and filled states share the same darker border color. Use a single variable (e.g., `color/input-border-hover`) for both to keep the token set minimal. This pattern applies across all MUI outlined components, not just TextField.
