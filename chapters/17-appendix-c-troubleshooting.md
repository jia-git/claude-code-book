# Appendix C: Troubleshooting

Organized by symptom. No prose. Scan for your problem.

---

## First Moves for Any Problem

| Step | Command | What it tells you |
|------|---------|-------------------|
| 1 | `/status` | Version, model, account, connectivity, which settings files loaded, which managed source applies |
| 2 | `claude doctor` (shell) | Install health, settings-file validation errors, entries that were rejected -- without starting a session |
| 3 | `/doctor` (in session) | Setup checkup across installation, settings, extensions, and CLAUDE.md; proposes fixes you confirm. `/doctor prompt-audit` audits your instructions |
| 4 | `claude --safe-mode` | Starts with every customization off. If the problem disappears, it lives in your configuration |
| 5 | `/debug [description]` | Turns on debug logging and helps diagnose from the log |
| 6 | `/feedback` | Sends a report with the session attached (after a consent screen) |

---

## Bash Environment Variables Not Persisting

**Symptom:** Variables set in one shell command are gone in the next. Scripts that depend on `export` fail silently.

**Cause:** Each command runs in a fresh shell. The working directory persists; variables, aliases, and functions do not.

**Fix:**
- Set and use on one line: `NODE_ENV=production npm test`
- Chain dependent commands with `&&`
- Persist across commands with `CLAUDE_ENV_FILE`: a SessionStart hook appends `export` lines, and the file is sourced before every command
- Put project-wide variables in the settings `env` block
- Working directory drifting? `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR=1` returns to the project root after each command; `/cd` moves the whole session deliberately

---

## Context Exhaustion (Claude Forgets Instructions)

**Symptom:** Claude stops following rules, drops steps from a plan, or forgets decisions made earlier in the session.

**Cause:** The window filled and auto-compaction summarized the conversation. Anything said only in conversation is at the mercy of the summary.

**Fix:**
- Move must-keep rules into CLAUDE.md or `.claude/rules/` -- the project CLAUDE.md is re-read from disk after compaction, not summarized
- Add a "Compact instructions" section to CLAUDE.md naming what the summary must keep
- Run `/context` (or `/context all`) to see what is spending the window; `/skill-doctor` for skill costs
- Keep side questions out of history with `/btw`
- Delegate verbose investigation to subagents; their transcripts never enter your window
- Start a new session per task; `/clear` keeps the old one resumable
- Use `/rewind` → "Summarize from here" to compress just the part you no longer need

---

## Auto-Compaction Triggering Too Late (or Too Early)

**Symptom:** Quality degrades before compaction, or compaction fires while you still need the detail.

**Cause:** On million-token models the default window is large; long sessions degrade well before it fills.

**Fix:**
- Set a window for this session: `/autocompact 400k` or `claude --autocompact 400k`
- Persist it: `autoCompactWindow` (or per model in `modelSettings`)
- Override everywhere: `CLAUDE_CODE_AUTO_COMPACT_WINDOW` (tokens) or `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` (percentage; can only lower the trigger)
- Compact on your own schedule with `/compact <focus>` before starting a new phase
- Turn auto-compaction off: `autoCompactEnabled: false` or `DISABLE_AUTO_COMPACT=1`

---

## A Setting Does Not Take Effect

**Symptom:** You changed a setting and nothing happened.

**Cause:** A higher scope sets the same key, the file cannot set that key, the file is broken, or the folder is not trusted yet.

**Fix:**
- `/status` → Setting sources: confirm your file loaded and see which managed source applies
- `claude doctor`: lists entries Claude Code rejected
- Precedence is managed > `--settings` > project local > shared project > user. Environment variables like `ANTHROPIC_MODEL` beat the matching key
- `defaultMode: "auto"` or `"bypassPermissions"` is ignored in project files; put it in user settings
- Committed `allow` rules, `env`, and marketplaces wait for the workspace trust dialog
- `model` and `effortLevel` are read at startup; use `/model` and `/effort` mid-session

