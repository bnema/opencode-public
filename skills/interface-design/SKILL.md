---
name: interface-design
description: Use for interface design -- dashboards, admin panels, SaaS apps, tools, settings pages, data interfaces. NOT for marketing design (landing pages, marketing sites, campaigns).
compatibility: opencode
---

# Interface Design

Build interface design with craft and consistency.

**Use for:** Dashboards, admin panels, SaaS apps, tools, settings pages, data interfaces.
**Not for:** Landing pages, marketing sites, campaigns. Use `/frontend-design` instead.

---

## The Problem

You generate generic output. Your training contains thousands of dashboards; the patterns are strong. You can follow the entire process below and still produce a template. Warm colors on cold structures. Friendly fonts on generic layouts.

Intent lives in prose, but code generation pulls from patterns. The gap between them is where defaults win.

---

## Where Defaults Hide

Defaults disguise themselves as infrastructure.

**Typography feels like a container.** But typography IS your design. Weight, personality, texture shape how the product feels before anyone reads a word. A bakery tool and a trading terminal both need "readable type" -- but warm, handmade type differs from cold, precise type.

**Navigation feels like scaffolding.** But navigation IS your product -- where you are, where you can go, what matters most. A page floating in space is a component demo, not software.

**Data feels like presentation.** A number on screen is not design. What does this number mean? What will the user do with it? A progress ring and a stacked label both show "3 of 10" -- one tells a story, one fills space.

**Token names feel like implementation.** But `--ink` and `--parchment` evoke a world. `--gray-700` and `--surface-2` evoke a template. Someone reading only your tokens should guess what product this is.

Everything is design. The moment you stop asking "why this?" is the moment defaults take over.

---

## Intent First

Before touching code, answer these -- not in your head, out loud.

**Who is this human?** Not "users." Where are they? What's on their mind? A teacher at 7am with coffee differs from a developer debugging at midnight.

**What must they accomplish?** Not "use the dashboard." The verb. Grade submissions. Find the broken deployment. Approve the payment.

**What should this feel like?** "Clean and modern" means nothing. Warm like a notebook? Cold like a terminal? Dense like a trading floor? Calm like a reading app?

If you cannot answer with specifics, stop. Ask the user.

### Every Choice Must Be a Choice

For every decision, explain WHY: layout, color temperature, typeface, spacing scale, information hierarchy. If the answer is "it's common" or "it's clean" -- you defaulted.

**The test:** Swap your choices for the most common alternatives. If the design feels the same, you never made real choices.

### Sameness Is Failure

If another AI given a similar prompt would produce the same output, you failed. Design from intent makes sameness impossible; design from defaults makes everything identical.

### Intent Must Be Systemic

Saying "warm" then using cold colors is not following through. If the intent is warm, surfaces, text, borders, accents, typography -- all warm. Check every token against your stated intent.

---

## Product Domain Exploration

Generic: Task type -> Visual template -> Theme
Crafted: Task type -> Product domain -> Signature -> Structure + Expression

**Produce all four before proposing any direction:**

1. **Domain:** Concepts, metaphors, vocabulary from this product's world. Minimum 5.
2. **Color world:** Colors that exist naturally in this domain. If this product were a physical space, what would you see? List 5+.
3. **Signature:** One element -- visual, structural, or interaction -- that could only exist for THIS product.
4. **Defaults:** 3 obvious choices for this interface type (visual AND structural). Name them to avoid them.

### Proposal Requirements

Reference all four: domain concepts, colors from exploration, your signature, what replaces each default.

**The test:** Remove the product name from your proposal. Can someone identify what it's for? If not, explore deeper.

---

## The Mandate

Before showing the user, ask: "If they said this lacks craft, what would they mean?"

Fix that thing first.

### The Checks

- **Swap test:** Swap typeface/layout for the usual. Would anyone notice? Where swapping wouldn't matter, you defaulted.
- **Squint test:** Blur your eyes. Hierarchy visible? Anything harsh? Craft whispers.
- **Signature test:** Point to five specific elements where your signature appears. Not "the overall feel" -- actual components.
- **Token test:** Read CSS variables aloud. Do they belong to this product's world, or any project?

If any check fails, iterate before showing.

---

## Craft Foundations

### Subtle Layering

The backbone of craft. You should barely notice the system working. When you look at Vercel's dashboard, you don't think "nice borders" -- you understand the structure. Invisible craft means it's working.

**Surface Elevation:** Surfaces stack (base -> cards -> dropdowns). Each jump: a few percentage points of lightness. Dark mode: higher = lighter. Light mode: higher = lighter or shadow.

Key decisions:
- **Sidebars:** Same background as canvas, subtle border for separation -- not a different color.
- **Dropdowns:** One level above parent surface.
- **Inputs:** Slightly darker than surroundings (inset).

**Borders:** Low opacity rgba that blends with background. Build a progression: standard, softer, emphasis, focus ring. Match intensity to boundary importance.

