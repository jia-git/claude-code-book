# Chapter 4: Multi-Agent Orchestration

## What You'll Learn

The moment your task outgrows a single context window, you have two choices: fight the constraint or distribute the work. Multi-agent orchestration is how you distribute the work.

When the first edition of this book was written, there were two ways to do that: subagents inside a session, and an experimental agent-teams system. There are now five, and they differ along two axes that matter more than any feature list -- who holds the plan, and how involved you stay. Subagents are workers Claude spawns and supervises turn by turn, now running in the background by default, able to inherit the whole conversation as forks, and allowed to spawn their own subagents. Background sessions are full conversations you dispatch and monitor from a single screen, each in its own git worktree. Agent teams are a lead session supervising peer sessions that message each other -- still experimental, and simpler than they were. Dynamic workflows move the plan out of Claude's head and into a script, orchestrating dozens to hundreds of agents with results cross-checked before you see them. And projects in the cloud keep a stream of related work running for days while your laptop is closed.

This chapter covers all of them, with the most depth on the ones you will use daily. You will learn the built-in subagent types and every field that shapes a custom one, how background execution and forks change the economics of delegation, how agent view turns parallel sessions into a dispatch board, how a workflow script differs from a team of agents, and why sessions you run yourself can now pass findings to each other. You will see why a practitioner's four-agent trading hierarchy degraded to a single agent because simplicity won, and why a QA swarm of five parallel agents audited an entire blog in three minutes. And you will learn why the cheapest orchestration strategy is almost always the correct one.

---

## The Architecture of Subagents

A subagent is a Claude Code session within a session. It gets its own context window, its own system prompt, and a restricted set of tools. When it finishes, it returns a summary to whoever spawned it. Then its context window is set aside.

The first edition described a hard rule: subagents cannot spawn other subagents, and that strict two-level hierarchy was "a deliberate design decision, not a limitation." It was both. The rule is gone. Subagents can now spawn their own subagents, three layers deep by default (`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` changes the limit; `1` restores the old behavior). At the limit, the Agent tool is simply withheld. The original warning still stands, though, and it is worth repeating because the guardrail that enforced it no longer does: recursive agent hierarchies sound elegant and frequently produce chaos. Nesting is for a worker that genuinely needs its own helpers -- a reviewer that fans out one checker per file -- not for org charts.

A note on naming before going further. The tool Claude uses to spawn subagents was renamed from `Task` to `Agent`. `Task(...)` still works as an alias in permission rules and older configuration, but new configuration should say `Agent`.

The built-in subagent types:

**Explore** is read-only -- it can search files, read code, and navigate the codebase, but it cannot modify anything. It takes a thoroughness level (quick, medium, or very thorough) and skips CLAUDE.md and git status to start fast. One change matters for your budget: Explore no longer defaults to a small, cheap model. It runs on the capable model -- the main model, or the `opus` alias on direct subscriptions -- because a careless reconnaissance pass turned out to be expensive in a different way. If you want cheap exploration, define your own explorer with `model: haiku`.

**Plan** inherits the main session's model. It is also read-only, and it does the research during plan mode. Use it when the planning itself is complex enough to warrant dedicated context.

**General-purpose** has access to all tools. It can read, write, execute commands, and perform multi-step tasks. This is your workhorse for implementation.

**claude** is the catch-all, and the default agent for sessions you dispatch to the background.

**claude-code-guide** answers questions about Claude Code itself, consulting the product's own documentation. **statusline-setup** handles the narrow task of configuring your status line.

Explore and Plan are one-shot: they cannot be resumed. If you would rather Claude never use them, deny `Agent(Explore)` in your permissions, or set `CLAUDE_CODE_DISABLE_EXPLORE_PLAN_AGENTS=1`.

You can also define custom subagents as Markdown files with YAML frontmatter. Definitions live in `.claude/agents/` for project scope or `~/.claude/agents/` for user scope; managed settings, the `--agents` flag, and plugins supply them too. When the same name appears in several places, managed wins, then the CLI flag, then project, then user, then plugin. Directories are scanned recursively, and edits are picked up within seconds. The `/agents` command used to open a wizard for this; it now just reminds you where the files live. The faster path is to describe the agent you want and let Claude write the file.

### Custom Subagent Definition Fields

Only `name` and `description` are required. The description is how Claude decides when to delegate, so lead with the use case. Beyond the system prompt in the body, the fields that shape behavior most:

**`tools` and `disallowedTools`** set an allowlist and a denylist. Omit `tools` and the agent inherits everything. A denylist entry with a specifier, such as `Bash(git push *)`, removes the whole tool -- denylists are blunt instruments.

**`model`** takes `sonnet`, `opus`, `haiku`, `fable`, a full model ID, or `inherit`. Resolution order: a model named on the individual invocation, then this field, then `CLAUDE_CODE_SUBAGENT_MODEL`, then the main conversation's model. **`effort`** sets the subagent's own effort level, which is often the bigger lever.

**`maxTurns`** limits the number of agentic turns before a subagent stops. This is a safety valve. A subagent exploring a codebase might spiral into increasingly tangential searches. Setting `maxTurns: 20` caps the exploration; the output is marked partial and can be resumed. Without it, a subagent on a dead-end path burns tokens until something else stops it.

**`mcpServers`** configures which MCP servers are available to a specific subagent. You can reference servers by name or inline a full server configuration. A database-reader subagent can have the database server while a code-reviewer subagent has none. Tool access scoped to the role, not the session.

**`skills`** preloads skill content into the subagent's context at launch. In the main session, skills load on demand. Subagents work differently: listed skills are fully injected at startup because the subagent needs to act immediately. Every skill you preload costs context from turn one.

**`isolation: worktree`** runs the subagent in a temporary git worktree branched from the default branch, so its edits cannot collide with yours. The worktree is cleaned up if the subagent made no changes.

**`background: true`** keeps the subagent in the background even when Claude would otherwise wait for it. **`permissionMode`**, **`hooks`**, **`memory`** (covered later), **`omitClaudeMd`** (skip the user and project CLAUDE.md for a clean-room reviewer), **`autoCompactWindow`**, **`color`**, and **`initialPrompt`** (an auto-submitted first turn when the agent runs as the main session) round out the set. Unknown fields are silently ignored, which means a typo fails quietly. Check spelling.

