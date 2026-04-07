---
name: deepreview
description: Use when asked for a deep code review, thorough review, security audit, full review of changes, or pre-merge review. Triggers on "deep review", "thorough review", "audit my code", "review for security", "check for bugs", "review before merge".
---

# Deep Review

Multi-stage code review pipeline using parallel subagents. The entire review runs outside the main context. A gatherer agent collects context and runs static analysis, six reviewer agents execute parallel review passes, a verifier agent checks critical findings, then the gatherer formats the final report.

Languages: Go, Rust, TypeScript, Svelte, Astro.

## Pipeline

The main session is the **orchestrator**. It does NOT run any review stages itself — it dispatches all work to subagents and synthesizes results.

### Agent Roles

| Role | Count | Work | Model guidance |
|------|-------|------|----------------|
| gatherer | 1 | Stages 1+2 (context gathering, static analysis), Stage 5 (final formatting) | Fast/cheap model (e.g. Sonnet, GPT-4o-mini) |
| reviewer | 6 | Stage 3 (one review pass each) | Strong reasoning model (e.g. Opus, GPT-5, o3) |
| verifier | 1 | Stage 4 (verify CRITICAL/HIGH findings) | Strong reasoning model |

### Review Pass Assignments

| Pass | Lens |
|------|------|
| security | Attacker mindset |
| logic | Edge-case hunting |
| performance | Hot-path analysis |
| quality | Maintainability |
| api-contracts | Consumer impact |
| dependencies | Supply chain |

### Execution Flow

```
Step 1: Dispatch gatherer with Stage 1+2 instructions and the diff scope.
        Wait for result (diff, changed files, static analysis, architecture context).

Step 2: Dispatch all 6 reviewers IN PARALLEL.
        Each gets: the gatherer's full output plus their specific review pass instructions.
        Wait for all 6 to complete.

Step 3: Dispatch verifier with all CRITICAL and HIGH findings from the 6 reviewers.
        Wait for result (confirmed/dropped findings with evidence).

Step 4: Dispatch gatherer (or a new formatting subagent) with:
        - All reviewer findings
        - Verifier results (confirmed/dropped)
        - Instructions for Stage 5 (dedup, sort, format, summary)
        Wait for result. Present to user.
```

### Harness-Specific Hints

<details>
<summary>Pi</summary>

Use `subagent` single mode for steps 1, 3, and 4:
```
subagent({ agent: "worker", task: "<gatherer instructions>" })
```

For step 2, use `subagent` parallel mode with 6 tasks:
```
subagent({ tasks: [
  { agent: "worker", task: "<security pass instructions + gatherer output>" },
  { agent: "worker", task: "<logic pass instructions + gatherer output>" },
  ...
]})
```
</details>

<details>
<summary>OpenCode</summary>

Use the `task` tool for each subagent. For step 2, issue all 6 `task` tool calls in a single assistant message to run them concurrently.
</details>

<details>
<summary>Claude Code</summary>

Use the `subagent` tool (single/parallel modes) following the same pattern as Pi. If team agents are available, apply the same step structure.
</details>

---

## Stage 1: Context Gathering (Gatherer)

1. **Determine diff scope:**
   - User specifies base branch: `git diff --name-status <base>..HEAD`
   - User specifies committed/uncommitted: `git diff --name-status --cached` or `git diff --name-status`
   - Default: `git diff --name-status HEAD~N..HEAD` where N = unpushed commits (`git rev-list --count @{upstream}..HEAD`). Fall back to `HEAD~1` if no upstream.

2. **Empty diff?** Report "No changes to review" and stop.

3. **Read full content** of each changed file. Do not review only diff hunks.

4. **Trace imports one level deep** (direct dependencies only, skip external/third-party):
   - Go: parse `import` blocks, read local packages
   - Rust: parse `use`/`mod`, read local modules
   - TypeScript: parse `import`/`require`, read local files

5. **Git history:**
   - `git log --oneline -10 -- <changed-files>`
   - `git log --oneline -5` (recent repo context)

6. **Project conventions** (read if present):
   - `CLAUDE.md`, `AGENTS.md`
   - `.golangci.yml`, `eslint.config.*`, `clippy.toml`

## Stage 2: Static Analysis (Gatherer)

Auto-detect languages from changed file extensions. For each tool: check if installed (`which <tool>`), run if present, skip with warning if missing.

| Language | Tool | Command | Scoping |
|----------|------|---------|---------|
| Go | golangci-lint | `golangci-lint run --out-format json <files>` | Pass changed file paths |
| Rust | clippy | `cargo clippy --message-format json 2>&1` | Full crate, filter output to changed files |
| Rust | cargo-audit | `cargo audit --json` | Full crate |
| TS/Svelte/Astro | eslint | `npx eslint --format json <files>` | Pass changed file paths |
| All | gitleaks | `gitleaks detect --no-git -v --source <dir>` | Changed files directory |

