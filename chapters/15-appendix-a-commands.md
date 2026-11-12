# Appendix A: Command Reference

Current as of Claude Code 2.1.31x (late 2026). Run `claude --help`, `/help`, and `/release-notes` for the version in front of you; `claude --help` does not list every flag.

## CLI Commands

| Command | Description |
|---------|-------------|
| `claude` | Start interactive session in current directory |
| `claude "task"` | Start interactive session with initial prompt |
| `claude -p "task"` | Non-interactive run (the Agent SDK via CLI); print result and exit |
| `cat file \| claude -p "task"` | Process piped content (stdin capped at 10MB) |
| `claude -c` | Continue most recent conversation in current directory |
| `claude -r <id\|name>` | Resume a session by ID, name, or transcript path |
| `claude agents` | Open agent view: dispatch and monitor background sessions (`--json` for scripts) |
| `claude attach <id>` / `logs` / `stop` / `respawn` / `rm` | Manage a background session from the shell (`respawn --all` picks up a new binary) |
| `claude auth login` / `logout` / `status` | Sign in (`--console` for API billing, `--sso`), sign out, show auth status |
| `claude doctor` | Read-only install and settings diagnostics without starting a session |
| `claude update` / `claude install [version\|stable\|latest]` | Update, or install a specific native build |
| `claude mcp add` / `list` / `get` / `remove` | Manage MCP servers |
| `claude mcp login <name>` / `logout <name>` | Run or clear an MCP server's OAuth flow from the shell |
| `claude plugin ...` | Manage plugins: `install`, `list`, `enable`, `disable`, `validate`, `eval`, `prune`, `init` |
| `claude auto-mode defaults` / `config` / `reset` | Print built-in classifier rules, effective config, or reset your overrides |
| `claude ultrareview [target]` | Run a cloud multi-agent review non-interactively; `--json`, `--post` |
| `claude remote-control` | Host sessions you can drive from the web or mobile app |
| `claude purge [path]` | Delete local state for a project (`--dry-run`, `--all`) |
| `claude import [source]` | Bring configuration over from other coding agents |
| `claude setup-token` | Generate a long-lived token for CI (subscription accounts) |
| `claude self-hosted-runner` | Register this machine as a runner for a self-hosted cloud environment |
| `claude daemon status` / `logs` / `run` / `stop` | Inspect, run, or stop the background-session supervisor |
| `claude gateway --config <file>` | Run the self-hosted Claude apps gateway (administrators) |

## CLI Flags

### Session and Input

| Flag | Description |
|------|-------------|
| `--continue`, `-c` | Continue most recent conversation in this directory |
| `--resume`, `-r <id\|name>` | Resume session by ID, name, or `.jsonl` path; no argument opens the picker |
| `--fork-session` | New session ID preserving history (with `--resume` or `--continue`) |
| `--from-pr <number\|url>` | Open the picker filtered to sessions linked to a pull or merge request |
| `--name`, `-n <name>` | Name the session |
| `--model <alias\|id>` | Session model: `opus`, `sonnet`, `haiku`, `fable`, `best`, `opusplan`, or full ID |
| `--effort <level>` | `low`, `medium`, `high`, `xhigh`, `max`, or `ultracode` |
| `--fallback-model <list>` | Comma-separated models tried in order when the primary is overloaded or unavailable |
| `--advisor <model>` | Enable the advisor tool with a stronger model |
| `--agent <name>` | Run a custom agent as the main thread |
| `--agents <json\|path>` | Define subagents dynamically (file path accepted with `-p`) |
| `--permission-mode <mode>` | `default` (alias `manual`), `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions` |
| `--worktree`, `-w [name\|#pr]` | Start in an isolated git worktree under `.claude/worktrees/` |
| `--bg`, `--background` | Start as a background session and return immediately (not with `-p`) |
| `--exec <cmd>` | With `--bg`, run a shell command as a background job |
| `--add-dir <paths>` | Grant access to additional directories |
| `--teleport [id]` | Pull a cloud session into this terminal |
| `--environment <id>` | Start a cloud session on a self-hosted environment (`--ref <branch>` sets its checkout) |
| `--cloud "task"` | Start a new cloud session for this repository (`--remote` is a deprecated alias) |
| `--remote-control`, `--rc [name]` | Start an interactive session also drivable from web or mobile |
| `--desktop` | Open the desktop app on this directory (with `-c` or `-r <id>` to hand off a session) |
| `--chrome` / `--no-chrome` | Enable or disable the browser extension integration |
| `--ide` | Auto-connect to the IDE if exactly one is available |
| `--teammate-mode <mode>` | Agent team display: `in-process` (default), `auto`, or a split-pane mode for supported terminals |
| `--channels <list>` | Listen for channel notifications from listed MCP plugins (research preview) |
| `--safe-mode` | Start with all customizations disabled for troubleshooting |
| `--restricted` | Evaluation mode: no command-running tools, no user/project settings |
| `--bare` | Skip discovery of hooks, skills, plugins, MCP, memory, CLAUDE.md (recommended for scripts) |
| `--settings <file\|json>` | Session-only settings overrides |
| `--setting-sources <list>` | Load only `user`, `project`, and/or `local` settings |
| `--plugin-dir <path>` / `--plugin-url <url>` | Load a plugin (directory, `.zip`, or URL) for this session |
| `--disable-slash-commands` | Disable all skills and commands for this session |
| `--session-id <uuid>` | Use a specific session ID |
| `--ax-screen-reader` | Flat, screen-reader-friendly output |
| `--mcp-config <files>` / `--strict-mcp-config` | Load MCP servers from files; ignore all other MCP config |
| `--autocompact <auto\|tokens>` | Auto-compact window for this session |