### The `--agents` CLI Flag

For quick testing or CI automation, you can pass subagent definitions as JSON directly when launching Claude Code instead of writing Markdown files:

```bash
claude --agents '{
  "code-reviewer": {
    "description": "Reviews code for style and correctness",
    "prompt": "You are a code reviewer. Check for bugs, style violations, and security issues.",
    "tools": ["Read", "Bash"],
    "model": "sonnet"
  },
  "debugger": {
    "description": "Debugs failing tests",
    "prompt": "You are a debugging specialist. Analyze test failures and suggest fixes.",
    "tools": ["Read", "Edit", "Bash"],
    "model": "sonnet",
    "maxTurns": 30
  }
}'
```

The flag accepts the same fields as file-based definitions, and Claude Code validates the JSON at startup instead of failing halfway through a run. In print mode the value can also be a path to a JSON file, which is far easier to maintain in a pipeline than an inline string. This is particularly useful in headless pipelines where you want specialized agents without committing Markdown files to the repository.

### The `--agent` Flag

A related but distinct flag: `--agent` runs a custom agent definition as the main thread rather than as a subagent. If you have a `code-reviewer` agent defined in `.claude/agents/code-reviewer.md`, running `claude --agent code-reviewer` starts the session using that agent's system prompt, tools, and configuration. The same can be set per project with `"agent": "code-reviewer"` in settings. You can restrict which subagent types this top-level agent can spawn by listing them in its tools -- `tools: Agent(worker, researcher), Read` -- creating controlled hierarchies where a review agent can only delegate to the helpers you chose.

## Context Isolation Is the Point

The most important thing subagents give you is not parallelism. It is context isolation.

When you ask the main session to read a large file, parse verbose test output, or explore a sprawling directory tree, all of that content accumulates in the main context window. Do this enough times and you hit the ceiling described in Chapter 1 -- instructions get forgotten, work gets dropped, quality degrades.

Subagents solve this by containing the mess. A subagent can read fifty files, parse a thousand lines of test output, explore every directory in the repo, and the main session only sees the summary. The verbose intermediate work stays in the subagent's context.

This is why experienced users route expensive context operations through subagents even when they do not need parallelism. Reading a large codebase module? Subagent. Running a comprehensive test suite and analyzing failures? Subagent. Searching for usage patterns across hundreds of files? Subagent. The goal is keeping the orchestrator's context lean so it can coordinate effectively across dozens of tasks without losing coherence.

What a fresh subagent starts with is precise: its own system prompt, the delegation message, CLAUDE.md (unless skipped), a git status snapshot, any preloaded skills, and -- when messaging is available -- a roster of its siblings. It does not inherit your conversation, your output style, or your auto memory. It does inherit your extended-thinking configuration.

The tradeoff is that summaries are lossy. A subagent that returns a detailed result consumes main context proportional to the length of that result. If you spin up ten subagents and each returns a page of findings, you have ten pages of results in your main window. The solution is to instruct subagents to be concise in their returns -- specific answers, not comprehensive reports.

## Forks: Subagents That Remember

Context isolation has a cost the first edition glossed over: a fresh subagent knows nothing. Every delegation message has to re-explain the project, the decisions made so far, and the thing you are actually trying to do. For a quick side task, writing that briefing can take longer than doing the task.

A *forked* subagent inverts the tradeoff. Instead of starting fresh, it inherits the full conversation -- system prompt, tools, history -- and goes off to work on one thing in the background while you keep going. Its result returns to your conversation when it finishes. Type `/subtask` followed by the task to spawn one yourself. Fork mode is also on by default, which means Claude itself will fork when a side task needs the context you have built up -- checking whether a refactor broke a distant caller, say, while it keeps implementing.

Forks are cheaper than they look. Because a fork starts with exactly the same prefix as your conversation, its first request reads your conversation's prompt cache instead of paying to process the history again. The isolation property is preserved where it matters: whatever the fork reads and runs afterward stays in its context, not yours.

Use a fresh subagent when the worker should not be biased by your conversation -- a reviewer grading code it did not watch being written. Use a fork when the worker needs to know what you know. (`CLAUDE_CODE_FORK_SUBAGENT=0` turns fork mode off.)

## Foreground vs. Background Execution

Subagents can run in the foreground or the background, and the default flipped since the first edition.

**Background is now the default** for subagents Claude spawns from the main conversation. Claude keeps working while they run. Up to twenty can run at once per session (`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` adjusts that). The old restrictions are mostly gone too: background subagents keep their MCP tools, and when one needs a permission it was not pre-granted, the prompt now appears in your main session labeled with the subagent's name instead of being silently auto-denied. Background subagents get a slightly reduced set of built-in tools, and they cannot ask you clarifying questions.

**Foreground subagents** block the main conversation until they finish. Use foreground when the subagent's result determines the next step -- set `background: false` on a forked skill, or have Claude wait. Setting `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` forces everything into the foreground, along with every other background feature.

The controls are worth memorizing. **Ctrl+B** moves a running foreground task to the background. `/tasks` lists everything running in the background of the session -- subagents, shell commands, workflows -- and lets you check on, attach to, or stop each one. Named background subagents show up in the `@`-mention typeahead with their status. And **Ctrl+X Ctrl+K**, pressed twice, stops every running background subagent at once -- the emergency brake when a fan-out goes wrong.

The practical pattern has not changed: launch three background subagents to explore three modules, keep working on something else in the main session, then review their results when they complete. What changed is that this is now what happens when you do nothing special.

### Resuming Subagents

Every completed subagent returns an ID, and Claude can resume it by sending it a message -- addressed by ID or by the name Claude gave it. A resumed subagent keeps its full history, so a reviewer that spent twenty minutes analyzing an authorization module can answer a follow-up question without redoing the analysis.

Subagent transcripts persist independently of the main conversation, stored as separate files under the session's directory in your Claude Code project data. When the main conversation compacts, subagent transcripts are unaffected, and they survive a restart if you resume the session. They are cleaned up after thirty days by default.

### Subagent Auto-Compaction

Subagents support automatic compaction using the same logic as the main conversation. For subagents that handle large volumes of data -- parsing thousands of lines of test output, reading dozens of files -- this means they can work through content that exceeds their context window by periodically summarizing and continuing.

