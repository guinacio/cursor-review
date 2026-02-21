---
name: cursor-review
description: This skill should be used when the user asks for a "cursor review", "external code review", "second opinion on code", "GPT review", "cross-model review", "review my changes with cursor", or wants an independent AI model (GPT-5.3 Codex High via Cursor CLI) to validate code correctness, security, and implementation quality.
allowed-tools: Bash, Read, Write, Grep, Glob
---

# Cursor Code Review

Delegate code review to GPT-5.3 Codex High via Cursor CLI headless mode for an independent second opinion from a different AI model.

## Scope Selection

Determine review scope from `$ARGUMENTS`:

- **`diff`** (default when no argument provided): Review only git changes (staged + unstaged + untracked)
- **`full`**: Review entire codebase structure and key source files
- **A file path or glob**: Review only the specified files (e.g., `src/auth/*.ts`)

> **Known limitation — diff scope:** GPT-5.3 Codex High occasionally misreads quoted strings in unified diff format, producing false positives (e.g., flagging `"value"` or `["a", "b"]` as missing quotes). For small changes, prefer specific file scope (e.g., `/cursor-review agent_core.py`) — the agent reads the actual file and avoids this diff-parsing confusion. Diff scope is best for larger, multi-file changes where the structural context outweighs the risk of false positives.

## Workflow

### Phase 1: Gather Context

Collect project context by running these commands via Bash:

**For `diff` scope:**
```
git diff HEAD --stat
git diff HEAD
git diff --cached
git status --short
```

**For `full` scope:**
```
git log --oneline -20
```
Then use Glob to find source files (limit to 100 most relevant files).

**For specific file scope:**
Read the specified files directly with the Read tool.

**Always gather:**
- Project type: check for `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`, `*.csproj`, `Gemfile`, or similar
- Branch name: `git branch --show-current`
- Recent commits: `git log --oneline -5`

### Phase 2: Plan Auto-Detection

Search for implementation plans to compare against:

1. Use Glob to check `.claude/plans/` for any `.md` files — read the most recent one (limit to first 200 lines)
2. Check for `PLAN.md` or `TODO.md` in the project root
3. If on a feature branch, try: `gh pr view --json body,title 2>/dev/null`
4. Check for `.cursor/rules` or `AGENTS.md` files

If a plan is found, include it in the review prompt for plan compliance checking. If no plan exists, skip the plan compliance category.

### Phase 3: Build Review Prompt

Read the prompt template from `references/review-prompt-template.md` in this skill's directory.

Construct the final prompt by replacing template placeholders:
- `{{PROJECT_TYPE}}` — detected project type (e.g., "Node.js/TypeScript", "Python/Django")
- `{{BRANCH_NAME}}` — current git branch
- `{{SCOPE}}` — the review scope being used
- `{{PLAN_CONTENT}}` — plan text if found, or remove the plan compliance section entirely
- `{{CODE_CONTENT}}` — the git diff output, file contents, or file listing depending on scope

### Phase 4: Handle Diff Size

Count total lines of the code content to review. Apply this strategy:

- **Under 2,000 lines**: Embed the code directly in the prompt and pass via temp file (see Phase 5).
- **2,000–20,000 lines**: Use the **delegation strategy** — do NOT embed code in the prompt. Instead, write a concise prompt (~1–2K chars) that tells the Cursor agent which files to read from the working directory. The Cursor agent has full read access to the codebase and will gather context itself. List specific file paths or use instructions like "read all `.py` files in `src/`".
- **Over 20,000 lines**: Same delegation strategy, but scope to the top 50 most-changed files (by diff line count). Warn the user that coverage is partial and suggest running targeted reviews on specific directories.

> **Why delegation?** Windows has a ~32K character command-line limit. Even writing to a temp file and reading it into a PowerShell variable, prompts over ~30K characters will fail with "The filename or extension is too long". Keeping prompts short and letting the Cursor agent read files itself is the only reliable approach for large reviews.

### Phase 5: Invoke Cursor CLI

The `agent` command (Cursor CLI headless mode) is a Node.js-based CLI available via PowerShell and CMD, but **NOT available in bash/Git Bash**. You **must** use PowerShell to invoke it.

#### Platform Note (Windows)

- `agent` is at `~\AppData\Local\cursor-agent\agent.ps1` and is auto-added to PowerShell/CMD PATH by Cursor
- Do NOT use `cursor.cmd agent -p` — the wrapper doesn't forward flags properly to the `agent` subcommand
- Do NOT try to call `agent` directly from Bash — it will return "command not found"

#### Invocation Method

**Always use PowerShell with the `-File -` pattern and a bash heredoc to avoid variable interpolation issues:**

1. Write the assembled review prompt to a temp file first:
   ```bash
   # Write prompt to temp file (use the Write tool, not echo)
   # e.g., /tmp/cursor-review-prompt.txt or %TEMP%\cursor-review-prompt.txt
   ```

2. Invoke via PowerShell from Bash:
   ```bash
   powershell -NoProfile -File - <<'PSEOF'
   $prompt = Get-Content 'C:\path\to\cursor-review-prompt.txt' -Raw
   & agent -f -p $prompt
   PSEOF
   ```

