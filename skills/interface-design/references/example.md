# Craft in Action

How subtle layering translates to real decisions. Learn the thinking, not the code. Your values will differ; the approach won't.

---

## The Mindset

You should barely notice the system working. Vercel's dashboard: you don't think "nice borders" -- you understand the structure. Supabase: you don't think "good surface elevation" -- you know what's above what. Invisible craft means it's working.

---

## Dashboard with Sidebar and Dropdown

### Surface Decisions

Each elevation jump: a few percentage points of lightness. Barely visible in isolation, but stacked surfaces reveal hierarchy. Whisper-quiet shifts you feel rather than see.

**Don't:** Dramatic jumps between elevations. Different hues for different levels. Keep the same hue, shift only lightness.

### Border Decisions

Low opacity rgba borders blend with their background. Barely there -- defining edges without demanding attention. Solid hex borders look harsh.

**Test:** From arm's length, if borders are the first thing you notice, reduce opacity. If you can't find where regions end, increase slightly.

### Sidebar: Same Background as Canvas

Different-colored sidebars fragment visual space into "sidebar world" and "content world." Same background with subtle border separation keeps the sidebar part of the app, not a separate region. Vercel and Supabase do this.

### Dropdown: One Level Above Parent

If dropdown and card share the same surface level, the dropdown blends in. One level higher (just light enough to feel "above") without dramatic difference. Overlay borders need slightly more opacity for containment.

---

## Form Controls

### Input Background: Darker, Not Lighter

Inputs are inset -- they receive content. Slightly darker background signals "type here" without heavy borders.

### Focus: Subtle but Visible

A noticeable increase in border opacity suffices. No glowing rings or dramatic color.

---

## Adapt to Context

Your product may need warmer hues, cooler hues, different lightness progression, or light mode (principles invert: higher elevation = shadow, not lightness).

**The principle is constant:** barely different, still distinguishable.

---

## Craft Check

1. Blur your eyes or step back
2. Hierarchy visible?
3. Anything jumping out?
4. Regions distinguishable?

Hierarchy visible and nothing harsh = subtle layering works.