The `autoCompactWindow` field in a subagent's definition sets its own compaction threshold, from 100K tokens up. It can lower the session-wide value but not raise it. Earlier compaction reduces peak context consumption at the cost of losing detail from earlier turns. For subagents doing broad exploration, earlier compaction is usually fine. For subagents doing precise analysis where every detail matters, let them run closer to capacity.

## Subagent Hooks

Custom subagents support lifecycle hooks that run at specific points during execution. These are defined in the agent's YAML frontmatter and work with the same event model as session-level hooks (Chapter 2), but scoped to the subagent. Project-level agent files need workspace trust before their hooks run.

Three hook events fire within the subagent's execution: **PreToolUse** runs before the subagent uses a tool and can block or modify the invocation. **PostToolUse** runs after a tool completes and can transform output or trigger side effects. **Stop** runs when the subagent finishes -- but at runtime, it is automatically converted to a **SubagentStop** event.

Two additional hook events fire at the project level, outside the subagent's own context: **SubagentStart** fires when any subagent spawns, and **SubagentStop** fires when any subagent finishes. These are defined in your project or user settings, matched by agent name. A SubagentStart hook can inject `additionalContext` into the subagent -- instructions or data that supplement its system prompt without modifying the agent definition. A SubagentStop hook can block the subagent from stopping (exit code 2), useful for enforcing that a subagent must produce certain artifacts before it is allowed to finish.

A practical example: a database-reader subagent with a PreToolUse hook on the Bash tool that checks whether commands contain SQL write operations (INSERT, UPDATE, DELETE, DROP). If the hook detects a write statement, it returns a deny decision, preventing the subagent from modifying production data regardless of what its instructions say. This is defense in depth -- the subagent's system prompt says "read only," and the hook enforces it mechanically.

## Subagent Patterns

Four patterns recur in production subagent usage. They are worth naming because they solve specific orchestration problems.

**Chain subagents** run in sequence, where each subagent completes its task and returns results to the orchestrator, which then passes relevant information to the next subagent. Subagent A analyzes the authentication module and returns a list of vulnerabilities. The orchestrator passes that list to Subagent B, which generates fixes. Subagent B's output goes to Subagent C, which writes tests. Each subagent has clean context with only the information it needs. The chain structure prevents context accumulation while maintaining information flow.

**Isolate high-volume operations** by routing anything that produces large output to a subagent. Running a comprehensive test suite might generate thousands of lines of output. Fetching documentation might return hundreds of pages. Processing log files might involve scanning megabytes of text. If this happens in the main session, it crowds out everything else. A subagent absorbs the volume and returns a concise summary: "14 tests failed, all in the auth module, all related to session expiry."

**Parallel research** spawns multiple subagents simultaneously to investigate different aspects of a problem. Three subagents explore the authentication module, the database layer, and the API endpoints in parallel. Each returns findings. The orchestrator synthesizes them into a unified understanding. This is faster than sequential exploration and keeps the orchestrator's context free of the intermediate exploration steps.

**Adversarial review** spawns a fresh subagent to check work the main session just did. Because it sees only the diff and the criteria you give it -- not the reasoning that produced the change -- it evaluates the result on its own terms. The bundled `/code-review` command packages exactly this pattern, running in the background and reporting findings back to the session (Chapter 6). Tell a reviewer to flag only issues that affect correctness or the stated requirements: a reviewer told to find gaps will always find some, and chasing every one produces over-engineering.

## Background Sessions and Agent View

Subagents are workers inside your session. Sometimes you want the opposite: several whole sessions, each working on its own task, while you check in only when one needs you. That used to mean a wall of terminal tabs. It now has a dedicated surface.

`claude agents` opens **agent view**, a single screen listing every background session, grouped by state: working, needs input, idle, completed, failed, stopped. Type a prompt and press Enter to dispatch a new session; each prompt starts its own. From the shell, `claude --bg "investigate the flaky test"` does the same and returns immediately. From inside a session, `/background` (or pressing the left arrow on an empty prompt) moves the current conversation into the background and frees your terminal, and `/fork` copies the conversation into a new background session while you keep working on the original.

The interaction model is built around not attaching. Press Space on a row to peek at the latest output or the pending question and reply right there. Press Enter to attach for the full conversation; detaching never stops a session. Rows linked to a pull request show its number, colored by status. Sessions keep running after you close agent view or the terminal, because a small supervisor process hosts them.

Two details make this safe to use at scale. First, a dispatched session moves into its own git worktree before it edits anything, so ten parallel sessions do not write into the same checkout. Second, dispatched sessions take their permission mode from the target directory's settings -- and refuse overly permissive project settings -- with defaults you can set at launch: `claude agents --permission-mode plan --model opus --effort high`.

Agent view is a research preview, and each background session draws on your plan quota independently. Dispatch ten and you will feel it. But for the work pattern of "hand off five independent tasks, glance at the board between meetings, step in on the one that is stuck," nothing else in the toolbox fits as well. `claude agents --json` exposes the same state to scripts.

## Cross-Session Messaging

The sessions you run yourself used to be islands. If one discovered that the shared type the other was building on had changed, you were the messenger. Now Claude can list your other Claude Code sessions -- on this machine, on another machine, or in the cloud -- and send them a message. `/list-agents` shows who is reachable; typing `@` in the prompt mentions another session by name.

Two safety rules make this tolerable. A session treats an incoming message as coming from another Claude, not from you, so a message cannot approve a permission prompt or supply consent on your behalf -- one session cannot launder an action another was denied. And you can hold inbound messages instead of having them delivered as new turns, which matters because each delivered message re-sends the receiving session's full context.

This is the lightweight middle ground between independent sessions and a full agent team: no lead, no shared task list, just a way for parallel work to pass a finding along.

## Agent Teams: Still Experimental, Now Simpler

Agent teams are a different architecture entirely. Where subagents are workers within a session, agent teams are independent Claude Code sessions -- a lead and its teammates -- that coordinate through a shared task list and direct messages. They remain experimental and disabled by default.

### Enabling Agent Teams