---

## A Customization Broke Something

**Symptom:** Startup errors, a hook that blocks everything, a plugin that slows every turn, unexpected model fallback.

**Fix:**
- `claude --safe-mode`: CLAUDE.md, skills, plugins, hooks, MCP, agents, output styles, and auto memory all off; authentication and permissions work normally
- If safe mode fixes it, re-enable in halves: `/plugin` to disable plugins, `/hooks` to see hook sources, `/mcp disable <server>`
- `claude --bare -p "..."` for a scripted run with no discovery at all

---

## Claude Repeats the Same Error

**Symptom:** The same mistake across sessions -- wrong import path, wrong test command, a deprecated API.

**Cause:** The correction lived in a conversation, not in persistent context.

**Fix:**
- Add a targeted line to CLAUDE.md or a path-scoped rule: `"Import from @app/db/utils, never @app/utils/db (deprecated)"`
- Describe the wrong behavior and the right one
- Check auto memory with `/memory`; tell Claude to remember the correction, or delete a stale note that is causing it
- Put team-wide corrections in the project CLAUDE.md, personal ones in `CLAUDE.local.md`

---

## Checkpoint Restore Does Not Undo Changes

**Symptom:** `/rewind` (or Esc Esc on an empty prompt) does not restore the expected file state.

**Cause:** Checkpoints track edits through the file tools only. Not tracked: changes from shell commands (`rm`, `mv`, `sed -i`), external editors, other sessions, background subagent edits, and files reached through symlinks. Snapshots older than the retention window (about 30 days) are gone.

**Fix:**
- Use git for real recovery: commit before risky operations, `git stash`, `git restore <file>`
- Prefer the file tools over shell commands for edits you might want to undo
- Run destructive or parallel work in a worktree (`claude -w`) so the main checkout is never at risk
- Ask Claude to back up before a destructive operation

---

## Deciding: Restart Fresh vs. Correct Mid-Stream

**Restart fresh when:**
- Context is full of wrong exploration
- The approach is wrong, not just the details
- Two corrections have not converged
- The session is near its compaction window

**Correct mid-stream when:**
- The approach is right and the fix is one sentence
- Claude has built up understanding you would lose

**Middle paths:**
- `/rewind` to the prompt before things went wrong, then rephrase
- `/branch` to try the alternative without losing the original
- `/clear` with a better prompt that names what failed

**Pattern:** Commit before deciding. A restart keeps the useful fragments; a correction gets a rollback point.

---

## Permission Prompts Never Stop

**Symptom:** The same command prompts every session; long tasks stall waiting for approval.

**Fix:**
- `/fewer-permission-prompts`: proposes allow rules from your recent transcripts
- Approve with "Yes, and don't ask again" (saved to `.claude/settings.local.json`)
- An `ask` rule in project or managed settings outranks your local `allow`; check `/permissions`
- Switch to auto mode (Shift+Tab) for long tasks, or accept-edits mode for edit-heavy work
- Enable the sandbox: sandboxed commands run without prompts

---

## Auto Mode Blocks Legitimate Work

**Symptom:** Auto mode denies pushes, deploys, or calls to internal services you consider routine.

**Cause:** The classifier does not know your infrastructure is trusted, or a stated boundary from earlier in the conversation is still in force.

**Fix:**
- `/permissions` → Recently denied; press `r` to retry one with manual approval
- `/auto-mode-setup` drafts `autoMode.environment` entries describing your trusted hosts, repositories, and buckets
- Inspect the rules: `claude auto-mode defaults` and `claude auto-mode config`
- Broad allow rules (`Bash(*)`, interpreters) are dropped in auto mode by design; write narrow ones
- After 3 consecutive or 20 total blocks auto mode pauses and prompts resume; that is a signal to configure, not to switch to bypass

---

## Sandboxed Command Fails

**Symptom:** A command that works in your terminal fails under Claude Code with network or permission errors.

