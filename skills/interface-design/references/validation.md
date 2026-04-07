# Memory Management

When and how to update `.interface-design/system.md`.

## When to Add Patterns

Add when: component used 2+ times, reusable across project, has specific measurements worth remembering.

## Pattern Format

```markdown
### Button Primary
- Height: 36px
- Padding: 12px 16px
- Radius: 6px
- Font: 14px, 500 weight
```

## Don't Document

One-off components, temporary experiments, variations better handled with props.

## Pattern Reuse

Before creating a component, check system.md. Pattern exists? Use it. Need variation? Extend, don't create new.

Memory compounds: each pattern saved makes future work faster and more consistent.

---

# Validation Checks

If system.md defines values, check consistency:

- **Spacing** -- all values multiples of defined base?
- **Depth** -- declared strategy used throughout? (borders-only = no shadows)
- **Colors** -- defined palette used, not random hex?
- **Patterns** -- documented patterns reused, not reinvented?
