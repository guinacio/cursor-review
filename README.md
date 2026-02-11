# cursor-review

A Claude Code plugin marketplace that provides **cross-model code review** using [Cursor CLI](https://cursor.com/docs/cli/headless) and GPT-5.3 Codex High.

Get an independent second opinion on your code from a different AI model, checking correctness, bugs, security, tests, performance, error handling, architecture, and plan compliance.

## Prerequisites

- [Claude Code](https://claude.com/claude-code) v1.0.33+
- [Cursor](https://cursor.com) installed with CLI access (`agent` command available in your PATH)

## Installation

### From GitHub Marketplace

```shell
/plugin marketplace add guinacio/cursor-review
/plugin install cursor-review@cursor-review-marketplace
```

### Local Development

```shell
claude --plugin-dir ./plugins/cursor-review
```

## Usage

### Basic diff review (default)

```shell
/cursor-review:cursor-review
```

Reviews git changes (staged + unstaged) against 8 quality categories.

### Full codebase review

```shell
/cursor-review:cursor-review full
```

Reviews the entire project structure and key source files.

### Review specific files

```shell
/cursor-review:cursor-review src/auth/login.ts
```

### Auto-trigger

Claude can also invoke this skill automatically when it detects situations that would benefit from a second opinion (large PRs, security-sensitive changes, complex refactors).

## What Gets Reviewed

| Category | What It Checks |
|----------|---------------|
| **Code Correctness** | Logic errors, type errors, race conditions, off-by-one |
| **Bugs** | Null refs, resource leaks, edge cases, state mutation |
| **Security** | OWASP Top 10 — injection, XSS, auth, SSRF, secrets |
| **Tests** | Coverage gaps, brittle tests, weak assertions |
| **Performance** | N+1 queries, memory leaks, algorithmic complexity |
| **Error Handling** | Swallowed errors, missing validation, poor messages |
| **Architecture** | SOLID violations, coupling, circular dependencies |
| **Plan Compliance** | Auto-detects plans and checks implementation alignment |

## Output Format

The review produces a structured report with:

- **Summary**: scope, files reviewed, issue counts by severity
- **Issues by severity**: CRITICAL > HIGH > MEDIUM > LOW, each with file:line, description, and suggested fix
- **Plan compliance** (when a plan is detected): alignment assessment, gaps, deviations
- **Positive observations**: what the code does well
- **Overall assessment**: quality score (1-10), highest risk area, recommendation

## Plan Auto-Detection

The skill automatically searches for implementation plans in:

- `.claude/plans/*.md`
- `PLAN.md` or `TODO.md` in the project root
- GitHub PR descriptions (via `gh pr view`)

If a plan is found, the review includes a plan compliance section comparing the implementation against intended requirements.

## License

MIT