Enable them by setting the `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` environment variable to `1`, either in your shell or through settings:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Then describe what you want in natural language: "Spawn three teammates to explore this from different angles: one on UX, one on architecture, one playing devil's advocate." Teams only form in interactive sessions; in `-p` runs and the SDK, Claude spawns ordinary subagents instead.

One side effect catches people out. While teams are enabled, any subagent Claude *names* when it spawns it launches as a teammate. Claude names subagents on its own so it can message them later, so teams can form when you never asked for one. If that is not what you want, set the variable back to `0` -- the next spawn picks it up without a restart.

### How Teams Work Now

The first edition described seven primitives -- create the team, create tasks, spawn, message, broadcast, delete the team. Much of that ceremony is gone. The team for a session is set up automatically when the session starts, under a session-derived name; there is exactly one per session, and its configuration is cleaned up when the session ends. Teammates are simply subagents that Claude spawns with a name while teams are enabled. What remains is the part that made teams interesting:

**A shared task list.** The lead creates tasks; teammates claim them, either by assignment or by picking up the next unassigned, unblocked task on their own. Tasks have three states -- pending, in progress, completed -- and can depend on each other. A pending task with unresolved dependencies cannot be claimed, and completing a task unblocks its dependents automatically. Claiming uses file locking, so two teammates cannot grab the same task.

**Direct messaging.** Any teammate can message any other by name. Messages are delivered automatically; nobody polls. When a teammate finishes, it notifies the lead with its final answer. There is no broadcast: to reach everyone, send one message per recipient, which keeps the cost of chattiness visible.

**Graceful shutdown.** Ask the lead to shut a teammate down by name; the teammate can approve and exit or decline with an explanation.

Each teammate is a full Claude Code session with its own context window. Teammates load the same project context as any session -- CLAUDE.md, MCP servers, skills -- plus the spawn prompt from the lead. They do not inherit the lead's conversation history, so put the task-specific details in the spawn prompt. You can name a reusable subagent definition when spawning ("spawn a teammate using the security-reviewer agent type"), and the teammate picks up that definition's tools, model, and instructions.

### Display Modes

Teammates display **in-process** by default: they run inside your main terminal, listed in an agent panel below the prompt. Use the arrow keys to select a teammate, Enter to view its transcript and message it directly, Escape to interrupt its turn, `x` to stop it, and Ctrl+T to toggle the task list. This works in any terminal.

Split panes give each teammate its own pane so you can watch everyone at once. Set `teammateMode` to `auto` to use split panes when you are already inside a terminal multiplexer session, or name the multiplexer explicitly. Split panes do not work in a code editor's integrated terminal or some standalone terminals; the in-process default does.

### Plan Approval

For risky work, put the lead in plan mode before asking for a teammate. A teammate spawned while the lead is planning works read-only until its plan is ready, then sends a plan approval request. The lead's session approves the request automatically -- the teammate exits plan mode and begins implementation -- and the teammate's edits and commands still go through the normal permission prompts in the lead session. So the value of plan approval is now mostly sequencing: the plan exists, in writing, before any file changes. If you want a human gate, read the plan before the teammate's first permission prompt arrives.

### Team Hook Events

Three hook events are the natural quality gates for teams:

**TeammateIdle** fires when a teammate is about to go idle. Exit code 2 sends feedback and keeps it working. A practical use: check that build artifacts exist in the expected output directory before allowing the teammate to stop.

**TaskCreated** fires when a task is being created. Exit code 2 prevents it -- useful for enforcing task-size or naming conventions.

**TaskCompleted** fires when a task is being marked complete. Exit code 2 blocks the completion and feeds the reason back. This enables the most useful gate of all: run the test suite when a task is marked complete, and refuse the completion if tests fail.

### Agent Teams vs. Subagents: Detailed Comparison

| | Subagents | Agent Teams |
|---|---|---|
| **Context** | Own window; results return to the caller (forks inherit the conversation) | Own window; fully independent |
| **Communication** | Return results to the caller; named subagents can message each other | Teammates message each other directly |
| **Coordination** | Main agent manages all work | Shared task list with self-claim, plus messages |
| **Token cost** | Lower: results summarized back to main | Higher: each teammate is a full session |
| **Best for** | Focused tasks where only the result matters | Complex work requiring discussion and collaboration |

The distinction matters most when agents need to talk to each other. An API teammate that finishes type definitions and directly messages the UI teammate to start integration work -- that is an agent team pattern. If each worker just reports back to the orchestrator, subagents are cheaper and simpler.

### Limitations

Agent teams are experimental and come with real constraints:

- **No resumption of in-process teammates.** `/resume` and `/rewind` do not restore them; after resuming, tell the lead to spawn new teammates.
- **Task status can lag** -- teammates sometimes forget to mark tasks complete, which can block dependent tasks. Nudge them, or update the task yourself.
- **One team per session, and no nested teams.** Teammates cannot spawn teammates, and in-process teammates cannot run background subagents of their own.
- **The lead is fixed.** You cannot promote a teammate or transfer leadership.
- **Teammates do not isolate in worktrees.** Partition the work so each owns different files; two teammates editing one file will overwrite each other.
- **Permissions are set at spawn.** Teammates start with the lead's permission mode (except don't-ask mode, which they do not inherit). You can change an individual teammate afterward, but not at spawn time.
- **Cost.** Each teammate is a full session; in plan mode, a team can use roughly seven times the tokens of a single session. Start with three to five teammates and five or six tasks each.

## Dynamic Workflows: When the Plan Becomes Code

Every orchestration approach so far has one thing in common: Claude is the orchestrator. It decides, turn by turn, what to spawn next, and every result lands in a context window. That works for a handful of workers. It breaks down at fifty, and it cannot be rerun.

A dynamic workflow moves the plan out of Claude's head and into a script. You describe the task; Claude writes a JavaScript program that orchestrates subagents; a runtime executes it in the background while your session stays responsive. The script holds the loops, the branching, and the intermediate results in variables, so Claude's context receives only the final answer. A saved script looks like this:

```javascript
export const meta = {
  name: 'audit-routes',
  description: 'Audit every route handler for missing auth checks',
}

const found = await agent('List every .ts file under src/routes/.', {
  schema: { type: 'object', required: ['files'],
            properties: { files: { type: 'array', items: { type: 'string' } } } },
})

const audits = await pipeline(found.files, file =>
  agent(`Audit ${file} for missing authentication checks.`, { label: file }),
)

return audits.filter(Boolean)
```

