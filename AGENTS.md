# Agent Delegation Policy

The main agent coordinates the work. Token usage is a hard constraint. Use the cheapest adequate path.

## Core Principle

`grunt-worker` is the default implementer.

If code needs to be changed and the work is clear enough to describe concretely, delegate implementation to `grunt-worker`. Do not keep implementation in the main agent just because you can do it yourself. Do not jump to `senior-engineer` just because it feels safer or higher quality.

Treat delegation as the default for implementation and the main agent as the coordinator, not the hands-on coder.

## Delegation Order

1. Do the smallest direct work in the main context only when delegation would be slower than doing it yourself.
2. For implementation work, try `grunt-worker` first.
3. Escalate to `senior-engineer` only when the task genuinely requires engineering judgment.
4. Escalate to `deep-thinker` only when a bounded first pass still leaves the root cause or decision unclear.

If you are about to implement in the main agent or send implementation straight to `senior-engineer`, stop and ask: "Why is `grunt-worker` not sufficient?" If the answer is not concrete, use `grunt-worker`.

## Rules

0. **Honor `#nopowers` as a direct user override.** If the user's prompt includes `#nopowers`, skip the superpowers skill-selection flow and do not invoke any skill from the superpowers skill set unless the user explicitly requests one. Treat `#nopowers` as applying to delegated subagents as well.

1. **Do small direct work in the main context when that is cheaper.** Do not delegate simple reads, tiny edits, or routine verification just because a subagent exists.

2. **`grunt-worker` is the main implementation path.** Use it by default for code changes when the task is clear, bounded, and mostly execution. This includes repetitive edits, boilerplate, renames, wiring work, test updates, well-specified bug fixes, well-specified features, and any task you can express as concrete steps. The parent agent must provide exact instructions because `grunt-worker` does not infer intent or make decisions.

3. **Do not hoard implementation in the main agent.** If the task is implementation-heavy and does not need active judgment at each step, delegate it. "I can do this quickly myself" is not a reason to skip `grunt-worker` unless the edit is genuinely tiny.

4. **`senior-engineer` is not the default implementer.** Use `senior-engineer` for debugging, ambiguous requirements, non-obvious tradeoffs, code review, logic verification, or contained fixes that require reasoning. Do not use `senior-engineer` for routine implementation that `grunt-worker` can execute.

5. **When judgment produces a concrete plan, hand implementation back down.** If `senior-engineer` clarifies the approach, use `grunt-worker` to carry out the resulting edits unless the remaining work still needs sustained judgment.

6. **Use `deep-thinker` sparingly.** Only use it for root-cause investigations that remain unclear after a bounded first pass, architecture or refactor decisions, technology choices, or cases where multiple viable approaches must be compared.

7. **Use `researcher` for bounded evidence gathering.** Use it to inspect a repo, clone a reference repo into `/tmp`, scan files, collect facts, and return concise evidence without broad synthesis.

8. **Prefer one agent over cascades.** Assume subagents do not dispatch further subagents unless their frontmatter explicitly grants `task` permission. In this repo, treat nested delegation as the exception, not the default.

9. **The main agent decides escalation.** Do not let a cheaper agent decide whether a task needs deeper reasoning unless the parent prompt asks for that explicitly.

10. **Typed reviewers are opt-in, not default.** Use `go-reviewer`, `rust-reviewer`, or `typescript-reviewer` only when the change is substantial enough that language-specific risk justifies the extra pass.

11. **Delegate design work to the creative designer.** UI/UX direction, interface layout, visual identity, and component design go to `creative-designer`. Do not force UI decisions into the main agent; pass the overall goal and product or technical constraints.

12. Prepend your git commands with `GIT_EDITOR=true`.

13. **Do not** store specs, plans, issues in the repo but in `$OBSIDIAN_PROJECTS_PATH/{projectName}/...`.

## Decision Checklist

Use `grunt-worker` when the answer to these questions is yes:

1. Is the task primarily implementation?
2. Is the task clear enough to describe in concrete steps?
3. Can the worker execute it without making important product or architecture decisions?

If all three are yes, delegate to `grunt-worker`.

Use `senior-engineer` only when at least one of these is true:

1. The right fix is unclear.
2. The task needs debugging or root-cause analysis.
3. The work depends on tradeoffs, design judgment, or logic review.
4. You need an expert review of a change, not routine execution.

## Examples

Use `grunt-worker`:

- Rename a function across the repo.
- Apply a well-specified refactor.
- Add a clearly described endpoint or UI control.
- Update tests to match an already chosen implementation.
- Perform repetitive edits across many files.

Use `senior-engineer` first:

- Investigate why a test is flaky.
- Decide between two implementation approaches.
- Review a risky change for regressions.
- Diagnose a bug with unclear root cause.

Use `senior-engineer`, then `grunt-worker`:

- Ask `senior-engineer` to identify the right fix for a race condition, then ask `grunt-worker` to apply the chosen patch.
- Ask `senior-engineer` to review an ambiguous feature request and turn it into an implementation plan, then ask `grunt-worker` to execute that plan.

## Anti-Patterns

Do not do these:

- Defaulting to the main agent for multi-file implementation.
- Defaulting to `senior-engineer` for routine code changes.
- Treating "higher quality" as sufficient reason to skip `grunt-worker`.
- Keeping implementation in the coordinating agent after the task becomes clear.
- Escalating because of uncertainty you could remove with one bounded research or judgment pass.

| Agent | Use for |
|---|---|
| `grunt-worker` | Default implementation agent. Use for clear, bounded execution work: bulk edits, renames, reformatting, boilerplate, repetitive file operations, well-specified bug fixes, well-specified features, and any task that can be handed off as concrete steps. |
| `researcher` | Cheap, bounded evidence gathering: inspect repos, scan files, clone references into `/tmp`, and return concise facts. |
| `senior-engineer` | Seasoned engineering judgment: ambiguous or high-judgment work, debugging, review, logic verification, tradeoff decisions, and clarifying the right approach before handoff. |
| `deep-thinker` | High-cost deep reasoning for unclear root-cause work, architecture decisions, refactoring plans, and technology choices. |
| `creative-designer` | UI/UX design thinking, interface design, and visual direction. |
| `go-reviewer` | Go code review: idiomatic patterns, concurrency safety, error handling, security. |
| `rust-reviewer` | Rust code review: ownership, lifetimes, unsafe usage, error handling, security. |
| `typescript-reviewer` | TypeScript/JS code review: type safety, async correctness, Node.js security. |
| `go-build-resolver` | Fix Go build errors, go vet issues, linter warnings; surgical fixes only. |
| `rust-build-resolver` | Fix Rust cargo build errors, borrow checker issues, dependency problems; surgical fixes only. |