### System Prompt

| Flag | Description |
|------|-------------|
| `--system-prompt <text>` | Replace the default system prompt |
| `--system-prompt-file <path>` | Replace with file contents |
| `--append-system-prompt <text>` | Append to the default prompt |
| `--append-system-prompt-file <path>` | Append file contents |
| `--system-prompt-snapshot off` | Rebuild the prompt on every request instead of reusing the recorded one |
| `--exclude-dynamic-system-prompt-sections` | Move per-user context out of the system prompt for cross-user cache reuse |
| `--append-subagent-system-prompt[-file]` | Append text to every subagent's prompt (`-p` only) |

All system prompt flags work in interactive and non-interactive mode. A line containing only `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` in replacement text splits cached and per-run sections.

### Non-Interactive (Print Mode)

| Flag | Description |
|------|-------------|
| `--output-format text\|json\|stream-json` | Output format (`stream-json` needs `--verbose`) |
| `--input-format text\|stream-json` | Input format |
| `--json-schema <schema>` | Return validated JSON in `structured_output` |
| `--max-turns <n>` | Limit agentic turns |
| `--max-budget-usd <amount>` | Stop when estimated spend reaches the cap |
| `--permission-prompts host\|none` | Who answers permission prompts; `none` denies them |
| `--permission-prompt-tool <mcp_tool>` | Delegate permission decisions to an MCP tool |
| `--no-session-persistence` | Do not save the session |
| `--include-partial-messages` | Stream partial events (with `stream-json`) |
| `--include-hook-events` | Include hook lifecycle events in the stream |
| `--forward-subagent-text` | Include subagent text and thinking in the stream |
| `--init` / `--maintenance` | Run Setup hooks with that matcher before the session |
| `--init-only` | Run Setup and SessionStart hooks, then exit |
| `--prompt-suggestions` | Emit a predicted next prompt after each turn (`stream-json`) |

### Permission Flags

| Flag | Description |
|------|-------------|
| `--allowedTools <rules>` | Rules that run without prompting |
| `--disallowedTools <rules>` | Deny rules; a bare tool name removes the tool |
| `--tools <list>` | Restrict which built-in tools exist (`""` none, `"default"` all) |
| `--dangerously-skip-permissions` | Bypass mode; only in isolated containers or VMs |
| `--allow-dangerously-skip-permissions` | Add bypass mode to the Shift+Tab cycle without starting in it |

### Diagnostics

| Flag | Description |
|------|-------------|
| `--debug[=categories]` | Debug logging, optionally filtered (`--debug='mcp,startup'`) |
| `--debug-file <path>` | Write debug log to a file |
| `--verbose` | Full turn-by-turn output |
| `--version`, `-v` | Print version |

## Slash Commands (Selected)

Type `/` to see everything available to you. Commands marked *skill* are bundled skills Claude can also invoke on its own.