If a tool is missing, warn and continue:

```
Warning: golangci-lint not found, skipping Go static analysis
Install: go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
```

Collect all findings. Include them in the output passed to Stage 3 reviewers.

## Stage 3: AI Multi-Pass Review (Reviewers)

### Architecture Detection (Gatherer, included in output for reviewers)

Detect project architecture to include in reviewer context:
- Cross-reference the `hexarch` skill if available
- Otherwise check directory signals: `ports/`+`adapters/`+`domain/` (hexagonal), `domain/`+`application/`+`infrastructure/` (clean), `internal/`+`cmd/` (Go standard)

### Six Review Passes

Each reviewer receives: diff, changed file list, static analysis findings, architecture context, and their specific pass instructions below.

**Each pass assigns its own severity** (CRITICAL, HIGH, MEDIUM, LOW) based on impact and exploitability. Severity follows impact, not category.

If a reviewer finds something relevant to another pass, include it in their output so the verifier and formatter can cross-reference.

Cross-reference with static analysis output. If a tool already caught an issue, reference it instead of duplicating.

**Pass 1 — Security (attacker mindset):**
Injection (SQL, command, template), auth bypass, secret exposure, SSRF, path traversal, insecure deserialization, XSS, CSRF. Architecture: domain importing HTTP/DB packages expands the attack surface.

**Pass 2 — Logic bugs (edge-case hunting):**
Nil/null dereference, off-by-one, race conditions, unchecked errors, unreachable code, wrong operators, integer overflow. Architecture: misplaced business logic means missed invariants.

**Pass 3 — Performance (hot-path analysis):**
Unnecessary allocations, N+1 queries, missing indexes, unbounded loops, blocking calls in async, excessive copying. Architecture: wrong-layer DB calls hide N+1 patterns.

**Pass 4 — Code quality (maintainability):**
Duplication, cyclomatic complexity, dead code, unclear naming, god functions. Primary home for architecture layer violations.

**Pass 5 — API contracts (consumer impact):**
Breaking changes, missing boundary validation, type mismatches, undocumented behavior changes, port hygiene violations.

**Pass 6 — Dependencies (supply chain):**
Known CVEs, outdated packages, unnecessary deps, license issues. Cross-reference cargo-audit and gitleaks findings.

## Stage 4: Verification (Verifier)

After all reviewers complete, the verifier checks every CRITICAL and HIGH finding. Skip verification for MEDIUM and LOW.

For each critical/high finding, generate 1-2 verification commands:
- `grep` / `rg` for pattern confirmation (error checks, nil guards, missing imports)
- `ast-grep --pattern` for structural verification (control flow, signatures, type usage)
- `git log` to check if the pattern was intentional

Rules:
- Max 2 commands per finding
- 5s timeout per command
- Verification contradicts the finding: drop it entirely. No hedging.
- Verification confirms: attach output as evidence

Example:

```
Finding: "Nil dereference on error path at handler.go:42"
Verify:  ast-grep --pattern '$X := $FUNC($$$); $$$; if $X == nil' --lang go handler.go
Result:  match at line 42. Confirmed — attach as evidence.
```

## Stage 5: Triage and Present (Gatherer)

The gatherer receives all reviewer findings and verifier results (confirmed/dropped), merges them, deduplicates, and sorts by severity.

### Severity Levels

| Level | Action |
|-------|--------|
| CRITICAL | Must fix before merge |
| HIGH | Should fix |
| MEDIUM | Consider fixing |
| LOW | Nice to have |

### Finding Format

```
CRITICAL SECURITY | handler.go:42 | Nil pointer dereference on error path
  ParseToken() error is checked but token is used before the nil check.

  Evidence:
  $ ast-grep --pattern '$X := $F($$$); $$$; if $X == nil' --lang go handler.go
  > handler.go:42: token := ParseToken(r); fmt.Println(token.UserID); if token == nil {

  Fix: Move nil check before first use of token.
```

Omit the Evidence section for MEDIUM and LOW findings.

### Summary

End every review with:

```
Found: N CRITICAL | N HIGH | N MEDIUM | N LOW
```

**Disclaimer (always include at the top of the report):**
```
These findings were generated by automated analysis. Verify each finding
against the current code before acting on it. Only fix what is confirmed.
```

If CRITICAL findings exist, offer to fix them.

## Tool Requirements

All tools assumed pre-installed. Missing tools get a warning; the review continues without them.

| Tool | Install |
|------|---------|
| golangci-lint | `go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest` |
| clippy | Ships with `rustup` |
| cargo-audit | `cargo install cargo-audit` |
| eslint | Project-local via `npx` |
| ast-grep | `cargo install ast-grep` or `npm i -g @ast-grep/cli` |
| gitleaks | `brew install gitleaks` or download binary |
