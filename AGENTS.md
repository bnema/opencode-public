# Agent Delegation Policy

The main agent coordinates the work. Token usage is a hard constraint. Use the cheapest adequate path.

## Rules

1. **Do small direct work in the main context when that is cheaper.** Do not delegate simple reads, small edits, or routine verification just because a subagent exists.

2. **Default to `grunt-worker` for straightforward implementation work when it requires essentially no thinking.** Use it when the task is clear, bounded, fully specified, and mostly execution: repetitive edits, boilerplate, renames, well-specified features, or long step-by-step changes with little to no inference. The parent agent must provide concrete instructions because `grunt-worker` does not infer intent or make decisions.

3. **Use `senior-engineer` when the task needs engineering judgment.** Reach for it when the work involves non-obvious tradeoffs, debugging, logic verification, ambiguous requirements, contained fixes that require reasoning, or pragmatic follow-up after implementation.

4. **Use `deep-thinker` sparingly.** Only use it for root-cause investigations that remain unclear after a bounded first pass, architecture or refactor decisions, technology choices, or cases where multiple viable approaches must be compared.

5. **Use `researcher` for bounded evidence gathering.** Use it to inspect a repo, clone a reference repo into `/tmp`, scan files, collect facts, and return concise evidence without broad synthesis.

6. **Prefer one agent over cascades.** Nested subagent calls are allowed, but use them only when they reduce total token use or materially improve quality.

7. **The main agent decides escalation.** Do not let a cheaper agent decide whether a task needs deeper reasoning unless the parent prompt asks for that explicitly.

8. **Typed reviewers are opt-in, not default.** Use `go-reviewer`, `rust-reviewer`, or `typescript-reviewer` only when the change is substantial enough that language-specific risk justifies the extra pass.

9. **Delegate design work to the creative designer.** UI/UX direction, interface layout, visual identity, and component design go to `creative-designer`. Do not force UI decisions into the main agent; pass the overall goal and product or technical constraints.

10. Prepend your git commands with `GIT_EDITOR=true`.

11. **Do not** store specs, plans, issues in the repo but in $OBSIDIAN_PROJECTS_PATH/{projectName}/...

| Agent | Use for |
|---|---|
| `grunt-worker` | Default for straightforward implementation only when the work is fully specified and requires little to no reasoning: bulk edits, renames, reformatting, boilerplate, repetitive file operations, and clear low-judgment coding tasks |
| `researcher` | Cheap, bounded evidence gathering: inspect repos, scan files, clone references into `/tmp`, and return concise facts |
| `senior-engineer` | Seasoned engineering judgment: ambiguous or high-judgment implementation work, review, logic verification, contained fixes, and pragmatic follow-up |
| `deep-thinker` | High-cost deep reasoning for unclear root-cause work, architecture decisions, refactoring plans, and technology choices |
| `creative-designer` | UI/UX design thinking, interface design, and visual direction |
| `go-reviewer` | Go code review: idiomatic patterns, concurrency safety, error handling, security |
| `rust-reviewer` | Rust code review: ownership, lifetimes, unsafe usage, error handling, security |
| `typescript-reviewer` | TypeScript/JS code review: type safety, async correctness, Node.js security |
| `go-build-resolver` | Fix Go build errors, go vet issues, linter warnings — surgical fixes only |
| `rust-build-resolver` | Fix Rust cargo build errors, borrow checker issues, dependency problems — surgical fixes only |