| Command | Description |
|---------|-------------|
| `/help` | Show available commands |
| `/clear [name]` | New conversation with empty context (previous one stays resumable) |
| `/compact [focus]` | Summarize the conversation to free context |
| `/autocompact [auto\|tokens]` | Set the auto-compact window |
| `/context [all]` | Context usage grid; `all` adds per-MCP-tool costs |
| `/btw [question]` | Side question that never enters conversation history |
| `/rewind` | Restore code/conversation, or summarize from/up to a point (aliases `/checkpoint`, `/undo`) |
| `/branch [name]` | Branch the conversation and switch into the copy |
| `/fork [prompt]` | Copy the conversation into a new background session |
| `/subtask <task>` | Spawn a forked subagent that inherits the conversation |
| `/background [prompt]` | Move this session to the background (agent view) |
| `/resume [session]` | Session picker; paste a PR URL to find its session |
| `/rename [name]` | Rename the session |
| `/cd <path>` | Move the session to another directory |
| `/add-dir <path>` | Add a working directory |
| `/config [key=value]` | Settings UI, or set a setting directly |
| `/model [model]` | Switch model; arrows adjust effort; `s` applies to session only |
| `/effort [level\|auto\|ultracode]` | Set effort level or turn on ultracode |
| `/fast [on\|off]` | Toggle fast mode |
| `/advisor [model\|off]` | Enable the advisor tool |
| `/plan [task]` | Enter plan mode |
| `/goal [condition\|clear]` | Keep working until an evaluator confirms the condition |
| `/loop [interval] [prompt]` | *skill* -- run a prompt repeatedly; self-paced without an interval |
| `/schedule [description]` | Create or manage cloud scheduled tasks (alias `/routines`) |
| `/permissions` | Manage allow/ask/deny rules; review recent auto mode denials |
| `/auto-mode-setup` | Draft trusted-environment entries for auto mode |
| `/fewer-permission-prompts` | *skill* -- propose an allowlist from your transcripts |
| `/sandbox` | Configure the shell sandbox |
| `/hooks` | View configured hooks and their sources |
| `/update-config [request]` | *skill* -- describe a settings change; Claude edits settings.json |
| `/mcp [reconnect\|enable\|disable ...]` | Manage MCP servers and authentication |
| `/plugin [subcommand]` | Plugin manager; `/reload-plugins` applies changes |
| `/skills` | List skills; toggle visibility; sort by token cost |
| `/skill-doctor` | Show each skill's context cost and usage |
| `/reload-skills` | Rescan skill and command directories |
| `/memory` | Edit CLAUDE.md files; browse and toggle auto memory |
| `/init` | Generate a starter CLAUDE.md |
| `/import` | Bring over configuration from other coding agents |
| `/doctor [prompt-audit]` | *skill* -- full setup checkup with proposed fixes |
| `/debug [description]` | *skill* -- turn on debug logging and troubleshoot |
| `/code-review [effort\|ultra] [--fix] [--comment] [target]` | *skill* -- review changes for bugs (alias `/review`; `ultra` = cloud ultrareview) |
| `/security-review` | Review the branch diff for security issues |
| `/simplify [target]` | *skill* -- parallel cleanup review, applies fixes |
| `/batch <instruction>` | *skill* -- split a large change across 5-30 worktree subagents |
| `/verify`, `/run` | *skill* -- launch and drive your app to confirm a change works |
| `/deep-research <question>` | Bundled dynamic workflow: cited, cross-checked report |
| `/workflows` | Watch, pause, resume, or save dynamic workflow runs |
| `/tasks` | Background work in this session (alias `/bashes`); `t` teleports to cloud sessions |
| `/list-agents` | Sessions and agents Claude can message (alias `/peers`) |
| `/diff` | Review working-tree changes (live panel in fullscreen mode) |
| `/usage` | Session cost, plan limits, usage breakdown (aliases `/cost`, `/stats`) |
| `/insights` | HTML report on how you use Claude Code |
| `/status` | Version, model, account, connectivity, managed-settings source |
| `/statusline` | Configure the status line |
| `/output-style [style]` | List or switch output styles (Concise, Explanatory, custom) |
| `/teleport` | Pull a cloud session into this terminal (alias `/tp`) |
| `/remote-env` | Choose the default cloud environment |
| `/autofix-pr [prompt]` | Hand the current PR to a cloud session that fixes CI failures and comments |
| `/remote-control` | Make this session drivable from web or mobile (alias `/rc`) |
| `/desktop` | Continue this session in the desktop app (alias `/app`) |
| `/chrome` | Configure the browser extension integration |
| `/ide` | Manage IDE integrations |
| `/tui [default\|fullscreen]` | Switch renderer |
| `/voice [hold\|tap\|off]` | Voice dictation |
| `/terminal-setup` | Install multiline-input key bindings for your terminal |
| `/team-onboarding` | Generate an onboarding guide from your usage |
| `/powerup` | Interactive feature lessons |
| `/release-notes` | Browse the changelog |
| `/export [file]` | Export the conversation as text |
| `/feedback`, `/bug` | Send feedback or a bug report (with consent screen) |

