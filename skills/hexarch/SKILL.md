---
name: hexarch
description: Use when scanning a repo for hexagonal architecture violations, reviewing code placement across layers, auditing dependency direction, or checking port hygiene. Triggers on "hex arch", "ports and adapters", "clean architecture layers", "domain imports infrastructure", "business logic in adapter", "wrong layer", "missing port".
---

# Hexagonal Architecture Audit

Scan a repository for hexagonal architecture violations. Language-agnostic. Detects wrong-direction dependencies, misplaced business logic, missing ports, and leaky abstractions. Auto-fixes mechanical violations, reports structural ones.

## Phase 1: Layer Discovery

Identify the project's hexagonal layers before scanning.

1. **Read project docs first.** Check `CLAUDE.md`, `AGENTS.md`, and architecture docs in the repo for explicit layer definitions. These are authoritative — use them over auto-detection.

2. **Auto-detect if no docs.** Scan directory names for known patterns:

| Layer | Common directory names |
|-------|----------------------|
| Domain/core | `domain`, `core`, `model`, `models`, `entities` |
| Ports | `ports`, `interfaces`, `boundaries` |
| Adapters/infra | `adapters`, `infrastructure`, `infra`, `driven`, `driving` |
| Application | `application`, `usecase`, `usecases`, `use_cases`, `services` |
| Entry points | `cmd`, `entrypoints`, `api`, `cli`, `ui`, `web` |

3. **Build the layer map** — a mapping of each directory to its hex arch layer. This map is passed to all three agents.

4. **Detect mode.** Run `git diff` and `git diff --cached`. If changes exist: **diff mode** (scan full structure for context, filter output to changed files). If clean tree: **audit mode** (report all violations).

## Phase 2: Launch Three Agents in Parallel

Use the Agent tool to launch all three agents concurrently in a single message. Pass each agent the layer map and the full diff (if any).

### Agent 1: Dependency Direction

Scan all import/require/use statements across the codebase. Detect imports by language:
- Go: `import` blocks
- TypeScript/JavaScript: `import`/`require`
- Rust: `use`/`mod`
- Python: `import`/`from ... import`
- Other: grep for common import patterns

Flag these violations:

**Domain imports infrastructure** (Critical) — Domain/core packages importing DB drivers, HTTP libs, cloud SDKs, or framework packages. The domain layer must have zero outward dependencies.
- Auto-fix: if a port interface already exists in the ports layer whose methods match the usage, replace the infrastructure import with the port. Search ports for matching interfaces before reporting.
- Report: when no matching port exists (one needs to be created).

**Wrong-direction dependencies** (Critical) — Any inner layer importing an outer layer. The dependency rule is: adapters -> ports -> application -> domain. Never the reverse.
- Auto-fix: when an existing port can replace the direct import.
- Report: otherwise.

**Adapter-to-adapter coupling** (High) — Adapters importing other adapters instead of communicating through ports.
- Report only.

**Leaking framework types** (High) — HTTP request/response objects, ORM models, CLI framework types, or cloud SDK types appearing in domain or application layer function signatures, struct fields, or return types.
- Report only.

### Agent 2: Responsibility Placement

Read function and method bodies in each layer. Look for code that belongs elsewhere.

**Business logic in adapters** (Critical) — Validation rules, domain calculations, state machines, conditional business decisions, or transformation logic living in adapter code. Adapters should only translate between external interfaces and ports — no decisions.
- Report only (requires design decision to fix).

**Application layer doing infrastructure work** (High) — Use cases or application services making direct DB calls, HTTP requests, file system I/O, or network operations instead of going through ports.
- Report only.

**God domain package** (Medium) — A single domain/core package containing many unrelated types with no sub-boundaries. Look for packages with 10+ types spanning unrelated business concepts.
- Report only.

### Agent 3: Port Hygiene

Scan interface definitions and their usage across the codebase.

**Ports with concrete types** (High) — Port interfaces whose method signatures accept or return infrastructure types (DB models, HTTP types, ORM entities, file handles) instead of domain types.
- Report only.

**Missing ports** (Medium) — Adapters used directly by application or domain code without a port interface in between, making them non-swappable.
- Auto-fix: when a port interface exists but application code references the concrete adapter type instead — replace with the port interface.
- Report: when no port interface exists yet.

## Phase 3: Aggregate and Fix

Wait for all three agents to complete. Aggregate findings. Apply auto-fixes for mechanical violations. Skip false positives without arguing — just note and move on.

### Output Format

Group by severity, then by file:

```
## Critical
- `path/to/file:line` — Violation name: description. Why it violates hex arch. What to do instead.

## High
- `path/to/file:line` — Violation name: description.

## Medium
- `path/to/file:line` — Violation name: description.

## Auto-fixed
- `path/to/file:line` — What was changed and why.
```

**Diff mode** header: `Scanning N changed files (full project structure used for context)`

**Audit mode** header: `Full repository audit — N files scanned across M layers`

End with: `Found X critical, Y high, Z medium violations. Auto-fixed N.`