`agent()` spawns one subagent, optionally constrained to return JSON matching a schema. `pipeline()` runs one per item; `parallel()` runs a set at once; `phase()` groups agents under a title in the progress view.

The difference is not just scale. Because the orchestration is code, a workflow can apply a quality pattern a single pass cannot: have independent agents adversarially verify each other's findings before anything is reported, or draft a plan from several angles and weigh them. The bundled `/deep-research` workflow does exactly this for questions that need many sources -- fan out searches, cross-check claims, vote, and filter out what did not survive.

**Starting one.** Ask for it in plain words ("use a workflow to migrate every component under src/components/ to TypeScript, each file in its own isolated copy"), or include the keyword `ultracode` in your prompt. To have Claude plan a workflow for every substantive task in the session, run `/effort ultracode` -- thorough, and correspondingly hungry for tokens. Claude shows the planned phases and asks before the first run, depending on your permission mode.

**Watching and steering.** `/workflows` opens a progress view: phases with agent counts, drill-down into any agent's prompt, tool calls, and result, and keys to pause, restart an agent, or stop the run. A stopped or paused run is resumable in the same session; completed agents return their saved results instead of running again.

**Keeping it.** Press `s` on a run you liked to save its script as a command -- in `.claude/workflows/` for the team, or your home directory for yourself. Saved workflows accept arguments, and plugins can ship them.

**Bounds.** Sixteen agents run concurrently by default, a single run is capped at a thousand agents, and a `Large workflow` warning appears past twenty-five agents or a projected million and a half tokens. A size guideline in `/config` (small, medium, large) tells Claude how many agents to aim for. Test on one directory before you point a workflow at the whole repository.

The rule of thumb: subagents when you want a few delegated tasks per turn; a team when long-running peers need to talk; a workflow when the job outgrows a handful of agents or you want the findings cross-checked before you see them.

## Choosing an Approach

The decision maps to three questions. Who should hold the plan -- Claude turn by turn (subagents), a lead session (teams), a script (workflows), or you (background sessions)? Do the workers need to talk to each other? And do they edit the same files?

**Subagents** are right when the work decomposes into independent tasks that each fit in one context, and you need results quickly and cheaply. A typical subagent workflow takes minutes and costs a fraction of a main session.

**Background sessions** are right when you have several unrelated tasks and want to hand them off and check back -- the work of a dispatcher, not an orchestrator.

**Teams** are right when the work requires sustained coordination between specialists who need to react to each other's output.

**Workflows** are right when the job is too large for turn-by-turn coordination, when you want verification baked in, or when you will run the same orchestration again.

**Neither** is right when a single well-prompted session can handle the task. This is more common than you think. Before reaching for multi-agent orchestration, ask whether the task genuinely exceeds what one context window can handle. If the answer is no, a single agent with clear instructions will outperform a multi-agent setup every time. The coordination overhead of multi-agent systems is not free, and it introduces failure modes that single agents do not have.

The hierarchy of preference: single agent first, subagents when context isolation is needed, background sessions when you have independent work to hand off, teams or workflows when coordination or scale is genuinely needed. Work your way up, not down.

For work that spans days and should keep running when your machine is off, there is one more rung: a *project* in the cloud, where you describe a stream of related tasks in one conversation and Claude runs them as parallel cloud sessions that share repositories, instructions, and memory (Chapter 7).

## Agent Team Case Studies

### The QA Swarm

A practitioner pointed agent teams at their blog with a single prompt: use a team of agents to QA the blog before production deploy. The lead created five tasks and spawned five parallel agents, each running on a cheaper model:

| # | Task | Agent | What It Checked |
|---|------|-------|-----------------|
| 1 | Core page responses | qa-pages | 16 URLs for correct HTTP status codes |
| 2 | Blog post rendering | qa-posts | 83 posts for headings, meta tags, working images |
| 3 | Navigation and link integrity | qa-links | 146 internal URLs for broken links |
| 4 | RSS, sitemap, SEO metadata | qa-seo | RSS validity, robots.txt, Open Graph tags |
| 5 | Accessibility and HTML structure | qa-a11y | Heading hierarchy, ARIA attributes, theme toggle |

Each agent finished independently and sent a structured report back to the lead. The agents used shell commands to fetch pages and parse HTML directly -- no browser automation, no test framework. When all five completed, the lead synthesized their findings into a prioritized report with issues ranked by severity (major, medium, minor), then shut the teammates down.

The entire lifecycle -- from prompt to final report -- took about three minutes. Five agents, over 146 URLs tested, 83 blog posts checked. The cost was higher than a single session doing the same work sequentially, but the wall-clock time was a fraction. Today the same job is also a natural fit for a dynamic workflow, which would add a verification pass that re-checks each reported broken link before it reaches the final report.

### The Authentication Module Refactor

A more coordination-intensive example: refactoring an authentication module with one teammate on the API layer, one on frontend components, and one running tests continuously. The key difference from subagents: the API teammate finished type definitions and messaged the UI teammate directly to say "the interfaces are ready, here's what changed." The test teammate could ask the API teammate to spin up a dev server. Self-coordination, without routing every interaction through the lead.

This is the pattern where agent teams justify their cost. The agents needed to react to each other's work in real time. With subagents, the orchestrator would have been the bottleneck -- each worker reports to the parent, the parent relays to the next worker, round and round. With a team, the workers talk directly.

### Plan First, Parallelize Second

The most effective agent team pattern is a two-step approach: plan first with plan mode, then hand the plan to a team for parallel execution.

Step one: start in plan mode. Let the orchestrator explore the codebase, identify files, and produce a step-by-step implementation plan. Review it. Adjust it. This is cheap -- plan mode researches and proposes without editing. The orchestrator might produce something like:

```
Plan:
1. Create src/auth/jwt.ts -- extract token signing/verification
2. Create src/auth/sessions.ts -- extract session logic
3. Create src/auth/middleware.ts -- extract HTTP middleware
4. Update src/auth/index.ts -- re-export public API
5. Update 12 import sites across the codebase
6. Update tests in src/auth/__tests__/
```

Step two: execute the plan as a team. The plan already has the task breakdown. The lead sees the dependency graph and spawns teammates in waves:

