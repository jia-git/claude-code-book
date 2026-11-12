# Appendix B: Configuration Reference

Current as of Claude Code 2.1.31x (late 2026). Settings files are strict JSON: no comments, no trailing commas. Run `/status` to see which files loaded and `claude doctor` to see which entries were rejected.

## Settings Files and Precedence

| Priority | Scope | Location | Shared |
|----------|-------|----------|--------|
| 1 (highest) | Managed | `managed-settings.json` in a system directory, MDM/OS policy, or server-managed settings from the admin console | Organization-wide |
| 2 | Command line | `--settings <file-or-json>` | This session only |
| 3 | Project local | `.claude/settings.local.json` | No (kept out of git automatically) |
| 4 | Shared project | `.claude/settings.json` | Yes (commit it) |
| 5 (lowest) | User | `~/.claude/settings.json` | No |

**Rules worth memorizing:**

- A higher scope overrides the same key below it, but **list keys merge**: `permissions.allow` from three files is one combined list. Exceptions: `fallbackModel` and `modelPicker` take the whole value from one source; a managed `availableModels` is applied as-is.
- Environment variables are not a level in the stack. Each variable/key pair has its own rule (`ANTHROPIC_MODEL` beats the `model` key from any file).
- `permissions.defaultMode` set to `auto` or `bypassPermissions` is ignored in project and local files. Set it in user or managed settings.
- `permissions.allow`, `additionalDirectories`, `extraKnownMarketplaces`, and most `env` values from a committed file wait until the folder is trusted. `deny` and `ask` rules apply immediately.
- A few restrictive values are honored from any scope even against managed settings (for example `permissions.blockReadsOutsideWorkingDirectories: true`, a lower `maxEffortLevel`).
- Most edits apply to a running session without a restart. `model` and `effortLevel` are read once at startup; use `/model` and `/effort` mid-session.
- `~/.claude.json` is Claude Code's own state file: account, MCP servers at user and local scope, per-project trust, and global config keys written by `/config`. You rarely edit it by hand.

