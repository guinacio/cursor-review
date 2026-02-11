# cursor-review plugin

External AI code review via Cursor CLI (GPT-5.3 Codex High). Provides a second opinion checking correctness, bugs, security, tests, performance, error handling, architecture, and plan compliance.

## How It Works

1. Claude gathers context (git diff, project type, plan files)
2. Claude builds a comprehensive review prompt from a template
3. The prompt is sent to GPT-5.3 Codex High via `agent -p "prompt"` (Cursor CLI headless mode)
4. GPT-5.3 performs an independent code review
5. Claude parses and presents the results in a structured format

## Skill: `/cursor-review:cursor-review`

### Arguments

| Argument | Description |
|----------|-------------|
| *(none)* | Reviews git diff (default) |
| `diff` | Explicitly review git changes only |
| `full` | Review entire codebase |
| `<file-path>` | Review specific file(s) |

### Review Categories

1. **Code Correctness** — logic errors, type errors, race conditions
2. **Bugs** — null references, resource leaks, edge cases
3. **Security** — OWASP Top 10 (injection, XSS, auth, SSRF, secrets)
4. **Tests** — coverage gaps, brittle tests, weak assertions
5. **Performance** — N+1 queries, memory leaks, algorithmic complexity
6. **Error Handling** — swallowed errors, missing validation
7. **Architecture** — SOLID violations, coupling, circular dependencies
8. **Plan Compliance** — auto-detects plans and checks alignment

### Severity Levels

- **CRITICAL**: Will cause failures, data loss, or security breaches
- **HIGH**: Likely to cause bugs or significant issues
- **MEDIUM**: Should be fixed but not blocking
- **LOW**: Suggestions for improvement
- **INFO**: Observations and positive notes

## Prerequisites

- Cursor CLI installed with `agent` command available in PATH
- Cursor authenticated (run `cursor` interactively once if needed)

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `agent` command not found | Install Cursor and ensure CLI is in PATH |
| Authentication error | Run `cursor` interactively to log in |
| Timeout | Reduce scope — use `diff` instead of `full` |
| Empty output | Trust the workspace by running `agent -p "hello"` once |