> **The `-f` flag** (force trust) is required for non-interactive invocations. Without it, Cursor will prompt for workspace trust confirmation and hang indefinitely in headless mode.

**Important execution details:**

- Set a Bash timeout of **300000ms** (5 minutes) to prevent hangs
- Always use the `<<'PSEOF'` heredoc (single-quoted delimiter) to prevent bash from interpolating `$prompt` as a shell variable
- Do NOT use `powershell -Command "..."` with `$prompt` inside — bash will interpret `$prompt` as empty
- The Cursor agent runs in the **current working directory** and can read any file there — leverage this for the delegation strategy
- The Cursor agent may read additional files beyond what you specify — this is expected and read-only

#### Delegation Prompt Example (for large codebases)

When the codebase exceeds ~2K lines, use a short prompt like this instead of embedding all code:

```
You are an expert code reviewer. Review the Python project in the current working directory.

Key files to read: main.py, agent_core.py, config.py, heartbeat.py, tray_icon.py, chat_gui.py, mcp_config.py, mcp_servers/browser_guard/server.py

Focus on: code correctness, bugs, security (OWASP Top 10), error handling, architecture.

For each issue: state category, severity (CRITICAL/HIGH/MEDIUM/LOW), file:line, description, and suggested fix.

End with: quality score (1-10), highest risk area, recommendation (APPROVE/APPROVE WITH COMMENTS/REQUEST CHANGES/BLOCK).
```

This keeps the prompt under 1K characters while still getting a thorough review.

### Phase 6: Present Results

After receiving the Cursor output:

1. Parse the review output into the 8 review categories
2. Assign severity levels: **CRITICAL** / **HIGH** / **MEDIUM** / **LOW** / **INFO**
3. Format the final report:

```markdown
## External Code Review (GPT-5.3 Codex High via Cursor)

### Summary
- Scope: [diff/full/specific files]
- Files reviewed: [count]
- Issues found: [total] ([critical] critical, [high] high, [medium] medium, [low] low)

### Critical Issues
1. **[Category]** `file:line` — Description
   **Fix:** Suggested remediation

### High Priority
[same format]

### Medium Priority
[same format]

### Low Priority / Suggestions
[same format]

### Plan Compliance
*(only if a plan was detected)*
- **Alignment:** [assessment of how well code matches the plan]
- **Gaps:** [planned features not yet implemented]
- **Deviations:** [implementation choices that differ from the plan]

### Positive Observations
- [what the code does well]

---
*Review performed by GPT-5.3 Codex High via Cursor CLI*
```

4. If the Cursor output is empty or malformed, present what was received and note the issue
5. If multiple Cursor invocations were used (chunking), consolidate and deduplicate findings

## Error Handling

Handle these failure modes with clear messages:

- **`agent: command not found` (Exit code 127)**: You're trying to call `agent` from bash. Switch to PowerShell invocation (see Phase 5). The `agent` command is only in PATH for PowerShell/CMD.
- **`The filename or extension is too long`**: The prompt exceeds Windows' ~32K character command-line limit. Switch to the delegation strategy — write a shorter prompt that tells the agent to read files itself (see Phase 4).
- **`'p' is not in the list of known options`**: You're using `cursor.cmd agent -p` — the wrapper doesn't forward flags. Use the standalone `agent` command via PowerShell instead.
- **`The term '=' is not recognized`**: You used `powershell -Command` with `$prompt` — bash interpreted `$prompt` as an empty shell variable. Use the `powershell -File -` heredoc pattern instead.
- **`Workspace Trust Required` (hangs indefinitely)**: You forgot the `-f` flag. The agent prompts for interactive trust confirmation which can't be answered in headless mode. Always use `agent -f -p`.
- **Timeout (>5 min)**: Kill the process, report any partial output received, suggest reducing scope with `diff` or specific file paths.
- **Authentication error**: "Cursor CLI requires authentication. Run `cursor` interactively first to log in."
- **Empty output**: "Cursor returned no output. Try running `agent -p 'hello'` in PowerShell interactively once to verify it works."
- **Rate limit / quota error**: Report the error and suggest retrying later.

## Important Notes

- This skill is **read-only** — do NOT modify any files during the review process
- The Cursor agent runs in the project's working directory and may read files for context
- **Windows-only**: This skill requires PowerShell and Cursor Desktop installed with the `agent` CLI available
- **Always use the delegation strategy for `full` scope** — never try to embed an entire codebase in the prompt
- If the user provides additional focus areas or concerns in their message, incorporate those into the review prompt as priority items
- For `full` scope, focus the prompt on architecture and design patterns rather than line-by-line analysis
- Consult `references/review-categories.md` for extended antipatterns when interpreting and enriching the results

## Additional Resources

### Reference Files

For detailed review guidance, consult these files in this skill's directory:
- **`references/review-prompt-template.md`** — The structured prompt template sent to GPT-5.3 with all category definitions and output format requirements
- **`references/review-categories.md`** — Extended examples, antipatterns, and red flags per category for interpreting and enriching Cursor's output