### JSON Schema

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json"
}
```

Gives autocomplete and validation in editors that support JSON schema. The schema can lag new releases.

---

## Settings Keys (Selected)

Keys marked **[M]** are honored only in managed settings.

### Permissions

| Key | Description |
|-----|-------------|
| `permissions.allow` / `.ask` / `.deny` | Rule lists. Deny beats ask beats allow |
| `permissions.defaultMode` | Starting mode: `default`, `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions` |
| `permissions.additionalDirectories` | Extra working directories (file access only; most `.claude/` config is not discovered there) |
| `permissions.blockReadsOutsideWorkingDirectories` | Deny reads outside the working directories |
| `permissions.disableBypassPermissionsMode` | Set to `"disable"` to remove bypass mode |
| `disableAutoMode` | Remove auto mode from the mode cycle |
| `autoMode` | Classifier configuration: `environment` (trusted infrastructure), `allow`, hard-deny entries, `classifyAllShell` |
| `useAutoModeDuringPlan` | Let plan mode use the auto mode classifier for commands |
| `allowManagedPermissionRulesOnly` **[M]** | Ignore allow/ask/deny rules from user and project files |

### Permission Rule Syntax

| Rule | Matches |
|------|---------|
| `Bash` | Every shell command |
| `Bash(npm run test *)` | Commands starting with `npm run test ` (space before `*` matters) |
| `Bash(git diff *)` | Any `git diff` invocation |
| `Read(./.env*)` | Reads of `.env` files relative to the working directory |
| `Edit(./migrations/**)` | Edits anywhere under `migrations/` (covers all file-editing tools) |
| `Read(~/secrets/**)` | Paths under the home directory |
| `WebFetch(domain:example.com)` | Fetches from one domain |
| `Agent(Explore)` | One named subagent |
| `Agent(model:opus)` | Subagent calls that request a model (deny/ask rules only) |
| `Skill(deploy *)` | A skill and its arguments |
| `mcp__tracker__*` | All tools on one MCP server (exact server name) |
| `Bash(dangerouslyDisableSandbox:true)` | Parameter match (deny/ask rules) |

Paths: `./` relative to the working directory, `~/` home, `//` filesystem root, a leading `/` anchors at the working directory. Bash rules match the command as written; pair them with the sandbox for anything security-critical.

### Model and Effort

| Key | Description |
|-----|-------------|
| `model` | Model new sessions start with (alias or full ID) |
| `fallbackModel` | Ordered list of backup models |
| `availableModels` | Models users may select; `deniedModels` **[M]** blocks specific ones |
| `effortLevel` | Default effort: `low`, `medium`, `high`, `xhigh` |
| `maxEffortLevel` | Effort cap; the lowest cap across scopes wins |
| `modelSettings` | Per-model `effortLevel`, `maxEffortLevel`, `autoCompactWindow` |
| `advisorModel` | Model for the advisor tool (unset = off) |
| `fastMode` / `fastModePerSessionOptIn` | Fast mode on by default / require `/fast` each session |
| `alwaysThinkingEnabled` | `false` turns extended thinking off where the model allows it |
| `modelOverrides` | Map model IDs to provider-specific IDs |
| `modelPricing` **[M]** | Report costs at contracted rates (`multiplier`, `overrides`) |
| `language` | Response language |

### Context and Memory

| Key | Description |
|-----|-------------|
| `autoCompactEnabled` / `autoCompactWindow` | Auto-compaction on/off; token window at which it triggers |
| `autoMemoryEnabled` / `autoMemoryDirectory` | Auto memory on/off; where it is stored |
| `claudeMdExcludes` | Glob patterns for CLAUDE.md files to skip (monorepos) |
| `claudeMd` **[M]** | Organization-wide instructions delivered through settings |
| `plansDirectory` | Where plan files are written |
| `fileCheckpointingEnabled` | File checkpoints for `/rewind` |
| `bashOutputMaxChars` | Shell output kept in context |
| `skillOverrides` | Per-skill visibility: hide from the model, from the menu, or both |
| `skillListingBudgetFraction` / `skillListingMaxDescChars` | How much context the skill listing may use |
| `env` | Environment variables for every session (`{"KEY": "value"}`) |

### Agents, Worktrees, and Sessions

| Key | Description |
|-----|-------------|
| `agent` | Run the main thread as a named custom agent |
| `teammateMode` | Agent team display: `in-process` (default), `auto`, or a split-pane mode |
| `worktree.baseRef` | Branch new worktrees from the remote default or local `HEAD` |
| `worktree.sparsePaths` | Sparse checkout paths for new worktrees (large monorepos) |
| `worktree.symlinkDirectories` | Directories to symlink into worktrees instead of copying |
| `worktree.bgIsolation` | Whether background sessions get their own worktree |
| `disableAgentView` | Turn off background sessions and agent view |
| `crossSessionInbound` | Accept, hold, or refuse messages from other sessions |
| `cleanupPeriodDays` | Delete transcripts older than this at startup (default 30) |
| `enableWorkflows` / `disableWorkflows` | Dynamic workflows on or off |

### Sandbox

| Key | Description |
|-----|-------------|
| `sandbox.enabled` | Sandbox shell commands (macOS, Linux, WSL2) |
| `sandbox.failIfUnavailable` | Refuse to start if the sandbox cannot run |
| `sandbox.autoAllowBashIfSandboxed` | Run sandboxed commands without prompting (default true) |
| `sandbox.allowUnsandboxedCommands` | `false` removes the escape hatch: no command runs outside the sandbox |
| `sandbox.excludedCommands` | Commands that always run outside the sandbox |
| `sandbox.filesystem.allowWrite` / `.denyWrite` / `.allowRead` / `.denyRead` | Filesystem boundaries beyond the working directory |
| `sandbox.network.allowedDomains` / `.deniedDomains` | Network allowlist (starts empty) and denylist |
| `sandbox.network.allowUnixSockets` / `.allowAllUnixSockets` | Unix socket access |
| `sandbox.network.allowLocalBinding` | Allow binding to local ports |
| `sandbox.network.httpProxyPort` / `.socksProxyPort` | Use your own filtering proxy |
| `sandbox.credentials` | Mask credentials so sandboxed processes never see the real values |
| `sandbox.enableWeakerNestedSandbox` | Weaker mode for unprivileged containers (reduces security) |
| `sandbox.network.allowManagedDomainsOnly` **[M]** | Only managed network rules apply |

**Example:**

```json
{
  "sandbox": {
    "enabled": true,
    "allowUnsandboxedCommands": false,
    "excludedCommands": ["docker"],
    "network": {
      "allowedDomains": ["registry.example.com", "*.internal.example.com"]
    }
  },
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Edit(./.env*)",
      "Edit(./secrets/**)"
    ]
  }
}
```

The sandbox covers shell commands (and the Monitor tool). Read and Edit deny rules cover the file tools. You need both.

### Hooks and Automation

| Key | Description |
|-----|-------------|
| `hooks` | Event-to-handler configuration (see Hook Schema below) |
| `disableAllHooks` | Turn off all hooks and the custom status line |
| `allowedHttpHookUrls` | URLs HTTP hooks may call |
| `httpHookAllowedEnvVars` | Environment variables HTTP hooks may interpolate into headers |
| `allowManagedHooksOnly` **[M]** | Only managed (and SDK) hooks run |

### MCP

| Key | Description |
|-----|-------------|
| `enableAllProjectMcpServers` | Approve every server in project `.mcp.json` files |
| `enabledMcpjsonServers` / `disabledMcpjsonServers` | Approve or reject specific project servers |
| `allowedMcpServers` / `deniedMcpServers` | Allowlist / denylist by name, command, or URL pattern. Empty allowlist = lockdown; denylist wins |
| `disableClaudeAiConnectors` | Ignore connectors from your Claude account |
| `managedMcpServers` **[M]** | Servers the organization provides |
| `allowManagedMcpServersOnly` **[M]** | Only organization-provided servers may connect |

### Plugins and Skills

| Key | Description |
|-----|-------------|
| `enabledPlugins` | Plugins on for this scope (`"name@marketplace": true`) |
| `extraKnownMarketplaces` | Marketplaces teammates are offered when they trust the folder |
| `pluginConfigs` | Per-plugin user configuration values |
| `disableBundledSkills` | Remove bundled skills and workflows |
| `disableSkillShellExecution` | Block shell commands embedded in skills |
| `strictKnownMarketplaces` **[M]** | Restrict marketplaces to an approved list |
| `blockedMarketplaces` **[M]** | Block specific marketplaces |
| `strictPluginOnlyCustomization` **[M]** | Allow agents, hooks, MCP, or skills only from plugins |

Marketplace sources can be a hosted repository shorthand, any git URL, a URL to a marketplace file, an npm package, or a local file or directory. Plugin scripts reference their own directory as `${CLAUDE_PLUGIN_ROOT}` and persistent data as `${CLAUDE_PLUGIN_DATA}`.

### Git and Attribution

| Key | Description |
|-----|-------------|
| `attribution.commit` | Commit trailer text; `""` omits it |
| `attribution.pr` | Pull request description footer; `""` omits it |
| `attribution.sessionUrl` | Whether to include a link to the session |
| `includeGitInstructions` | Built-in commit and PR instructions on or off |
| `prUrlTemplate` | Pull request URL format for self-hosted version control |
| `includeCoAuthoredBy` | Deprecated; use `attribution` |

### Authentication and Providers

| Key | Description |
|-----|-------------|
| `apiKeyHelper` | Script that prints an API credential (refresh with `CLAUDE_CODE_API_KEY_HELPER_TTL_MS`) |
| `awsAuthRefresh` / `awsCredentialExport` / `gcpAuthRefresh` | Cloud-provider credential scripts |
| `otelHeadersHelper` | Script that prints OpenTelemetry headers |
| `forceLoginMethod` | `claudeai` or `console` |
| `forceLoginOrgUUID` | Organization to select at login |
| `forceLoginGatewayUrl` **[M]** | Send sign-in through your Claude apps gateway |
| `allowedProviders` **[M]** | Restrict which model providers may be used |

### Updates

| Key | Description |
|-----|-------------|
| `autoUpdatesChannel` | `latest` or `stable` |
| `minimumVersion` | Do not auto-update below this version |
| `requiredMinimumVersion` / `requiredMaximumVersion` **[M]** | Refuse to run outside this version range |

### Interface

| Key | Description |
|-----|-------------|
| `statusLine` | Custom status line command |
| `fileSuggestion` | Custom script for `@` file autocomplete (large monorepos) |
| `respectGitignore` | Whether the `@` picker honors `.gitignore` (default true) |
| `outputStyle` | Active output style |
| `tui` | Renderer: `default` or `fullscreen` |
| `editorMode` | `normal` or `vim` |
| `theme`, `spinnerVerbs`, `spinnerTipsEnabled`, `prefersReducedMotion`, `showTurnDuration`, `timeFormat` | Display preferences |
| `companyAnnouncements` | Messages shown at startup |

### Managed-Only Enterprise Controls

| Key | Description |
|-----|-------------|
| `policyHelper` | Script that returns policy dynamically |
| `forceRemoteSettingsRefresh` | Refuse to start until server-managed settings are fetched |
| `disableSideloadFlags` | Block `--plugin-dir`, `--plugin-url`, and similar session-only loading |
| `sshHostAllowlist` | Hosts the desktop app may open SSH sessions to |
| `channelsEnabled` / `allowedChannelPlugins` | Channel notifications and which plugins may send them |

---

## Environment Variables (Selected)

Claude Code reads well over two hundred environment variables. These are the ones experienced users actually set. Any of them can also go in a settings file's `env` block.

### Authentication and Providers

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | API key; overrides subscription login while set |
| `ANTHROPIC_AUTH_TOKEN` | Custom `Authorization: Bearer` value (gateways) |
| `CLAUDE_CODE_OAUTH_TOKEN` | Subscription token for automation (from `claude setup-token`) |
| `ANTHROPIC_BASE_URL` | Custom API endpoint (proxies, LLM gateways) |
| `ANTHROPIC_CUSTOM_HEADERS` | Extra request headers, `Name: Value` per line |
| `CLAUDE_CODE_USE_BEDROCK` / `CLAUDE_CODE_USE_VERTEX` / `CLAUDE_CODE_USE_FOUNDRY` | Route through a cloud provider (`1`) |
| `CLAUDE_CODE_CLIENT_CERT` / `CLAUDE_CODE_CLIENT_KEY` | Mutual TLS |

### Models and Reasoning

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_MODEL` | Session model; beats the `model` setting |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` / `_SONNET_MODEL` / `_HAIKU_MODEL` / `_FABLE_MODEL` | What each alias resolves to (pin versions on cloud providers) |
| `ANTHROPIC_SMALL_FAST_MODEL` | Deprecated; use `ANTHROPIC_DEFAULT_HAIKU_MODEL` |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Model for subagents |
| `CLAUDE_CODE_EFFORT_LEVEL` | Effort level, including `auto`; beats `--effort` and `/effort` |
| `MAX_THINKING_TOKENS` | Fixed thinking budget, for older models that use one |
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Output token cap per request |
| `CLAUDE_CODE_DISABLE_FAST_MODE` | Disable fast mode |
| `CLAUDE_CODE_DISABLE_1M_CONTEXT` | Hold sessions to a 200K window |

### Context and Compaction

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | Auto-compact window in tokens (100K-1M); beats `/autocompact` |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | Trigger compaction at a percentage of the window (can only lower it) |
| `DISABLE_AUTO_COMPACT` / `DISABLE_COMPACT` | Turn off auto-compaction / all compaction |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | `1` disables auto memory |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | `1` skips all CLAUDE.md files |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` | `1` loads CLAUDE.md from `--add-dir` directories |
| `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` | `1` disables code restore in `/rewind` |

### Tools and Execution

| Variable | Description |
|----------|-------------|
| `BASH_DEFAULT_TIMEOUT_MS` / `BASH_MAX_TIMEOUT_MS` | Shell command timeouts (defaults 2 and 10 minutes) |
| `BASH_MAX_OUTPUT_LENGTH` | Shell output characters returned to Claude |
| `CLAUDE_BASH_MAINTAIN_PROJECT_WORKING_DIR` | Return to the project directory after each command |
| `CLAUDE_ENV_FILE` | Script sourced before every shell command (populated by SessionStart and related hooks) |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | Disable background commands, background subagents, and Ctrl+B |
| `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` | Concurrent subagents per session (default 20) |
| `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` | Nesting depth (default 3) |
| `CLAUDE_CODE_ENABLE_TODO_TOOLS` | `1` provides task tools on every model |
| `CLAUDE_CODE_TASK_LIST_ID` | Share a task list across sessions |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | `1` enables agent teams |
| `CLAUDE_CODE_WORKFLOW_MAX_CONCURRENT_AGENTS` | Concurrency for dynamic workflows |
| `CLAUDE_CODE_DISABLE_WORKFLOWS` / `CLAUDE_CODE_DISABLE_CRON` | Turn off workflows / scheduled tasks and `/loop` |

### MCP

| Variable | Description |
|----------|-------------|
| `ENABLE_TOOL_SEARCH` | Control deferred MCP tool loading |
| `MAX_MCP_OUTPUT_TOKENS` | Cap on MCP tool result size (default 25,000 tokens) |
| `MCP_TIMEOUT` / `MCP_TOOL_TIMEOUT` | Server startup / tool call timeouts (milliseconds) |
| `CLAUDE_CODE_MCP_AUTO_BACKGROUND_MS` | Move long MCP calls to the background after this long |

### Non-Interactive and CI

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_SIMPLE` | Same as `--bare` |
| `CLAUDE_CODE_MAX_TURNS` | Turn cap when `--max-turns` is not passed |
| `CLAUDE_CODE_FORWARD_SUBAGENT_TEXT` | Same as `--forward-subagent-text` |
| `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS` | How long `-p` waits for background work (default 10 minutes) |
| `CLAUDE_CODE_RETRY_WATCHDOG` | Wait through long outages instead of failing (unattended runs) |
| `CLAUDE_CODE_RESTRICTED` | Same as `--restricted` |
| `CLAUDECODE` | Set to `1` in Claude Code's subprocesses, so scripts can detect it |
| `CLAUDE_CODE_REMOTE` | `true` inside cloud sessions |

### Network

| Variable | Description |
|----------|-------------|
| `HTTPS_PROXY` / `HTTP_PROXY` / `NO_PROXY` | Corporate proxy configuration |
| `CLAUDE_CODE_CERT_STORE` | CA sources: `bundled`, `system`, or both |
| `API_TIMEOUT_MS` | API request timeout (default 10 minutes) |
| `CLAUDE_CODE_MAX_RETRIES` | API retry count |

### Telemetry and Privacy

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_ENABLE_TELEMETRY` | `1` enables OpenTelemetry export (configure with standard `OTEL_*` variables) |
| `OTEL_LOG_USER_PROMPTS` / `OTEL_LOG_TOOL_DETAILS` | Include prompt text / tool details in telemetry |
| `DISABLE_TELEMETRY` / `DISABLE_ERROR_REPORTING` | Opt out of usage telemetry / error reports (any value opts out, even `0`) |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | Disable updates, telemetry, and feature-flag fetching |
| `DISABLE_AUTOUPDATER` / `DISABLE_UPDATES` | Disable background updates / all updates |

### Debugging

| Variable | Description |
|----------|-------------|
| `CLAUDE_CODE_DEBUG_LOGS_DIR` | Debug log location (debug mode must be on) |
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | `verbose`, `debug`, `info`, `warn`, `error` |
| `CLAUDE_CODE_SAFE_MODE` | Same as `--safe-mode` |
| `CLAUDE_CONFIG_DIR` | Move `~/.claude` elsewhere (settings, history, plugins) |

---

## Tools Inventory

| Tool | Description | Needs permission |
|------|-------------|------------------|
| `Read` | Read files, images, PDFs, notebooks | No |
| `Write` | Create or overwrite files | Yes |
| `Edit` | Targeted string replacement | Yes |
| `NotebookEdit` | Edit notebook cells | Yes |
| `Bash` / `PowerShell` | Run shell commands | Yes |
| `Monitor` | Background command whose output lines stream to Claude | Yes |
| `Glob` / `Grep` | File and content search (absent by default on Linux, macOS, WSL) | No |
| `LSP` | Definitions, references, diagnostics via language-server plugins | No |
| `WebFetch` / `WebSearch` | Web access | Yes |
| `Agent` | Spawn a subagent (formerly `Task`) | No |
| `SendMessage` / `ListAgents` | Message teammates, subagents, or other sessions | No |
| `Skill` | Run a skill in the main conversation | Yes |
| `Workflow` | Run a dynamic workflow | Yes |
| `TaskCreate` / `TaskGet` / `TaskList` / `TaskUpdate` | Task checklist (default only on some models) | No |
| `TaskStop` | Stop a background task or agent | No |
| `TodoWrite` | Legacy checklist (`CLAUDE_CODE_ENABLE_TASKS=0`) | No |
| `CronCreate` / `CronList` / `CronDelete` / `ScheduleWakeup` | Session-scoped scheduling (`/loop`) | No |
| `RemoteTrigger` | Cloud scheduled tasks (`/schedule`) | Yes |
| `AskUserQuestion` | Multiple-choice questions | No |
| `EnterPlanMode` / `ExitPlanMode` | Plan mode transitions | No / Yes |
| `EnterWorktree` / `ExitWorktree` | Move the session into or out of a worktree | No |
| `ToolSearch` | Load deferred tools on demand | No |
| `ListMcpResourcesTool` / `ReadMcpResourceTool` | MCP resources | No |
| `PushNotification` | Desktop and phone notifications | No |
| `Artifact` | Publish a page to the Claude apps | Yes |

Removed since the first edition: `TaskOutput` (Claude reads the output file with `Read`), `TodoRead`, `TeamCreate`, `TeamDelete` (agent teams are created by asking).

---

## Hook Configuration Schema

### Structure

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validate.sh",
            "if": "Bash(git push *)",
            "timeout": 30,
            "onFailure": "block"
          }
        ]
      }
    ]
  }
}
```

Event name → list of matcher groups → list of handlers.

### Handler Types

| Type | Key fields | Default timeout |
|------|-----------|-----------------|
| `command` | `command`, `args` (exec form, no shell), `shell` | 600s |
| `http` | `url`, `headers`, `allowedEnvVars` | 600s |
| `mcp_tool` | `server`, `tool`, `input` | 600s |
| `prompt` | `prompt` (`$ARGUMENTS` = event JSON), `model` | 30s |
| `agent` | `prompt` (experimental; multi-turn with tools) | 60s |

Common fields: `if` (one permission-rule pattern), `timeout` (seconds), `statusMessage`, `once` (run once per session), `async` (background, cannot block), `asyncRewake` (background; exit 2 wakes Claude), `onFailure` (`"continue"` default, or `"block"` to fail closed).

Prompt and agent handlers return `{"ok": true}` or `{"ok": false, "reason": "..."}`.

### Events

| Can block | Events |
|-----------|--------|
| Yes | `PreToolUse`, `PostToolBatch`, `UserPromptSubmit`, `UserPromptExpansion`, `Stop`, `SubagentStop`, `TeammateIdle`, `TaskCreated`, `TaskCompleted`, `ConfigChange`, `PreCompact`, `PreModelSwitch`, `Elicitation`, `ElicitationResult`, `WorktreeCreate`, `WorktreeRemove` |
| No | `SessionStart`, `Setup`, `SessionEnd`, `PermissionRequest`*, `PermissionDenied`, `PostToolUse`**, `PostToolUseFailure`, `Notification`, `MessageDisplay`, `SubagentStart`, `StopFailure`, `CwdChanged`, `DirectoryAdded`, `FileChanged`, `InstructionsLoaded`, `PostCompact`, `PostModelSwitch` |

\* PermissionRequest ignores exit 2; deny with its JSON `decision` instead.
\** PostToolUse cannot un-run the tool, but `decision: "block"` feeds a reason back to Claude and `updatedToolOutput` replaces the result.

### Matchers

- `"*"`, `""`, or omitted: everything
- Plain names and lists: `Edit|Write` or `Edit,Write`
- Anything else is an unanchored regex: `^Bash$`, `mcp__memory__.*`
- MCP tools: `mcp__<server>__<tool>`; plugin servers: `mcp__plugin_<plugin>_<server>__<tool>`
- SessionStart matches its source: `startup`, `resume`, `clear`, `compact`, `fork`

### Input (stdin JSON)

Common: `session_id`, `prompt_id`, `transcript_path`, `cwd`, `permission_mode`, `effort`, `hook_event_name`, plus `agent_id`/`agent_type` inside subagents. Tool events add `tool_name`, `tool_input`, `tool_use_id`.

### Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success. A single JSON object on stdout is applied |
| `2` | Block. Stderr goes to Claude as the reason; JSON is ignored |
| Other (including `1`) | Non-blocking error; the action proceeds unless `onFailure: "block"` |

### JSON Output

| Field | Effect |
|-------|--------|
| `continue: false` | Stop Claude entirely |
| `stopReason` | Message shown to the user when stopping |
| `systemMessage` | Warning shown to the user |
| `decision: "block"` + `reason` | Block for UserPromptSubmit, PostToolUse, Stop, SubagentStop, ConfigChange |
| `hookSpecificOutput.hookEventName` | Required inside `hookSpecificOutput` |
| `hookSpecificOutput.permissionDecision` | PreToolUse: `allow`, `deny`, `ask`, `defer` |
| `hookSpecificOutput.permissionDecisionReason` | Reason shown with the decision |
| `hookSpecificOutput.updatedInput` | PreToolUse: replace tool arguments |
| `hookSpecificOutput.additionalContext` | Context for Claude (capped at 10,000 characters) |
| `hookSpecificOutput.updatedToolOutput` | PostToolUse: replace the tool result |
| `hookSpecificOutput.decision` | PermissionRequest: `{"behavior": "allow" \| "deny", "updatedInput": {...}}` |
| `retry: true` | PermissionDenied: Claude may try another approach |

### Environment Variables in Hooks

| Variable | Description |
|----------|-------------|
| `CLAUDE_PROJECT_DIR` | Project root. Quote it: `"$CLAUDE_PROJECT_DIR"` |
| `CLAUDE_PLUGIN_ROOT` / `CLAUDE_PLUGIN_DATA` | Plugin install directory / persistent data directory |
| `CLAUDE_ENV_FILE` | SessionStart, Setup, CwdChanged, FileChanged: append `export` lines to set variables for later shell commands |
| `CLAUDE_EFFORT` | Current effort level |
| `CLAUDE_CODE_REMOTE` | `true` in cloud sessions |

```bash
#!/bin/bash
if [ -n "$CLAUDE_ENV_FILE" ]; then
  echo 'export NODE_ENV=production' >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

### Trust

Hooks from a repository's settings wait for the workspace trust dialog in interactive sessions. `-p` and SDK runs treat the folder as trusted; use `--bare` or `disableAllHooks` when scripting over code you did not write. `/hooks` shows what is configured and where it came from.

---

## MCP Configuration Format

### .mcp.json (Project Scope)

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
      "url": "https://mcp.example.com/mcp",
      "headers": { "Authorization": "Bearer ${TRACKER_TOKEN}" }
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `type` | `stdio` for local processes, `http` for remote servers (`sse` is deprecated) |
| `command` / `args` / `env` | Local process; `env` supports `${VAR}` and `${VAR:-default}` |
| `url` / `headers` | Remote endpoint |
| `headersHelper` | Command that prints headers at connect time (short-lived tokens) |

Scopes: local (default, stored in `~/.claude.json` for this project), project (`.mcp.json`), user (`~/.claude.json`, all projects). Same name in several scopes: local beats project beats user, which beats plugin servers and account connectors; organization-provided servers rank above all.

---

## Full Settings.json Example

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "model": "opus",
  "effortLevel": "high",
  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": [
      "Bash(npm run test *)",
      "Bash(npm run lint)",
      "Bash(git diff *)",
      "Bash(git log *)"
    ],
    "ask": [
      "Bash(git push *)"
    ],
    "deny": [
      "Read(./.env*)",
      "Read(./secrets/**)",
      "Edit(./.env*)"
    ]
  },
  "env": {
    "NODE_ENV": "development"
  },
  "sandbox": {
    "enabled": true,
    "network": {
      "allowedDomains": ["registry.example.com"]
    }
  },
  "claudeMdExcludes": ["**/vendor/**/CLAUDE.md"],
  "worktree": {
    "symlinkDirectories": ["node_modules"]
  },
  "attribution": {
    "pr": ""
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/lint.sh"
          }
        ]
      }
    ]
  }
}
```
