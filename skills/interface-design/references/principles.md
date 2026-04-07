# Core Craft Principles

Quality floor. These apply regardless of design direction.

---

## Surface & Token Architecture

### Primitive Foundation

Every color traces to primitives:
- **Foreground** -- text (primary, secondary, muted)
- **Background** -- surfaces (base, elevated, overlay)
- **Border** -- edges (default, subtle, strong)
- **Brand** -- primary accent
- **Semantic** -- functional (destructive, warning, success)

No invented colors. Map everything to primitives.

### Surface Elevation

```
Level 0: Base (app canvas)
Level 1: Cards, panels (same visual plane)
Level 2: Dropdowns, popovers (floating above)
Level 3: Nested dropdowns, stacked overlays
Level 4: Highest elevation (rare)
```

Dark mode: higher = lighter. Light mode: higher = lighter or shadow. Elevated surfaces need visual distinction from what's beneath.

### Subtlety

Study Vercel, Supabase, Linear -- surfaces barely different but still distinguishable, borders light but not invisible.

**Surfaces:** Differences between levels: a few percentage points of lightness, not dramatic jumps. Dark mode: surface-100 ~7% lighter, surface-200 ~9%, surface-300 ~12%.

**Borders:** Use low opacity (0.05-0.12 alpha dark mode, slightly higher light). Borders disappear when you're not looking, findable when needed.

**Squint test:** Hierarchy perceivable, no single border/surface jumping out.

**Common mistakes:**
- Borders too visible (1px solid gray instead of subtle rgba)
- Dramatic surface jumps
- Different hues for different surfaces
- Harsh dividers where subtle borders suffice

### Text Hierarchy

Four levels: primary (highest contrast), secondary (slightly muted), tertiary (metadata), muted (disabled/placeholder). Use all four.

### Border Progression

Default -> subtle/muted -> strong -> stronger (focus rings). Match intensity to boundary importance.

### Control Tokens

Dedicated tokens for control background, border, focus. Tune interactive elements independently from layout surfaces.

### Context-Aware Bases

Different app areas may need different bases: marketing (richer), dashboard (neutral), sidebar (may differ). Same hierarchy, different starting point.

### Alternative Backgrounds

Beyond shadows, contrasting backgrounds create depth. Use for empty states, code blocks, inset panels, visual grouping without borders.

---

## Spacing

Base unit (4px/8px), strict multiples. Scale by context: micro (icon gaps), component (buttons/cards), section (groups), major (distinct areas). Random values signal no system.

## Padding

Symmetrical. If top is 16px, others match. Exception: when content naturally creates visual balance.

```css
/* Good */
padding: 16px;
padding: 12px 16px;

/* Bad */
padding: 24px 16px 12px 16px;
```

## Border Radius

Sharp = technical, round = friendly. Scale: small (inputs/buttons), medium (cards), large (modals). Consistent -- inconsistent radius is as jarring as inconsistent spacing.

## Depth Strategy

Choose ONE:

**Borders-only:** Clean, technical, dense. Linear, Raycast style.

**Subtle single shadows:** `0 1px 3px rgba(0,0,0,0.08)`. Approachable, gentle depth.

**Layered shadows:** Multiple layers, premium depth. Stripe, Mercury style.

**Surface color shifts:** Background tints establish hierarchy without shadows.

```css
/* Borders-only */
border: 0.5px solid rgba(0, 0, 0, 0.08);

/* Single shadow */
box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);

/* Layered shadows */
box-shadow:
  0 0 0 0.5px rgba(0, 0, 0, 0.05),
  0 1px 2px rgba(0, 0, 0, 0.04),
  0 2px 4px rgba(0, 0, 0, 0.03),
  0 4px 8px rgba(0, 0, 0, 0.02);
```

## Card Layouts

Design each card's internal structure for its content. Keep surface treatment consistent: border weight, shadow, radius, padding, typography.

## Controls

Never use native `<select>`, `<input type="date">` -- they render OS-native elements that cannot be styled. Build custom: trigger button + dropdown, input + calendar popover, styled div + state.

Custom select triggers: `display: inline-flex` with `white-space: nowrap`.

## Typography

- **Headlines:** Heavy weight, tight letter-spacing
- **Body:** Comfortable weight
- **Labels/UI:** Medium weight, smaller sizes
- **Data:** Monospace, `tabular-nums` for alignment

Combine size, weight, and letter-spacing. If you squint and can't distinguish headline from body, hierarchy too weak.

## Monospace for Data

Numbers, IDs, codes, timestamps: monospace. `tabular-nums` for columnar alignment.

## Icons

Clarify, not decorate. One icon set. Standalone icons get subtle background containers. Align optically, not mathematically.

## Animation

Micro-interactions: ~150ms. Larger transitions: 200-250ms. Smooth deceleration easing. No spring/bounce in professional interfaces.

## Contrast Hierarchy

Four levels: foreground (primary) -> secondary -> muted -> faint. Use all four consistently.

## Color

Gray builds structure. Color communicates (status, action, emphasis, identity). Unmotivated color is noise. Color reinforcing the product's world is character.

## Navigation Context

Screens need grounding: sidebar/top nav, location indicator (breadcrumbs, active state), user context. Sidebars: same background as content, subtle border separation.

## Dark Mode

Borders over shadows. Desaturate semantic colors slightly. Same hierarchy structure, inverted values.