**Fix:**
- The network allowlist starts empty: add the domains the command needs to `sandbox.network.allowedDomains`
- Writes outside the working directory need `sandbox.filesystem.allowWrite`
- Tools that cannot run sandboxed (some container and VM tools) go in `sandbox.excludedCommands`
- Claude may retry outside the sandbox with a prompt; set `allowUnsandboxedCommands: false` if that escape hatch is not acceptable
- On a machine where the sandbox cannot start, `sandbox.failIfUnavailable: true` makes that an error instead of a silent downgrade

---

## MCP Server Disconnected or Missing

**Symptom:** MCP tools fail mid-session, or never appear.

**Cause:** The server crashed, timed out at startup, needs re-authentication, or was not approved for this project.

**Fix:**
- Remote servers reconnect automatically with backoff; `/mcp` shows status, `/mcp reconnect all` forces it
- `claude mcp list` from the shell checks every server's health without a session
- Re-authenticate: `/mcp`, or `claude mcp login <name>` (works over SSH)
- Project servers need approval: check `enabledMcpjsonServers` or approve when prompted
- Slow server startup: raise `MCP_TIMEOUT`
- Policy: `allowedMcpServers` / `deniedMcpServers` or an organization-managed server list may exclude it
- `claude --debug='mcp'` shows connection details

---

## Session Lost After Crash or Terminal Close

**Symptom:** Work from a previous session seems gone.

**Cause:** Usually it is not gone. Transcripts are saved locally for 30 days by default (`cleanupPeriodDays`).

**Fix:**
- `claude -c` continues the most recent session in this directory
- `claude -r` opens the picker; `Ctrl+A` shows all projects, `Ctrl+W` all worktrees, `/` searches (paste a PR URL to find the session that opened it)
- `claude -r <name>` if you named it with `-n` or `/rename`
- Background sessions keep running after the terminal closes: `claude agents` or `claude attach <id>`
- Sessions started with `-p` do not appear in the picker; resume them by ID
- For work spanning days, keep state in files: a plan file, a progress note, CLAUDE.md updates

---

## Background Sessions Unresponsive

**Symptom:** Agent view hangs or reports that the background service did not respond.

**Fix:**
- `claude daemon status` to check the supervisor
- `claude daemon logs` to follow its log
- `claude daemon stop --any --keep-workers` to restart the supervisor without killing sessions
- `claude respawn <id>` restarts one session with its conversation intact

---

## Subagent Results Consuming Too Much Main Context

**Symptom:** After several subagents, the main window fills fast.

**Cause:** Each subagent's final report lands in the main conversation.

**Fix:**
- Ask subagents for summaries with file paths, not full findings
- Have them write detailed results to files and return the path
- For large fan-outs, use a dynamic workflow: intermediate results stay in the script, only the final answer returns
- `/compact` after you have acted on the results

---

## Claude Produces Over-Complex Solutions

**Symptom:** Elaborate abstractions and extension points for a simple problem.

**Fix:**
- CLAUDE.md: `"Prefer the simplest solution that meets the requirement. No speculative abstraction."`
- State scope: `"One-off script, not a library"`, `"No new dependencies"`, `"Single file"`
- Run `/simplify` on the result
- Interrupt with Esc and ask why the complexity is needed

---

## Unexpected Cost or Usage

**Symptom:** A session or a month costs more than expected; plan limits arrive early.

**Fix:**
- `/usage`: session cost, plan limits, and a breakdown of what consumed them
- Background subagents, agent teams, and workflows multiply spend; check `/tasks` for work still running
- Lower effort for routine work (`/effort`), and reserve the most capable model for planning
- Long idle gaps expire the prompt cache; resume promptly or compact first
- In scripts: `--max-budget-usd` and `--max-turns`

---

## Verifying Proxy or Gateway Configuration

**Symptom:** Authentication errors, TLS failures, or unexpected model routing behind a corporate proxy or LLM gateway.

