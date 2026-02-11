---
name: cursor-review
description: This skill should be used when the user asks for a "cursor review", "external code review", "second opinion on code", "GPT review", "cross-model review", "review my changes with cursor", or wants an independent AI model (GPT-5.3 Codex High via Cursor CLI) to validate code correctness, security, and implementation quality.
allowed-tools: Bash, Read, Grep, Glob
---

# Cursor Code Review

Delegate code review to GPT-5.3 Codex High via Cursor CLI headless mode for an independent second opinion from a different AI model.

## Scope Selection

Determine review scope from `$ARGUMENTS`:

- **`diff`** (default when no argument provided): Review only git changes (staged + unstaged + untracked)
- **`full`**: Review entire codebase structure and key source files
- **A file path or glob**: Review only the specified files (e.g., `src/auth/*.ts`)

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

- **Under 8,000 lines**: Pass everything in a single Cursor invocation
- **8,000–20,000 lines**: Split into logical file groups by top-level directory (e.g., `src/`, `tests/`, `config/`). Run one Cursor invocation per group. Consolidate all results.
- **Over 20,000 lines**: Review only the top 50 most-changed files (by diff line count). Warn the user that coverage is partial and suggest running targeted reviews on specific directories.

### Phase 5: Invoke Cursor CLI

Execute the review using the Cursor CLI headless agent. The command runs directly on cmd:

```
agent -p "THE_ASSEMBLED_PROMPT"
```

**Important execution details:**

- Set a Bash timeout of 300000ms (5 minutes) to prevent hangs
- If the assembled prompt exceeds 6,000 characters, write it to a temporary file first:
  1. Write prompt to a temp file (e.g., `%TEMP%\cursor-review-prompt.txt` on Windows or `/tmp/cursor-review-prompt.txt`)
  2. Invoke: `agent -p "$(cat /tmp/cursor-review-prompt.txt)"` or equivalent
- Cursor reads files from the current working directory automatically when file paths are mentioned in the prompt
- The Cursor agent may read additional files in the codebase for context — this is expected and read-only

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

- **Cursor CLI not found**: "The `agent` command was not found. Ensure Cursor CLI is installed and available in your PATH. Try running `agent --help` to verify."
- **Timeout (>5 min)**: Kill the process, report any partial output received, suggest reducing scope with `diff` or specific file paths
- **Authentication error**: "Cursor CLI requires authentication. Run `cursor` interactively first to log in, or set `CURSOR_API_KEY` environment variable."
- **Empty output**: "Cursor returned no output. The workspace directory may need to be trusted first. Try running `agent -p 'hello'` interactively once."
- **Rate limit / quota error**: Report the error and suggest retrying later

## Important Notes

- This skill is **read-only** — do NOT modify any files during the review process
- The Cursor agent runs in the project's working directory and may read files for context
- If the user provides additional focus areas or concerns in their message, incorporate those into the review prompt as priority items
- For `full` scope, focus the prompt on architecture and design patterns rather than line-by-line analysis
- Consult `references/review-categories.md` for extended antipatterns when interpreting and enriching the results

## Additional Resources

### Reference Files

For detailed review guidance, consult these files in this skill's directory:
- **`references/review-prompt-template.md`** — The structured prompt template sent to GPT-5.3 with all category definitions and output format requirements
- **`references/review-categories.md`** — Extended examples, antipatterns, and red flags per category for interpreting and enriching Cursor's output