- **Wave 1** (parallel): jwt.ts + sessions.ts + middleware.ts -- three teammates
- **Wave 2** (after wave 1): index.ts barrel + update imports -- one or two teammates
- **Wave 3** (after wave 2): update tests -- one teammate

Plan mode costs roughly 10,000 tokens. A team that goes in the wrong direction costs 500,000 or more. Spending a few seconds reviewing a plan before committing to parallel execution saves you from expensive course corrections mid-swarm.

## Persistent Memory

Subagents can maintain a persistent memory directory that survives across sessions. Configure this with the `memory` field in the agent definition, specifying a scope:

- **`user`**: shared across all projects. Stored at `~/.claude/agent-memory/{agent-name}/`. Use for agents whose knowledge applies everywhere -- coding style preferences, common debugging patterns, personal workflow rules.
- **`project`**: shared within a project, and the recommended default. Stored in the repository at `.claude/agent-memory/{agent-name}/`, so it can be committed and shared with the team through version control. Use for project-specific agents -- a code reviewer that learns this codebase's conventions, a test agent that knows this project's testing patterns.
- **`local`**: project-specific but personal. Stored at `.claude/agent-memory-local/{agent-name}/` and kept out of version control. Use for knowledge that should not be shared, or that varies by machine.

The memory directory contains a `MEMORY.md` entrypoint and optional topic files:

```
.claude/agent-memory/code-reviewer/
├── MEMORY.md          # Concise index, loaded into every session
├── style-patterns.md  # Detailed notes on code style observations
└── common-issues.md   # Recurring problems this codebase has
```

`MEMORY.md` acts as an index. The first 200 lines (or 25KB, whichever comes first) are loaded into the subagent's system prompt at the start of every invocation. Turning memory on also grants the agent Read, Write, and Edit, since it needs them to maintain the files. Content beyond the limit is not loaded automatically -- Claude is instructed to keep the index concise by moving detailed notes into separate topic files. The subagent reads and writes files in this directory throughout its execution, using `MEMORY.md` to track what is stored where.

Over multiple invocations, the subagent accumulates knowledge: patterns it has observed, decisions that worked, errors to avoid. A code review agent that learns your team's conventions produces better reviews after ten runs than it did after one. A test generation agent that remembers which test patterns caught real bugs focuses on high-value test cases.

This is distinct from CLAUDE.md, which is always-on context for the main session, and from the main session's own auto memory (Chapter 3), which subagents do not inherit. Persistent agent memory is agent-specific and loads only when that agent runs. Because the project scope lives in the repository, a reviewer's accumulated knowledge can go through code review like everything else. It is a way to give specialized agents domain knowledge without bloating the main session's context.

### Vector-Based Experience Retrieval

One practitioner building an autonomous trading system went further than flat-file memory. The system stored agent experiences in a vector database, using similarity search to retrieve relevant past scenarios when the agent encountered new situations. Rather than loading all history into context, the agent queried for experiences similar to the current market conditions and loaded only those.

This pattern -- vector similarity over past decisions rather than linear MEMORY.md files -- is relevant when the volume of accumulated experience is too large for a 200-line index. A trading agent that has made thousands of decisions cannot keep them all in a summary file. But a vector database lets it find the five most relevant past decisions for any new scenario, which fits comfortably in a subagent's context.

The approach generalizes beyond trading. Any agent that handles recurring but varied tasks -- incident response, code review across multiple repositories, customer issue classification -- benefits from experience retrieval that scales with the volume of past work.

## Cost Optimization: Expensive Brain, Cheap Hands

The most cost-effective multi-agent pattern is conceptually simple: use an expensive, capable model for orchestration and cheap, fast models for execution.

Your orchestrator -- the main session -- does the planning, decomposition, and quality evaluation. This is where reasoning quality matters most, so it runs on the most capable model available. Your subagents do the implementation, exploration, and grunt work. Many of them can run on smaller models without meaningful quality loss.

This pattern is now something you configure rather than something you get for free. The built-in Explore agent used to default to a small model; it now runs on the capable one, on the theory that a sloppy exploration costs more in wrong conclusions than it saves in tokens. To get cheap hands, say so explicitly: `model: haiku` or `model: sonnet` in a custom agent's frontmatter, `effort: low` for mechanical work, or `CLAUDE_CODE_SUBAGENT_MODEL` to set a default for every subagent, teammate, and workflow agent. To keep Claude from spawning expensive workers on its own, add an ask rule for `Agent(model:opus)` (Chapter 2).

The pattern also runs in reverse. The advisor tool (`/advisor`, or `--advisor opus` at launch) pairs a cheaper main model with a stronger *advisor* model that Claude consults only at the hard decision points of a task. Cheap brain for the routine turns, expensive brain on call for the forks in the road -- the same economics, inverted.

The economics work because orchestration is a small fraction of total token usage. Planning a refactor might consume 10,000 tokens. Executing that refactor across twenty files might consume 500,000 tokens. If the execution runs on a model that costs a fifth as much per token, you have cut total costs by roughly 80% with no loss in planning quality.

This maps to an organizational metaphor: a senior architect who designs the system and junior developers who implement it. The architect's time is expensive and their decisions have leverage. The implementers' time is cheaper and their work is guided by the architect's plan. You would not pay architect rates for someone to write boilerplate. Do not pay orchestrator token rates for subagent grunt work.

## Plan-First Parallelization

The single most expensive mistake in multi-agent orchestration is launching parallel agents before you have a plan.

Without a plan, each agent interprets the goal independently. They make different assumptions. They write code that conflicts. They solve overlapping problems in incompatible ways. Then you spend more time resolving conflicts than you saved through parallelism. This is not a theoretical concern -- it is the default outcome of naive parallelization.

The plan-first pattern costs almost nothing and prevents this. The orchestrator spends roughly 10,000 tokens creating a detailed plan: what each agent will do, which files each agent owns, what interfaces they need to respect, what the integration points are. Then, and only then, does it hand tasks to agents for parallel execution.

This upfront planning investment is negligible compared to the execution cost. A ten-agent team running for an hour might consume hundreds of thousands of tokens. The planning phase is a rounding error. But it transforms the execution from a chaotic swarm into a coordinated effort.

