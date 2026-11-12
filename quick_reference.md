# Claude Code Quick Reference

Current as of Claude Code 2.1.31x (November 2026). Full tables in Appendices A-C.

## CLI Commands

| Command | Description |
|---------|-------------|
| `claude` | Start interactive session |
| `claude "task"` | Start with an initial prompt |
| `claude -p "task"` | Non-interactive run; print result and exit |
| `claude -c` | Continue most recent conversation here |
| `claude -r <id\|name>` | Resume a session (no argument opens the picker) |
| `claude -w [name\|#pr]` | Start in a fresh git worktree |
| `claude --bg "task"` | Start a background session and return |
| `claude agents` | Agent view: dispatch and monitor background sessions |
| `claude --cloud "task"` | Start a cloud session for this repository |
| `claude doctor` | Install and settings diagnostics without a session |
| `claude --safe-mode` | Start with all customizations off |

## Essential CLI Flags

| Flag | Purpose |
|------|---------|
| `-p "prompt"` | Non-interactive mode |
| `--bare` | Skip hooks, skills, plugins, MCP, CLAUDE.md discovery (scripts) |
| `--model <alias>` | `opus`, `sonnet`, `haiku`, `fable`, `opusplan`, or full ID |
| `--effort <level>` | `low`, `medium`, `high`, `xhigh`, `max`, `ultracode` |
| `--permission-mode <mode>` | `default`, `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions` |
| `--output-format text\|json\|stream-json` | Output format (print mode) |
| `--json-schema <schema>` | Validated JSON in `structured_output` |
| `--max-turns <n>` / `--max-budget-usd <n>` | Cap turns / estimated spend |
| `--permission-prompts none` | Deny anything that would prompt (unattended runs) |
| `--allowedTools` / `--disallowedTools` | Allow or deny rules for this run |
| `--tools "Read,Bash"` | Restrict which built-in tools exist |
| `--append-system-prompt[-file]` | Add to the system prompt |
| `-n <name>` | Name the session |
| `--fork-session` | Resume into a new session ID |
| `--add-dir <path>` | Extra working directories |

## Slash Commands