**Removed or renamed since the first edition:** `/vim` (use `/config` → Editor mode), `/pr-comments` (ask Claude directly), `/ultraplan` (use plan mode), `/cost` and `/stats` (now `/usage` aliases), `/review` (now an alias of `/code-review`), `/agents` (now prints where agent files live; ask Claude to create agents), the `&` cloud prefix (use `claude --cloud`).

## Keyboard Shortcuts

### General

| Shortcut | Action |
|----------|--------|
| `Enter` | Send prompt / confirm |
| `Esc` | Interrupt Claude (work so far is kept), or close a dialog |
| `Esc` `Esc` | Clear input (draft goes to history), or on empty input open the rewind menu |
| `Ctrl+C` | Interrupt; on empty input, first press clears, second exits |
| `Ctrl+D` | Exit (press twice) |
| `Shift+Tab` | Cycle permission modes (Manual → Accept edits → Plan → enabled optional modes → Auto) |
| `Ctrl+Enter` / `Ctrl+X Ctrl+S` | Send queued messages now |
| `Ctrl+G` / `Ctrl+X Ctrl+E` | Open prompt (or plan) in your text editor |
| `Ctrl+S` | Stash or restore the current prompt |
| `Ctrl+R` | Reverse-search prompt history |
| `Ctrl+L` | Redraw the screen |
| `Ctrl+Z` | Suspend (Unix) |
| `Up` / `Down` | Move cursor or navigate history |
| `?` on empty input | Shortcut help |

### Session and Model

| Shortcut | Action |
|----------|--------|
| `Option+P` / `Alt+P` | Switch model without clearing the prompt |
| `Option+T` / `Alt+T` | Toggle extended thinking (no effect on models where thinking is always on) |
| `Option+O` / `Alt+O` | Toggle fast mode |

### Tools and Output

| Shortcut | Action |
|----------|--------|
| `Ctrl+O` | Toggle transcript viewer (tool details, model per message) |
| `Ctrl+B` | Background running Bash commands and agents (press twice in a multiplexer) |
| `Ctrl+X Ctrl+K` | Stop all background subagents (press twice to confirm) |
| `Ctrl+T` | Toggle Claude's task checklist |
| `Ctrl+V` / `Cmd+V` / `Alt+V` | Paste image from clipboard (varies by platform and terminal) |
| `←` on empty prompt | Background this session and open agent view |

## Permission Modes

| Mode (config value) | Runs without a prompt | Best for |
|------|----------|----------|
| **Manual** (`default`) | Reads only | Reviewing every action |
| **Accept edits** (`acceptEdits`) | Reads, in-directory edits, common filesystem commands | Iterating on code you review afterward |
| **Plan** (`plan`) | Reads (plus classifier-approved commands where auto is available) | Exploring before changing anything |
| **Auto** (`auto`) | Everything, reviewed by a background classifier | Long tasks; default for interactive sessions |
| **Don't ask** (`dontAsk`) | Pre-approved tools only; anything else denied | Locked-down CI and scripts |
| **Bypass** (`bypassPermissions`) | Everything | Isolated containers and VMs only |

Deny rules apply in every mode. Protected paths are never auto-approved outside bypass. `-p` runs start in Manual unless a mode is set.

## Output Formats

### `--output-format text`
Plain text response. Default for print mode.

### `--output-format json`
Single JSON object on completion, including `result`, `session_id`, `is_error`, usage, `total_cost_usd` with a per-model breakdown (client-side estimate), `structured_output` when `--json-schema` is used, and `permission_denials`.