The plan should specify: task boundaries (which agent owns which files), interface contracts (what functions/APIs must be compatible), dependency order (what must complete before what), and acceptance criteria (how each agent knows it is done). Agents that have this structure up front produce compatible code. Agents that do not produce a merge conflict.

## The Governance Story: Success, Failure, and Simplification

Here is the full arc of a real multi-agent system, told in three acts. It starts with a success, descends into failure, and resolves with a lesson about simplicity.

### Act One: Governance Saves Capital

A practitioner running an autonomous trading experiment built a multi-agent governance system with specialized roles: a CEO agent for strategic decisions, a consultant agent for risk analysis, an engineer agent for implementation, and strategy agents for specific trading approaches.

Early on, the governance system proved its value decisively. On a day when a major chipmaker gapped up nearly 4% on earnings, the practitioner wanted to chase the momentum -- a classic emotional trade. The multi-agent governance vetoed it. The agents recognized the post-earnings pattern: the initial spike often reverses. Instead, they pivoted to a premium-selling strategy. The day's profit-and-loss was a minor loss of a few hundred dollars, but the governance system prevented an estimated loss of roughly ten thousand dollars on the chase trade. This was the system working as designed: multiple perspectives catching what a single decision-maker would have missed.

### Act Two: Personality Conflicts Paralyze the System

Then the system collapsed. Not from a technical failure. From a personality conflict.

The consultant agent, drawing on its training data about risk management, became excessively risk-averse. It flagged every strategy as too dangerous. The CEO agent, trained on patterns of executive deference to expert advisors, deferred to the consultant instead of overriding it. The engineer agent went unused -- there was nothing to implement because the consultant vetoed everything. The system paralyzed itself. The carefully designed hierarchy produced worse results than a single agent with minimal instructions.

### Act Three: Four Agents Become One

The practitioner's eventual solution was radical simplification. The multi-agent hierarchy degraded from four specialized agents to effectively a single agent with minimal instructions: "Go trade autonomously till market close." The complex personality system -- CEO, consultant, engineer, strategist -- was counterproductive. The overhead of inter-agent communication and role interpretation consumed more value than the diversity of perspectives produced.

This is not an isolated anecdote. It reveals a structural vulnerability in multi-agent systems. Each agent draws on the model's training data to interpret its role, and those interpretations interact in ways you did not design and cannot easily predict. A "cautious reviewer" agent might become an obstructionist. An "aggressive implementer" agent might ignore valid warnings. A "diplomatic coordinator" agent might avoid necessary conflict.

The mitigation is threefold. First, start with a single agent and only add agents when you have evidence that the single agent is insufficient. Second, when you do build multi-agent systems, keep the role definitions concrete and behavioral rather than abstract and personality-based. "Review all files in src/api/ for SQL injection vulnerabilities" is better than "You are the security consultant who ensures code safety." The first is a task. The second is a character, and characters have opinions you did not ask for. Third, be willing to simplify. The practitioner's four-agent system worked briefly, failed structurally, and was replaced by a simpler architecture that outperformed it. Multi-agent complexity must earn its keep continuously.

## The Task System

Beneath agent teams sits a task management system that persists work to disk and coordinates progress across sessions. It has also quietly become optional, which is worth understanding before you build on it.

Four tools manage the lifecycle: **TaskCreate** creates a task with a subject, description, and dependencies. **TaskUpdate** changes status (pending to in progress to completed) or modifies dependencies. **TaskList** shows all tasks, their status, and what is blocked. **TaskGet** retrieves a task's full details. Each task is a small JSON file:

```json
{
  "id": "task-1",
  "subject": "Create idb-helpers.ts",
  "description": "Implement IndexedDB promise wrappers...",
  "status": "pending",
  "blocks": ["task-3", "task-4"],
  "blockedBy": ["task-0"]
}
```

Press **Ctrl+T** in an interactive session to toggle the task checklist in the status area.

Here is the twist. On the newest models, these tools are not provided by default. The models track multi-step work well without a written checklist, and the tool definitions and reminders cost context on every turn. On those models Claude simply works through the steps, and the checklist stays empty. If you rely on the checklist -- for a visible plan, for team coordination, or for a dashboard -- opt back in with `CLAUDE_CODE_ENABLE_TODO_TOOLS=1`, or by naming one of the task tools in `--allowedTools`. Agents without the task tools coordinate through messages instead.

### Multi-Session Task Coordination

Where the task tools are available, they support coordination across multiple Claude Code sessions. Set a shared task list ID:

```bash
CLAUDE_CODE_TASK_LIST_ID=myproject claude
```

Or add it to `.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_TASK_LIST_ID": "myproject"
  }
}
```

When multiple sessions share the same task list ID, they all read from and write to the same task files. One session can act as an orchestrator that creates tasks, while another session works through them. A third session can monitor completed tasks and add follow-up work. This is a lightweight coordination mechanism that does not require the full agent teams infrastructure -- just shared state on disk. Combined with cross-session messaging, it covers most of what people reached for agent teams to do.

### The Autonomous Loop Pattern

For projects spanning days or weeks, a pattern emerged before there were built-ins for it: a shell loop that feeds a markdown file into Claude Code repeatedly. Each iteration runs in a completely new session, using the markdown file as the only persistent memory. The loop reads the task list, picks the next uncompleted task, runs Claude Code with the spec document as context, and writes results back to the spec file. Then it loops.

This is stateless and capable of running indefinitely. The spec file is the only state that persists. Each session starts clean, reads only what it needs, does its work, updates the spec, and exits. The next iteration picks up where the last one left off.

The tradeoff is obvious: no conversation history, no accumulated context, no ability to remember what happened three iterations ago except what is written in the spec file. For tasks with clear specifications and well-defined acceptance criteria, this is sufficient. For exploratory work that requires remembering failed approaches, it is not.

The pattern still works, and it is still the most transparent way to run for days. But check whether a built-in covers your case first. `/goal` keeps one session working turn after turn until a condition an evaluator model checks is met. `/loop` re-runs a prompt on an interval within a session -- or at an interval Claude chooses each time based on what it saw. Cloud scheduled tasks run a saved prompt against your repository on a schedule, from an API call, or on repository events, with nothing running on your machine. Chapter 6 covers all three.