| Command | Purpose |
|---------|---------|
| `/plan` | Plan mode: read and propose, change nothing |
| `/goal <condition>` | Keep working until the condition is verifiably met |
| `/context` | What is filling the window |
| `/compact [focus]` | Summarize to free context |
| `/btw <question>` | Side question that stays out of history |
| `/clear` | Fresh context (old session stays resumable) |
| `/rewind` | Restore code/conversation, or summarize part of it |
| `/branch` / `/fork` | Branch the conversation / copy it into a background session |
| `/model` / `/effort` / `/fast` | Model, effort level, fast mode |
| `/permissions` | Rules and recent auto mode denials |
| `/fewer-permission-prompts` | Propose an allowlist from your transcripts |
| `/sandbox` | Configure the shell sandbox |
| `/hooks` | View configured hooks and sources |
| `/mcp` | Server status, auth, reconnect |
| `/memory` | CLAUDE.md files and auto memory |
| `/code-review` / `/security-review` | Review the diff for bugs / vulnerabilities |
| `/simplify` | Parallel cleanup pass on changed code |
| `/batch <instruction>` | Large change split across worktree subagents |
| `/loop [interval] <prompt>` | Repeat a prompt in this session |
| `/schedule` | Cloud scheduled tasks |
| `/autofix-pr` | Cloud session that fixes CI failures on your PR |
| `/tasks` | Background work in this session |
| `/usage` | Cost, plan limits, usage breakdown |
| `/status` / `/doctor` | What loaded / setup checkup with fixes |
| `/resume` / `/rename` | Session picker / rename |

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Shift+Tab` | Cycle permission modes |
| `Esc` | Interrupt (work so far is kept) |
| `Esc Esc` | Clear input, or on empty input open rewind |
| `Ctrl+G` | Edit prompt or plan in your editor |
| `Ctrl+O` | Transcript viewer |
| `Ctrl+B` | Background a running command or agent |
| `Ctrl+T` | Toggle task checklist |
| `Ctrl+S` | Stash / restore prompt |
| `Ctrl+Enter` | Send queued messages now |
| `Option+P` / `Alt+P` | Switch model |
| `Option+O` / `Alt+O` | Toggle fast mode |
| `Ctrl+J` or `\` + `Enter` | Newline |
| `!command` | Run a shell command; Claude sees the output |
| `Ctrl+C` / `Ctrl+D` | Interrupt or clear / exit |

## Permission Modes

| Mode | Runs without asking |
|------|---------------------|
| **Manual** (`default`) | Reads only |
| **Accept edits** | Reads, in-directory edits, common filesystem commands |
| **Plan** | Reads; proposes instead of changing |
| **Auto** | Everything, reviewed by a background classifier (default for interactive sessions) |
| **Don't ask** | Pre-approved tools only; everything else denied |
| **Bypass** | Everything. Isolated containers and VMs only |

Deny rules apply in every mode. `-p` runs start in Manual unless you set a mode.

## Settings Precedence (highest to lowest)

1. **Managed** -- deployed by your organization; cannot be overridden
2. **Command line** -- `--settings` for this session
3. **Project local** -- `.claude/settings.local.json` (kept out of git)
4. **Shared project** -- `.claude/settings.json` (committed)
5. **User** -- `~/.claude/settings.json`

List keys like `permissions.allow` merge across files. Rule evaluation: **Deny > Ask > Allow**.

## CLAUDE.md Best Practices

- Target under 200 lines; every line costs context on every request
- Only what Claude cannot infer from code: build and test commands, deviations from convention, architectural decisions, environment quirks
- Move path-specific guidance into `.claude/rules/*.md` with `paths:` frontmatter
- Use `@path` imports for reference material
- Project CLAUDE.md is re-read after compaction; rules given only in conversation are not
- Add a "Compact instructions" section to steer what summaries keep
- Personal notes go in `CLAUDE.local.md` or `~/.claude/CLAUDE.md`
- Let auto memory collect the small lessons; prune it with `/memory`
- Run `/doctor prompt-audit` periodically

## Context Cost Model

**Every request:**
- CLAUDE.md and its imports, unscoped rules
- Skill descriptions (not bodies)
- MCP tool names (full schemas load on demand via tool search)

**On demand:**
- Path-scoped rules, nested CLAUDE.md, skill bodies, MCP tool schemas

**Isolated:**
- Subagent and workflow work runs in separate windows; only the final report returns

**Zero:**
- Hooks run outside the context window, unless they return context

## Model Selection

| Choice | Use for |
|--------|---------|
| **Opus** (default on most plans) | Most work; planning, architecture, ambiguous problems |
| **Sonnet** | Fast, cheaper implementation and routine edits |
| **Haiku** | High-volume, simple subtasks |
| **opusplan** | Opus in plan mode, Sonnet for execution |
| **Effort** | The bigger dial now: `low` for routine, `high`/`xhigh` for hard problems, `ultrathink` in a prompt for one hard turn |
| **Advisor** | `/advisor` lets a stronger model weigh in at key moments |

## Top Prompt Patterns

### Verification Criteria (highest leverage)
```
Add auth to the API. Run the auth tests after. /api/protected should
return 401 without a token and 200 with a valid one.
```

### Verified Completion
```
/goal all tests in tests/auth pass and the lint step exits 0
```

### Plan Before Execute
Start in plan mode (`/plan`), review the plan, press `Ctrl+G` to edit it, then approve into auto or accept-edits mode.

### Exhaustive Questioning
```
Before you start, ask me questions about anything you are uncertain
about. Do not proceed until you have asked them.
```

### Constraint Explicitness
```
Do not modify the database schema. You can add files in src/utils/
but not src/core/. No new dependencies.
```

### Adversarial Review
```
Use a subagent to review this diff as a skeptical senior engineer.
Report only problems you can demonstrate.
```

### Simplicity Directive
```
Write the simplest implementation that satisfies the requirement.
No speculative abstraction.
```

## Four-Phase Workflow

1. **Explore** -- Plan mode. Read the code. Ask questions.
2. **Plan** -- Still plan mode. Review and edit the plan.
3. **Implement** -- Approve into auto or accept-edits mode. Use `/goal` for a verifiable finish line.
4. **Commit** -- `/code-review`, then commit. Small commits are cheap insurance.

## MCP Configuration

**.mcp.json** (project root, commit to version control):
```json
{
  "mcpServers": {
    "local-tool": {
      "type": "stdio",
      "command": "node",
      "args": ["tools/server.js"],
      "env": { "API_KEY": "${TOOL_API_KEY}" }
    },
    "tracker": {
      "type": "http",
      "url": "https://mcp.example.com/mcp"
    }
  }
}
```

- `/mcp` for status, auth, and `reconnect all`; `claude mcp list` from the shell
- Tool search loads MCP schemas on demand; large results are the real cost
- Background subagents keep MCP access
- Rule pattern: `mcp__<server>__<tool>`, `mcp__<server>__*` for a whole server

## Subagents

| Type | Notes |
|------|-------|
| **Explore** | Read-only search; runs on a capable model, so not automatically cheap |
| **Plan** | Read-only research for plan mode |
| **general-purpose** | All tools; implementation work |
| **Fork** | Inherits the full conversation (`/subtask`) |
| **Custom** | `.claude/agents/*.md` or `--agents` JSON |

Subagents run in the background by default, up to 20 at a time, nested up to 3 levels. For larger fan-outs, use a dynamic workflow.

## Common Failure Modes and Quick Fixes

| Problem | Fix |
|---------|-----|
| Forgets instructions mid-session | Put the rule in CLAUDE.md or a rule file; lower the auto-compact window; start fresh per task |
| Shell variables do not persist | Same-line assignment, or `CLAUDE_ENV_FILE` from a SessionStart hook |
| MCP tools stop working | `/mcp reconnect all`; `claude mcp list` |
| Rewind misses changes | Checkpoints skip shell-command edits and background subagent edits; use git |
| Same error across sessions | Targeted CLAUDE.md line with wrong and right behavior |
| Setting has no effect | `/status` → Setting sources; `claude doctor` |
| Something broke after a config change | `claude --safe-mode` |
| Policy hook does not block | Exit 2, not exit 1; add `onFailure: "block"` |
| Auto mode blocks routine work | `/auto-mode-setup`; `/permissions` → Recently denied |
| Session lost | `claude -c`, or `claude -r` and search |
| Stop hook loops forever | Check `stop_hook_active`; or use `/goal` instead |

## Key Environment Variables

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | API key |
| `ANTHROPIC_MODEL` | Session model |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` / `_SONNET_` / `_HAIKU_` | Pin what each alias resolves to |
| `CLAUDE_CODE_EFFORT_LEVEL` | Effort level |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Auto-compact window (tokens) |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Model for subagents |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | `1` enables agent teams |
| `CLAUDE_CODE_ENABLE_TODO_TOOLS` | `1` enables task tools on every model |
| `MAX_THINKING_TOKENS` | Fixed thinking budget (older models only) |
| `ENABLE_TOOL_SEARCH` | MCP deferred loading |
| `CLAUDE_CODE_SIMPLE` | Same as `--bare` |

## Headless / CI Pattern

```bash
# Scripted run with a hard boundary
claude --bare -p "Fix the failing tests" \
  --allowedTools "Read,Edit,Bash(npm test *)" \
  --permission-mode dontAsk \
  --max-turns 30 --max-budget-usd 5 \
  --output-format json

# Pipe input
git diff HEAD~1 | claude -p "Review for security issues" --output-format json

# Structured output
claude -p "List the public API functions in src/" \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}}}'
```

## Hook Configuration (settings.json)

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/check-bash.sh",
            "if": "Bash(rm *)",
            "onFailure": "block"
          }
        ]
      }
    ]
  }
}
```

Exit 2 blocks (stderr goes to Claude). Exit 1 does not block. Handler types: `command`, `http`, `mcp_tool`, `prompt`, `agent`. Hooks cost no context unless they return some.

## Permission Rule Syntax

```json
{
  "permissions": {
    "deny": ["Read(./.env*)", "Edit(./secrets/**)", "Bash(rm -rf *)"],
    "ask": ["Bash(git push *)"],
    "allow": ["Bash(npm run test *)", "Bash(git diff *)"]
  }
}
```

Patterns: `Tool`, `Tool(specifier)`, `mcp__server__tool`. Bash prefix rules need a space before `*`.

## Git Worktrees for Parallel Work

```bash
claude -w feature-auth        # new worktree + session
claude -w "#482"              # worktree branched from a pull request
```

`.worktreeinclude` lists gitignored files to copy in; `worktree.symlinkDirectories` shares heavy directories.

## One-Third Reality

Plenty of tasks still need a second attempt. That is normal. The correction cycle is fast, and every investment in verification -- tests, types, linting, a `/goal` condition -- raises the first-pass rate.
