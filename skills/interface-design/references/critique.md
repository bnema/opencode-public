# Critique

Your first build shipped the structure. Now review it as a design lead reviews a junior's work -- not "does this work?" but "would I put my name on this?"

---

## The Gap

Correct: layout holds, grid aligns, colors don't clash. Crafted: someone cared about every decision to the last pixel. Like a hand-thrown mug versus injection-molded. Both hold coffee. One has presence.

Your first output lives in correct. This pulls it toward crafted.

---

## See the Composition

Does the layout have rhythm? Great interfaces breathe unevenly -- dense tooling yields to open content, heavy balances light, the eye travels with purpose. Monotone density (same card size, same gaps everywhere) means no one decided.

Are proportions doing work? A 280px sidebar says "navigation serves content." A 360px sidebar says "these are peers." If you can't articulate what your proportions declare, they declare nothing.

Is there a clear focal point? The one thing the user came to do should dominate through size, position, contrast, or surrounding space. When everything competes equally, nothing wins.

---

## See the Craft

Spacing grid: every value a multiple of 4, no exceptions. But correctness alone isn't craft. A tool panel at 16px padding feels workbench-tight; the same card at 24px feels like a brochure. Density is a decision, not a constant.

Typography: if size alone separates headline from body from label, the hierarchy is too weak. Weight, tracking, and opacity create layers size can't.

Surfaces: whisper hierarchy. Remove every border mentally. Can you still perceive structure through surface color alone?

Interactive elements: every button, link, clickable region responds to hover and press. Missing states make an interface feel like a photograph of software.

---

## See the Content

Read every string as a user would. Does this screen tell one coherent story? Could a real person at a real company see exactly this data? Content incoherence breaks the illusion faster than any visual flaw.

---

## See the Structure

Open the CSS and find the lies: negative margins undoing parent padding, calc() as workarounds, absolute positioning escaping layout flow. The correct answer is always simpler than the hack.

---

## Again

Ask: "If they said this lacks craft, what would they point to?"

Fix it. Then ask again.

The first build was the draft. The critique is the design.

## Process

1. Open the file you built
2. Walk through each section: composition, craft, content, structure
3. Identify every default instead of decision
4. Rebuild those parts from the decision, not a patch
5. Don't narrate the critique. Do the work. Show the result.