### `--output-format stream-json`
Newline-delimited JSON events. Starts with a `system/init` event listing model, tools, MCP server status, loaded plugins, and any load errors; ends with a `result` event. Also emits `system/api_retry`, `permission_denied`, and (with flags) partial messages, hook events, and subagent messages tagged by `parent_tool_use_id`.

## Tool Names for Permission Rules

| Tool | Description |
|------|-------------|
| `Bash`, `PowerShell` | Shell command execution |
| `Monitor` | Background command whose output streams to Claude |
| `Read` | File reading (rules use gitignore-style paths) |
| `Edit` | File editing; `Edit(...)` rules cover all built-in file-editing tools |
| `Write` | File creation/overwrite |
| `NotebookEdit` | Notebook cell editing |
| `Glob`, `Grep` | File and content search (absent by default on Linux/macOS/WSL, where Claude searches via the shell) |
| `LSP` | Code intelligence via language-server plugins |
| `WebFetch`, `WebSearch` | Web access (`WebFetch(domain:example.com)`) |
| `Agent` | Spawn a subagent (formerly `Task`; `Agent(name)`, `Agent(model:opus)`) |
| `Skill` | Invoke a skill (`Skill(deploy *)`) |
| `Workflow` | Run a dynamic workflow (`Workflow(name)`) |
| `SendMessage`, `ListAgents` | Message other agents and sessions |
| `TaskCreate`, `TaskGet`, `TaskList`, `TaskUpdate`, `TaskStop` | Task list and background work (task tools opt-in on newest models) |
| `CronCreate`, `CronList`, `CronDelete`, `ScheduleWakeup` | Session-scoped scheduling |
| `AskUserQuestion` | Multiple-choice questions to the user |
| `EnterPlanMode`, `ExitPlanMode`, `EnterWorktree`, `ExitWorktree` | Mode and worktree transitions |

Permission rules use `Tool` or `Tool(specifier)`. Bash prefix rules need a space before `*`: `Bash(git diff *)`. Deny and ask rules can match parameters: `Tool(param:value)`.

## MCP Tool Pattern for Permission Rules

MCP tools are named `mcp__<server>__<tool>`; plugin servers are `mcp__plugin_<plugin>_<server>__<tool>`.

- `mcp__myserver__query` -- specific tool on a specific server
- `mcp__myserver__*` -- all tools on that server (exact server name only)
- `mcp__*` in a deny list -- all MCP tools

---

## Multiline Input Methods