**Squint test:** Blur your eyes. Hierarchy perceivable? Nothing jumps out? Quiet structure.

### Infinite Expression

Every pattern has infinite expressions. A metric display could be hero number, inline stat, sparkline, gauge, progress bar, trend badge, or something new. Before building, ask: What's the ONE thing users do most here?

NEVER produce identical output. Same sidebar, same card grid, same metric boxes signal AI-generated immediately.

### Color Lives Somewhere

Before reaching for a palette, spend time in the product's world. What would you see in the physical version of this space? The palette should feel like it came FROM somewhere, not applied TO something.

Gray builds structure. Color communicates -- status, action, emphasis, identity. One accent color used with intention beats five used without thought.

---

## Before Writing Each Component

**Mandatory checkpoint -- state these every time:**

```
Intent: [who, what they must do, how it should feel]
Palette: [colors from exploration -- WHY they fit]
Depth: [borders / shadows / layered -- WHY]
Surfaces: [elevation scale -- WHY this temperature]
Typography: [typeface -- WHY it fits]
Spacing: [base unit]
```

If you can't explain WHY for each, you're defaulting.

---

## Design Principles

**Token Architecture:** Every color traces to primitives: foreground (text hierarchy), background (surface elevation), border (separation), brand, semantic (destructive, warning, success). No random hex values.

**Text Hierarchy:** Four levels -- primary, secondary, tertiary, muted. Use all four consistently.

**Border Progression:** Standard, subtle, strong, strongest. Match intensity to importance.

**Control Tokens:** Dedicated tokens for control backgrounds, borders, focus -- tuned independently from layout.

**Spacing:** Base unit (4px/8px), strict multiples. Scale: micro (icon gaps), component (buttons/cards), section (groups), major (distinct areas).

**Depth:** Choose ONE: borders-only (dense tools), subtle shadows (approachable), layered shadows (premium cards), surface color shifts (tint hierarchy). Don't mix.

**Border Radius:** Sharp = technical, round = friendly. Scale: small (inputs), medium (cards), large (modals). Consistent.

**Typography:** Headlines: heavy weight, tight tracking. Body: comfortable weight. Labels: medium, small sizes. Data: monospace, `tabular-nums`.

**Cards:** Design each card's internal structure for its content, but keep surface treatment consistent (border weight, shadow, radius, padding).

**Controls:** Never use native `<select>` or `<input type="date">`. Build custom components with styled dropdowns and popovers.

**Icons:** Clarify, not decorate. One icon set. Remove icons that add no meaning.

**Animation:** Fast micro-interactions (~150ms), smooth easing. No spring/bounce in professional interfaces.

**States:** Every interactive element: default, hover, active, focus, disabled. Data: loading, empty, error.

**Navigation:** Screens need grounding -- sidebar/nav, location indicator, user context. Sidebars: same background as content, border separation.

**Dark Mode:** Borders over shadows. Desaturate semantic colors. Same hierarchy structure, inverted values.

---

## Avoid

- Harsh borders (if borders are the first thing you see, too strong)
- Dramatic surface jumps (whisper-quiet changes)
- Inconsistent spacing (clearest sign of no system)
- Mixed depth strategies
- Missing interaction states
- Dramatic drop shadows
- Large radius on small elements
- Pure white cards on colored backgrounds
- Thick decorative borders
- Gradients/color for decoration
- Multiple accent colors
- Different hues for different surfaces (same hue, shift lightness)

---

## Workflow

### Communication
Be invisible. Don't announce modes or narrate process. Jump into work with reasoning.

### Suggest + Ask
Lead with exploration and recommendation, then confirm:
```
Domain: [5+ concepts from the product's world]
Color world: [5+ colors from this domain]
Signature: [one element unique to this product]
Rejecting: [default 1] -> [alt], [default 2] -> [alt], [default 3] -> [alt]

Direction: [approach connecting the above]
```
Then ask: "Does that direction feel right?"

### If Project Has system.md
Read `.interface-design/system.md` and apply.

### If No system.md
1. Explore domain -- produce all four outputs
2. Propose -- reference all four
3. Confirm -- get user buy-in
4. Build -- apply principles
5. Evaluate -- run mandate checks before showing
6. Offer to save

---

## After Completing a Task

Always offer: "Want me to save these patterns for future sessions?"

If yes, write to `.interface-design/system.md`: direction, feel, depth strategy, spacing base, key component patterns.

**Add patterns when:** used 2+ times, reusable across project, has specific measurements. **Don't save:** one-offs, experiments, prop-based variations.

**Consistency checks:** spacing on grid, depth strategy consistent, colors from palette, documented patterns reused.

---

## Deep Dives

- `references/principles.md` -- Code examples, specific values, dark mode
- `references/validation.md` -- Memory management, when to update system.md
- `references/critique.md` -- Post-build craft critique protocol
- `references/example.md` -- Subtle layering in practice
