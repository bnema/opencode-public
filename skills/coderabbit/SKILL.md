---
name: coderabbit
description: Use when running CodeRabbit CLI code reviews, reviewing uncommitted or committed changes, comparing branches, or authenticating with CodeRabbit. Triggers on "code review", "cr review", "coderabbit", "review my changes", "review this branch".
---

# CodeRabbit CLI

AI code review from the terminal. The command is `cr` (alias for `coderabbit`).

## When to Use

- Review uncommitted or committed changes before push
- Compare current branch against a base branch
- Get AI feedback on bugs, security, and code quality

## Setup

```bash
cr auth login                    # OAuth login (opens browser)
cr auth login --api-key <key>    # API key auth (non-interactive)
cr auth status                   # Check auth state
cr auth org                      # Switch organization
cr auth logout                   # Log out
```

## Review

**Reviews take 2-10+ minutes. Set tool timeout to at least 600000ms (10 min).**

```bash
# All changes (committed + uncommitted) — the default
cr review

# Uncommitted changes only (staged + unstaged)
cr review -t uncommitted

# Committed changes only
cr review -t committed

# Compare against a base branch
cr review --base main

# Compare against a specific commit
cr review --base-commit abc123

# Feed project instructions to the reviewer
cr review -c .claude/CLAUDE.md -c .coderabbit.yaml
```

## Agent Usage

When calling `cr review` from a tool or subagent:

1. **Set timeout to 600000ms or higher** — reviews are slow
2. Use `--agent` for structured JSON findings
3. Pass `--no-color` to strip ANSI escape codes
4. Default output (`--plain`) is human-readable text

```bash
cr review --agent --no-color -t uncommitted
```

## Quick Reference

| Task | Command |
|------|---------|
| Review everything | `cr review` |
| Uncommitted only | `cr review -t uncommitted` |
| Committed only | `cr review -t committed` |
| Against main | `cr review --base main` |
| Against a commit | `cr review --base-commit <sha>` |
| With context files | `cr review -c file1 -c file2` |
| Agent output | `cr review --agent` |
| Interactive mode | `cr review --interactive` |
| Check auth | `cr auth status` |
| Login | `cr auth login` |
| Switch org | `cr auth org` |
| Update CLI | `cr update` |

## Common Mistakes

**Timeout too short.** Default 2-minute timeouts kill the review mid-flight. Always set 10+ minutes (600000ms).

**Wrong review type.** `cr review` includes both committed and uncommitted changes. Use `-t uncommitted` to review only working-tree changes.

**Missing `--no-color`.** ANSI codes clutter agent output. Always pass `--no-color` when parsing programmatically.

**No context files.** Without `-c`, the reviewer lacks project conventions. Pass your CLAUDE.md or coderabbit.yaml for better results.