| Method | Shortcut | Context |
|--------|----------|---------|
| Quick escape | `\` + `Enter` | Works in all terminals |
| Shift+Enter | `Shift+Enter` | Works natively in many modern terminals; run `/terminal-setup` for the rest |
| Option key | `Option+Enter` | macOS, after setting Option as Meta in your terminal |
| Control sequence | `Ctrl+J` | Works in any terminal without configuration |
| Paste | Paste directly | Multi-line pastes are detected automatically |

---

## Shell Mode (`!` Prefix)

Run a shell command directly by prefixing input with `!`:

```
! npm test
! git status
```

- Runs the command, adds its output to the conversation, and Claude responds to it
- Shows real-time progress; `Ctrl+B` backgrounds long-running commands
- History-based autocomplete with `Tab` from previous `!` commands in the project

---

## Background Bash Commands

Long-running commands can run in the background while the conversation continues.

**Triggering:** ask Claude to run something in the background, press `Ctrl+B` during a running command, or let a command that reaches its timeout move to the background automatically.

**Behavior:** output is written to a file Claude reads as needed; `/tasks` lists, inspects, and stops background work. For streaming reaction to output (tailing a log, watching a build), Claude uses the Monitor tool instead.

**Disable:** `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` (also disables background subagents and Ctrl+B).

---

## Prompt Suggestions

After Claude responds, a grayed-out next-prompt suggestion may appear.

- `Tab` or `Right` places it in the input; `Enter` submits
- Start typing to dismiss
- Generated by a short background request that reuses the prompt cache; skipped near usage limits
- Turn off in `/config`

---

## Task List

Where the task tools are enabled, Claude keeps a checklist for multi-step work, visible in the status area.

- `Ctrl+T` toggles the checklist
- The task tools are off by default on the newest models; enable with `CLAUDE_CODE_ENABLE_TODO_TOOLS=1`
- Share a task list across sessions: `CLAUDE_CODE_TASK_LIST_ID=my-project claude`

---

## Session Picker Shortcuts

`/resume` (or `claude --resume` with no argument) opens the picker.

| Shortcut | Action |
|----------|--------|
| `Up` / `Down` or `k` / `j` | Navigate sessions |
| `Right` / `Left` | Expand or collapse grouped sessions |
| `Enter` | Resume highlighted session |
| `1`-`9` | Resume session at that position |
| `Space` | Preview session |
| `Ctrl+R` | Rename highlighted session |
| `/` or type | Search; paste a PR URL to find the session that created it |
| `Ctrl+A` | Toggle all projects on this machine |
| `Ctrl+W` | Toggle all worktrees of this repository |
| `Ctrl+B` | Filter to the current git branch |
| `Esc` | Exit picker or search |

Background sessions appear marked `bg`. Sessions started with `-p`, the SDK, or a first-prompt `/loop` are hidden (resume them by ID).

---

## Agent View Shortcuts

`claude agents` opens agent view (research preview).

| Shortcut | Action |
|----------|--------|
| Type + `Enter` | Dispatch a new background session |
| `Space` | Peek at latest output and reply without attaching |
| `Enter` / `Right` | Attach to a session |
| `Left` on empty prompt, `/exit` | Detach (session keeps running) |
| `Ctrl+X`, `/stop` | Stop a session |
| `Ctrl+T` | Pin a session so it stays running while idle |
| `Ctrl+S` | Group by directory |
| `Ctrl+R` | Rename |

---

## Vim Editor Mode

Enable via `/config` → Editor mode (the `/vim` command was removed).

### Mode Switching

| Command | Action | From Mode |
|---------|--------|-----------|
| `Esc` or `Ctrl+[` | Enter NORMAL mode | INSERT, VISUAL |
| `i` | Insert before cursor | NORMAL |
| `I` | Insert at beginning of line | NORMAL |
| `a` | Insert after cursor | NORMAL |
| `A` | Insert at end of line | NORMAL |
| `o` | Open line below | NORMAL |
| `O` | Open line above | NORMAL |
| `v` / `V` | Character-wise / line-wise visual selection | NORMAL |

### Navigation (NORMAL Mode)

| Command | Action |
|---------|--------|
| `h`/`j`/`k`/`l` | Move left/down/up/right |
| `w` | Next word |
| `e` | End of word |
| `b` | Previous word |
| `0` | Beginning of line |
| `$` | End of line |
| `^` | First non-blank character |
| `gg` | Beginning of input |
| `G` | End of input |
| `f{char}` | Jump to next occurrence of character |
| `F{char}` | Jump to previous occurrence of character |
| `t{char}` | Jump to just before next occurrence |
| `T{char}` | Jump to just after previous occurrence |
| `;` | Repeat last f/F/t/T motion |
| `,` | Repeat last f/F/t/T motion in reverse |
| `%` | Jump to matching bracket |

In NORMAL mode, if the cursor is at the beginning or end of input and cannot move further, the arrow keys navigate command history instead.

### Editing (NORMAL Mode)

| Command | Action |
|---------|--------|
| `x` | Delete character |
| `dd` | Delete line |
| `D` | Delete to end of line |
| `dw`/`de`/`db` | Delete word/to end/back |
| `cc` | Change line |
| `C` | Change to end of line |
| `cw`/`ce`/`cb` | Change word/to end/back |
| `yy`/`Y` | Yank (copy) line |
| `yw`/`ye`/`yb` | Yank word/to end/back |
| `p` | Paste after cursor |
| `P` | Paste before cursor |
| `>>` | Indent line |
| `<<` | Dedent line |
| `J` | Join lines |
| `.` | Repeat last change |

### Text Objects (NORMAL Mode)

Text objects work with operators like `d`, `c`, and `y`:

| Command | Action |
|---------|--------|
| `iw`/`aw` | Inner/around word |
| `iW`/`aW` | Inner/around WORD (whitespace-delimited) |
| `i"`/`a"` | Inner/around double quotes |
| `i'`/`a'` | Inner/around single quotes |
| `i(`/`a(` | Inner/around parentheses |
| `i[`/`a[` | Inner/around brackets |
| `i{`/`a{` | Inner/around braces |