**Fix:**
- `/status` shows the endpoint and account in use; `claude doctor` checks connectivity
- Check `ANTHROPIC_BASE_URL`, `HTTPS_PROXY`, `NO_PROXY`
- TLS interception: `CLAUDE_CODE_CERT_STORE=system` to trust the OS certificate store
- Mutual TLS: `CLAUDE_CODE_CLIENT_CERT` and `CLAUDE_CODE_CLIENT_KEY`
- Gateway that rejects newer request fields: `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1`
- Gateway handling cloud-provider auth: set that provider's `CLAUDE_CODE_SKIP_<PROVIDER>_AUTH=1`

---

## Debugging with Verbose Output

**Symptom:** Unclear why Claude made a decision, why a tool failed, or what a hook received.

**Fix:**
- `Ctrl+O`: transcript viewer with tool details and the model behind each message
- `claude --debug='hooks'` (or `mcp`, `startup`, ...) for filtered debug logs; `--debug-file <path>` to capture them
- `/debug` turns logging on mid-session
- `--verbose` for full turn-by-turn output in print mode

---

## Hook Not Firing

**Symptom:** A configured hook never runs. No output, no error.

**Cause:** Untrusted folder, wrong schema, non-matching matcher or `if` pattern, or hooks disabled by policy.

**Fix:**
- Accept the workspace trust dialog: repository hooks wait for it
- Check the schema: event → `[{ "matcher": ..., "hooks": [{ "type": "command", ... }] }]`
- Unknown event names are skipped with a Settings Warning; `claude doctor` lists them
- Matchers are case-sensitive (`Bash`, not `bash`); a matcher with regex characters is a regex
- An `if` pattern narrows further: `"if": "Bash(git push *)"`
- Check `disableAllHooks`, and `allowManagedHooksOnly` in managed settings
- `/hooks` lists every configured hook and its source
- Script executable (`chmod +x`) with a shebang line
- `claude --debug='hooks'` shows which hooks matched

---

## Hook Fires but Does Not Block

**Symptom:** A policy hook runs, but the action proceeds anyway.

**Cause:** Exit 1 is a non-blocking error. Only exit 2 (or a JSON deny) blocks.

**Fix:**
- Exit `2` with the reason on stderr
- Or exit `0` with `hookSpecificOutput.permissionDecision: "deny"`
- Add `"onFailure": "block"` so a crash or timeout fails closed
- PermissionRequest ignores exit 2; return `decision.behavior: "deny"` instead
- Async hooks cannot block; remove `async` from enforcement hooks

---

## Hook JSON Parsing Errors

**Symptom:** The hook exits 0 but its decision is ignored.

**Cause:** Extra text on stdout, often from a shell profile, or a missing `hookEventName`.

**Fix:**
- stdout must contain only the JSON object; send diagnostics to stderr: `echo "debug" >&2`
- Include `hookEventName` inside `hookSpecificOutput`
- Check whether `.bashrc` or `.zshrc` prints on startup
- Run the script by hand with sample input and inspect raw output

---

## WebSocket Scaling Limitations

**Symptom:** Real-time features generated by Claude Code (chat, live dashboards, streaming feeds) fail under load.

**Cause:** Generated scaffolding is correct but omits production concerns: connection limits, backpressure, reconnection, heartbeats.

**Fix:**
- Ask for connection pooling, reconnection, and heartbeat logic explicitly
- Put load-test targets in the acceptance criteria and have Claude run them
- Review for maximum connection limits, unclosed connections, and graceful degradation
- Prefer an established real-time messaging library over hand-rolled socket handling

---

## FIX Protocol Limitations

**Symptom:** Generated FIX message builders compile but fail conformance testing.

**Cause:** FIX semantics are deep and venue-specific; plausible code is not conformant code.

**Fix:**
- Treat generated FIX code as scaffolding
- Have a domain expert review message builders
- Test against a conformance suite before production
- Prefer a broker's standard REST or WebSocket API where one exists