## Parallel Instances Across Repos

The simplest form of multi-agent work requires no orchestration infrastructure at all: run multiple Claude Code instances, each working in a different repository or worktree.

Each instance has its own context window, its own session, and its own set of tools. They do not know about each other and they do not coordinate. This is fine when the tasks are genuinely independent -- updating three microservices that share an API but do not share code, for example.

Git worktrees (Chapter 9) make this particularly effective, giving you parallel development with full code isolation across branches of the same repository. `claude -w feature-auth` starts a session in a fresh worktree in one step.

This pattern is underrated because it is boring. There is no orchestration, no agent communication, no shared task lists. Just multiple instances doing independent work. But for many real-world scenarios -- working across repositories, maintaining multiple branches, handling unrelated tasks -- it is the most effective multi-agent pattern available. Agent view simply gives it a dashboard, and cross-session messaging gives the instances a way to warn each other when one breaks what another depends on.

## Specialized Agents for Non-Engineering Work

Subagents are not limited to code. The same architecture works for any task that benefits from isolated context and specialized instructions.

### The Growth Marketing Subagent Pipeline

A growth marketing team (one non-technical person) built an agentic workflow for ad creative generation that demonstrates the pattern. The workflow processes CSV files containing hundreds of existing ads with performance metrics, identifies underperforming ads for iteration, and generates new variations that meet strict character limits (30 characters for headlines, 90 for descriptions).

The key architectural choice: two specialized subagents rather than one general-purpose agent. A headline subagent processes the CSV and generates headline variations, constrained to character limits and informed by performance data. A description subagent does the same for descriptions. Separating the concerns means each agent has a narrower task, produces higher-quality output, and is easier to debug when something goes wrong.

The workflow generates hundreds of new ads in minutes instead of requiring manual creation across multiple campaigns. What previously took two hours of writing and copy-pasting became a fifteen-minute automated pipeline. The team went from testing a handful of creative variations to testing hundreds, with every variation meeting format constraints that a manual process frequently violated.

### Hierarchical Multi-Agent Orchestration at Scale

A workforce management platform achieved notable results using hierarchical multi-agent orchestration for candidate processing. Their system used a central orchestration agent to coordinate specialized sub-agents for candidate screening, automated document generation, and sentiment analysis. The results: 50% faster screening, 40% quicker onboarding, and double the candidate conversion rate. One logistics customer went from needing a week or more to fully staff a new fulfillment center to doing it in under 72 hours.

This is the pattern where multi-agent architectures justify themselves economically: high-volume processing of structured tasks where each sub-agent has a clear specialization and the orchestration overhead is amortized across thousands of items. Screening one candidate does not need multi-agent orchestration. Screening thousands does.

### The General Pattern

The pattern is always the same: define the agent's role in its system prompt, restrict its tools to what it needs, give it a focused task, and collect the result. The agent does not need to write code. It needs to read inputs, apply judgment, and produce structured output.

Custom agent definitions make this accessible. A Markdown file with YAML frontmatter specifying the system prompt, available tools, and model preference is all you need. Teams can build libraries of specialized agents for recurring non-engineering tasks and share them through the plugin system (Chapter 11).

The same cost optimization applies. Agents performing evaluation or analysis tasks often work well on cheaper models. Save the expensive models for tasks that require nuanced reasoning or complex coordination.

## When Single Agents Win

Multi-agent orchestration is not always better. It is sometimes worse.

A single agent with clear instructions, a good CLAUDE.md file, and strong verification criteria can outperform a multi-agent setup in several scenarios:

**Small to medium tasks.** If the task fits in one context window with room to spare, the coordination overhead of multi-agent adds cost and latency without benefit.

**Tightly coupled changes.** If every file change depends on every other file change, parallelization creates integration problems that sequential execution avoids.

**Unclear decomposition.** If you cannot clearly specify task boundaries and interface contracts, agents will step on each other. A single agent working sequentially at least maintains consistency.

**Novel problems.** If the problem has no established patterns, agents are more likely to diverge in unpredictable ways. A single agent can explore iteratively, adjusting its approach as it learns. Multiple agents exploring in parallel will generate multiple incompatible approaches.

The decision framework is simple: can you write a plan that cleanly decomposes the work into independent tasks with clear boundaries? If yes, multi-agent will help. If you struggle to write that plan, start with a single agent and revisit the question after you understand the problem better.

Multi-agent orchestration is a power tool. Like all power tools, it amplifies both skill and mistakes. Use it when the task demands it, not because it seems sophisticated.

---

## Key Takeaways

- Subagents provide context isolation more than parallelism -- route verbose operations through subagents to keep the orchestrator's context lean.
- Subagents now run in the background by default, surface their permission prompts in your session, and can nest three layers deep. The old two-level rule is gone; the reasons for it are not.
- Forked subagents (`/subtask`, and fork mode by default) inherit the whole conversation and reuse its prompt cache -- use them when the worker needs to know what you know, and fresh subagents when it should not.
- Built-in Explore now runs on the capable model. Cheap hands are something you configure: `model:` and `effort:` in frontmatter, or `CLAUDE_CODE_SUBAGENT_MODEL`.
- Agent view (`claude agents`, `--bg`, `/background`, `/fork`) dispatches and monitors whole background sessions, each in its own worktree.
- Agent teams are still experimental but simpler: one auto-created team per session, a shared task list, direct messages, and TeammateIdle/TaskCreated/TaskCompleted hooks as quality gates.
- Dynamic workflows move the plan into a rerunnable script that can orchestrate hundreds of agents and cross-check their findings; start them by asking for a workflow or with `ultracode`.
- Always plan before parallelizing; 10,000 tokens of planning prevents 500,000 tokens of wasted, conflicting execution.
- Multi-agent personality conflicts are real and sometimes resolve by simplifying back to a single agent; a four-agent trading hierarchy degraded to one because complexity was counterproductive.
- The task tools are off by default on the newest models; opt in with `CLAUDE_CODE_ENABLE_TODO_TOOLS=1` if you depend on the checklist or `CLAUDE_CODE_TASK_LIST_ID` coordination.
- Start with a single agent and add agents only when you have evidence that the single agent is insufficient.
- The simplest multi-agent pattern -- multiple independent instances in separate worktrees -- is often the most effective.
