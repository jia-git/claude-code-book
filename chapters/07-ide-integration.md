# Chapter 7: IDE Integration Done Right

## What You'll Learn

Claude Code runs in your terminal. It also runs in a code editor extension, a desktop app, a web interface, and a mobile app. Same engine everywhere. Same CLAUDE.md files. Same settings. Same MCP servers. The difference is the surface -- the interface you interact with and the workflow patterns it enables.

Most developers pick one surface and stay there. That is fine, but it leaves workflow optimizations on the table -- and the number of surfaces has roughly doubled since the first edition. This chapter maps out the surfaces, their strengths, their gaps, and the patterns that emerge when you understand which surface fits which task. You will learn how the desktop application grew from a diff viewer into a workbench with parallel Git-isolated sessions, an integrated terminal, a built-in browser, and computer use; how cloud sessions run on managed VMs you can start from your terminal and pull back with teleport; how Remote Control lets your phone drive a session that is running on your desk; and how the browser extension and computer use let Claude verify its work in the actual running application. You will also confront the autocomplete gap -- the one capability where Claude Code does not compete with other tools -- and learn why the dominant usage pattern across teams is still terminal-first with visual surfaces pulled in when they add value.

---

## Same Engine, Every Surface

Claude Code's architecture separates the engine from the interface. The engine handles the agentic loop, context management, tool execution, permissions, and all the machinery described in previous chapters. The interface -- terminal, editor extension, desktop app -- is a presentation layer.

This matters because it means switching surfaces does not mean switching tools. Your CLAUDE.md files load regardless of surface. Your permission settings apply everywhere. Your MCP servers connect to every surface that supports them. A workflow you develop in the terminal works in the editor extension with only minor adaptations.

The surfaces currently available: the terminal (the original and most capable), extensions for a popular code editor and a major IDE family, a desktop application (now on Linux in beta as well), a web interface for cloud sessions, a mobile app, a browser extension, and integrations for team chat and CI. Each surface optimizes for different interaction patterns, but none of them changes what Claude Code fundamentally does. Configuration, project memory, and MCP servers are shared across the local surfaces; cloud sessions read what is committed to the repository plus what you configure for the cloud environment.

## The Terminal Surface

The terminal is where Claude Code started and where it remains most powerful. Every feature described in this book works in the terminal. No exceptions.

The terminal surface gives you the full CLI with all flags, direct access to piping and Unix composability (Chapter 6), vim-style modal editing for prompts (now toggled from `/config` rather than a dedicated command), background bash command execution, agent view for parallel sessions, and the Agent SDK. It is the only surface where you have complete control over session management, output formats, and automation integration.

It is also a better terminal citizen than it was. A fullscreen renderer (`/tui fullscreen`) removes flicker, adds mouse support, and keeps memory flat in long conversations, and in that mode `/diff` opens a live panel beside the conversation that refreshes as Claude edits. Prefix a line with `!` and the command runs directly in your shell, its output lands in the session, and Claude responds to it -- `! npm test` gets you the failures and an explanation in one step. `/voice` enables push-to-talk dictation for prompts. A screen reader mode replaces the visual interface with plain linear text. None of these change what Claude does; all of them change how long you can stand to sit in front of it.

Terminal-first workflows dominate among experienced users for a practical reason: the terminal does not get in the way. There is no GUI mediating your interaction. You type a prompt, Claude runs, you see results. The feedback loop is tight. When you need to inspect a file that Claude modified, you open it in your editor. When you need to run a test, you run it in another terminal pane. The terminal is not trying to be an IDE. It is a command surface that works alongside whatever IDE you prefer.

This terminal-first approach has an underappreciated benefit: it is IDE-agnostic. Teams with mixed editor preferences -- some using one code editor, others using another IDE family, others on a minimal text editor -- can standardize on Claude Code workflows without standardizing on an editor. The CLAUDE.md file, the agent definitions, the MCP configurations, the hook system -- all of it works identically regardless of which editor is open in the next window.

## The Code Editor Extension

The extension for the popular code editor adds a visual layer on top of the Claude Code engine. The key features are inline diff viewing, @-mention file references, plan review interfaces, and conversation history.

**Inline diffs** are the headline feature. When Claude proposes changes to a file, you see the diff rendered in the editor with syntax highlighting, accept/reject controls, and the ability to partially accept changes. This is materially faster than reviewing diffs in the terminal, especially for large changes across multiple files.

**@-mentions** let you reference files, symbols, and selections directly in your prompt. Click on a function name, type @, and the file path is inserted into the context. This is a convenience feature -- you can achieve the same thing in the terminal by typing the path -- but it reduces friction enough to change behavior. Developers who use @-mentions reference specific context more often, which leads to better-scoped prompts.

**Plan review** surfaces Claude's intended approach in a structured visual format before execution. In the terminal, plans are text blocks in the conversation (Ctrl+G opens them in your editor). In the editor extension, they are a reviewable document you can approve, comment on, or send back. This is particularly useful for large refactors where the plan has many steps and you want granular control over which steps proceed.

A permission-mode indicator at the bottom of the prompt box switches modes with a click; the extension starts in auto mode like the terminal, unless you set its own initial-mode setting (project settings cannot pick its starting mode, a deliberate guard). A focus view collapses each turn to your prompt, a one-line summary of tool calls with diff stats, and the final answer. Background subagents show up as a single status line under the card that launched them, with full transcripts in `/tasks`.

The extension is strongest when you are working on a single codebase and want tight integration between Claude's output and your editing workflow. It is weakest when you need features that exist only in the terminal -- specific CLI flags, piped workflows, or the Agent SDK.

## The Desktop Application

The desktop application is a standalone surface that runs Claude Code outside both your IDE and your terminal. Its Code tab is the Claude Code workspace: you select a project folder, choose a permission mode, pick a model, and start typing prompts, with the same engine underneath as the CLI. It bundles its own copy of Claude Code, so there is nothing separate to install.

What began as a visual wrapper is now closer to a workbench. Panes can be dragged into any layout or popped out into their own windows. There is an integrated terminal and file editor, side chats for questions you do not want in the main thread, app previews that show your running web app beside the conversation, a built-in browser Claude can drive the same way it drives a preview, and PR monitoring. On supported plans, computer use lets Claude operate native applications on your machine -- in the background on one desktop OS, so you can keep working -- and an iOS simulator pane lets Claude build, run, and tap through a mobile app while you watch. Desktop also has its own scheduled tasks, and when you hit a usage limit it can retry the interrupted turn on its own once the limit resets.

### Permission Modes

The desktop application uses the same permission modes as the terminal (Chapter 2), surfaced through a selector under the prompt box: Manual for approving each action, Accept edits for iterating on code you will review afterward, Plan for analysis without edits, Auto for classifier-reviewed autonomy, and Bypass only in isolated environments. Choosing Plan applies to the current session only.

You can switch modes mid-session. Start in Plan to map out an approach, approve the plan into Auto to execute, drop to Manual for the final verification steps.

### Visual Diff Review

When Claude modifies files, a diff stats indicator appears showing lines added and removed. Clicking it opens a full diff viewer with a file list on the left and changes on the right. You can comment on specific lines by clicking them -- type your feedback, press Enter, and after commenting on multiple locations, submit all comments at once. Claude reads the comments and makes the requested changes, which appear as a new diff. This is review-by-conversation, not review-by-reading. You point at problems, and Claude fixes them.

### Parallel Sessions with Git Isolation

Each session in the desktop application gets its own isolated copy of your project through Git worktrees. Click "New session" in the sidebar to start a second (or third, or tenth) parallel task. Changes in one session do not affect other sessions until you commit them. Worktrees are stored in your project's `.claude/worktrees/` directory by default. You can configure a custom location and a branch prefix in settings to keep Claude-created branches organized.

This turns the desktop application into a parallel processing interface. One session is refactoring the authentication module. Another is writing tests for the payment flow. A third is investigating a bug. Each session operates in its own Git branch, completely isolated from the others. When each session's work is done, you review and merge the branches independently.

### SSH and Remote Sessions

The desktop application connects to remote machines over SSH. Instead of selecting "Local" as your environment, select an SSH connection. Claude Code runs on the remote machine, executing commands and modifying files there. This is useful for codebases that require specific hardware, operating systems, or development environments that your local machine does not provide.

For long-running tasks, select a Cloud environment instead. Cloud sessions run on Anthropic-managed infrastructure (or your organization's own, through a self-hosted environment) and continue even if you close the application or shut down your computer. You can monitor them from the desktop application, the web interface, or a mobile app. More on cloud sessions in the next section.

Desktop and the terminal are no longer separate worlds. `claude --desktop` opens the app on the current directory, `/desktop` hands a running terminal session to it, and typing `/resume` in the desktop prompt picks up any session you started from the CLI with its full conversation intact.

### Connectors

The desktop application supports connectors that integrate external services -- version control platforms, team messaging tools, and project management systems. These connectors are configured through the application's settings and make Claude Code aware of your project's external context: PRs, issues, messages, tasks. The connector ecosystem is still growing, but the pattern is clear: Claude Code becomes the hub that connects your code to your team's communication and coordination tools.

## Claude Code on the Web

Claude Code runs in the browser as a fully cloud-hosted environment. No local setup required. You connect your version control account, select a repository, and start working. The repository is cloned to an isolated virtual machine, a cloud environment is applied, and Claude executes with full access to the codebase. The VM costs nothing extra; the session draws on your plan's usage like any other.

The web interface provides the same diff view as the desktop application -- you see exactly what Claude changed, comment on specific lines, compare against any branch, and iterate until the changes are ready. When satisfied, you create a pull request directly from the interface, and you can turn on auto-fix for it (Chapter 6).

### Cloud Environments

A cloud environment is a saved configuration: network access level, environment variables, network secrets, and a setup script that installs whatever your project needs. The default environment uses a trusted allowlist -- package registries, API endpoints, and similar infrastructure -- and you can open it fully or cut it off entirely. Outbound traffic goes through a security proxy, and version control credentials never enter the VM: a separate proxy attaches them server-side, so even a fully compromised session cannot read your token. Network secrets work the same way, attached to matching requests after they leave the sandbox. Setup scripts are cached, so the second session starts much faster than the first.

For organizations that cannot run code on someone else's infrastructure, self-hosted environments route cloud sessions to runners you operate, inside your own network, with the same interface on top.

### Session Sharing

Cloud sessions can be shared by changing their visibility. On organization plans the options are Private and Team; on individual plans, Private and Public. Recipients see the session's latest state when they open the link. Shared sessions let others review the conversation history and the resulting changes -- useful for code reviews, pair programming across time zones, and demonstrating workflows. Check a session before making it public: it may contain code and credentials from a private repository.

### When to Use Cloud Sessions

Cloud sessions shine for three scenarios. First, long-running tasks that would tie up your local machine -- kick off a large refactor, close the browser, and check back in an hour. Second, repositories you do not have locally -- review a colleague's project without cloning anything. Third, parallel execution -- start multiple cloud sessions on different tasks and monitor them from a single browser tab.

The limitation is narrower than it was. Cloning and pull-request creation still require the dominant code platform (including its self-hosted enterprise edition, on organization plans). From any other host -- or a repository with no remote at all -- `claude --cloud` can bundle your local repository and upload it, though the session cannot push results back to that remote. On some platforms, uncommitted changes to files named like credentials are left out of the upload automatically.

### Projects: Many Sessions, One Conversation

For a body of work that spans days -- a migration, a backlog, a quarter's worth of cleanup -- a *project* turns the cloud into a team. You describe the stream of work once, in one ongoing conversation; Claude starts parallel cloud sessions called threads, gives each the project's shared instructions and memory, and tells you which ones need you. Threads can also run on your own machine through Remote Control when you ask. Projects are in public beta on individual plans, and they are the right tool when you would otherwise be dispatching and tracking the same kind of session by hand every morning.

## Claude Code on Mobile

The mobile application provides a monitoring and dispatch interface. Kick off a cloud task from your phone, monitor its progress, steer Claude with follow-up prompts, and review the results when they are ready. You are not going to write complex prompts on a phone keyboard, but you can check on a long-running task during your commute, answer Claude's clarifying questions, or approve a PR from the couch. Push notifications ping you when a long task finishes or Claude needs an answer.

The bigger change is that the phone can now reach your *local* sessions, not just cloud ones. Three mechanisms do this, and they are easy to confuse:

**Remote Control** makes a session running on your machine -- in the terminal or the editor extension -- drivable from the web or the mobile app. Run `/remote-control` inside a session, or start one with `claude --rc`. The session keeps executing on your machine with your files, your tools, and your MCP servers; the phone is a window into it. `claude remote-control` runs a server mode that hosts sessions without a local terminal, and any machine running it shows up on your phone as a device you can start new sessions on.

**Dispatch** sends a task from your phone to the desktop app, which spawns a local session to do it.

**Cloud sessions** run on remote infrastructure and do not touch your machine at all.

The decision is about where the work needs to happen. If it needs your laptop's files and tools, use Remote Control or Dispatch. If it must keep running when the laptop is closed, use the cloud.

## The Browser Extension and Computer Use

A browser extension -- now generally available -- connects Claude Code to your real, logged-in browser. It bridges the gap between the codebase and the running application: Claude can navigate pages, fill forms, read console logs, observe network requests, and inspect the rendered output of the code it is working on. Enable it with `--chrome` at launch or configure it with `/chrome`; with several browsers connected, you choose which one Claude drives. This is most valuable for frontend debugging, where the symptom (a visual bug in the browser) and the cause (a CSS or JavaScript issue in the code) live in different contexts. The extension brings them together -- and because it uses your sessions, it also automates sites that have no API.

Computer use goes one step further, past the browser to native applications. In the CLI on one desktop OS, toggle the computer-use server on in `/mcp`, and Claude can open apps, click through UI, and take screenshots to verify a change that only a GUI can show -- a native app's onboarding flow, a visual regression in a desktop tool. The desktop app offers the same capability, and the iOS simulator pane is a specialized version of it.

The pattern these share is the most important idea in this chapter: verification moved closer to reality. A passing test suite says the code does what the tests check. A screenshot of the running app says the change actually worked. Chapter 8 makes the case that a check Claude can run is the single most important thing in a prompt; these surfaces are how you give Claude a check for work that has a visual result. The bundled `/run` and `/verify` skills package the same idea for your project -- launch the app, drive it, observe the result -- and `/run-skill-generator` teaches them how to launch yours.

## The IDE Family Plugin

The plugin for the major IDE family provides interactive diff viewing and context sharing. The implementation differs from the code editor extension in ways that reflect the IDE's architecture -- diffs integrate with the IDE's native diff viewer, context references use the IDE's symbol index, and the interaction model follows the IDE's conventions.

The practical differences between the two editor integrations are smaller than the architectural differences suggest. Both provide inline diff review. Both support context referencing. Both connect to the same Claude Code engine with the same capabilities. The choice between them is almost always driven by which IDE you already use, not by feature differences in the Claude Code integration.

## Cross-Surface Workflows

Surfaces are not isolated. Sessions can move between them. This is one of Claude Code's most distinctive architectural decisions -- no other tool in this category supports fluid session migration across interfaces.

### The /teleport Command

The `/teleport` command (also `/tp`) pulls a cloud session into your terminal. Start a long-running task on the web or from your phone, let it run while you are away, then pull it back to your local machine when you are ready to review and continue.

```
/teleport
```

This opens an interactive picker showing your cloud sessions. Select one, and Claude Code verifies you are in the correct repository, fetches and checks out the branch from the remote session, and loads the full conversation history into your terminal. If you have uncommitted local changes, it prompts you to stash them first.

The `--teleport` flag works the same way from the command line:

```
claude --teleport
claude --teleport <session-id>
```

The teleported copy is yours: new work in the terminal stays local and does not flow back to the cloud session. If you want to keep steering from your phone after pulling a session down, start Remote Control in the local session.

### Sending Work to the Cloud

The reverse direction -- terminal to cloud -- starts a *new* cloud session from your current repository:

```
claude --cloud "Fix the authentication bug in src/auth/login.ts"
```

The cloud VM clones your remote at your current branch, not your local checkout, so push first. (The first edition described an `&` prefix for this; `--cloud` replaced it, and `--remote` survives as a deprecated alias.) Each `--cloud` call starts an independent session, so three commands give you three parallel tasks. You can also queue a follow-up into a running cloud session from any machine with `claude -p "your message" --cloud <session-id>`. Use `/tasks` to monitor background work -- pressing `t` teleports you into a cloud session. An existing terminal session cannot be pushed to the cloud from the CLI, though the desktop app can send one of its local sessions there.

The pattern that emerges is planning locally and executing remotely. Start Claude in plan mode to collaborate on the approach, commit the plan to the repository, then send it to the cloud for autonomous execution:

```
claude --cloud "Execute the migration plan in docs/migration-plan.md"
```

### The /desktop Command

The `/desktop` command (alias `/app`) hands off a terminal session to the desktop application. The full conversation context moves to the GUI, where you get visual diff review and inline comments. This is the right move when the task shifts from exploration to review.

### Deep Links

One more cross-surface trick: a `claude-cli://` link opens a terminal session in the right repository with a prompt already filled in. Put them in runbooks, alert payloads, and dashboards, and an on-call engineer goes from "page received" to "Claude is investigating this exact alert in this exact repository" in one click.

### Workflow Patterns

The cross-surface workflow that emerges is: explore and plan in the terminal, implement in the terminal or editor, review diffs in the desktop application or editor extension. The terminal is the home base. Visual surfaces are specialized tools you invoke when review adds value.

This is not how most IDE integrations work. Most tools embed in the IDE and never leave. Claude Code inverts that relationship. The terminal is the primary surface, and the IDE is a supplementary view you pull up when needed. Understanding this inversion prevents the frustration of trying to force terminal-grade functionality through the editor extension.

## PR Review Status in the Footer

When working on a branch with an open pull request, Claude Code displays a clickable PR link in the terminal footer (for example, "PR #446"). The link has a colored underline indicating the review state:

- **Green**: approved
- **Yellow**: pending review
- **Red**: changes requested
- **Gray**: draft
- **Purple**: merged

The status updates automatically. Cmd+click or Ctrl+click opens the PR in your browser. This small feature eliminates a constant context switch -- checking PR status without leaving the terminal. It requires the version control platform's CLI to be installed and authenticated, and it now works for merge requests on the major alternative platform as well. Agent view (Chapter 4) shows the same status on each session's row.

## Eliminating Context Switching

One pattern that practitioners report consistently: Claude Code in the terminal eliminates the context switch between a cloud AI chat interface and the codebase. Without Claude Code, the workflow is: copy a code snippet, paste it into the chat interface, explain the problem, read the response, copy the suggested fix, paste it back into the editor, test it, and repeat. Each round trip crosses application boundaries and loses context.

With Claude Code, the entire conversation happens inside the project. Claude reads files directly. It writes changes directly. It runs tests directly. The context is the codebase, not a description of the codebase. The difference is not marginal -- it eliminates the mental overhead of translating between two environments and the mechanical overhead of copy-paste.

## Code Intelligence Plugins

The plugin system includes Language Server Protocol servers -- "code intelligence" plugins -- and the first edition undersold them. They are not for your editor. They are for Claude.

Without a language server, Claude finds where a symbol is defined the way a newcomer would: grep, then read three candidate files. With one, it calls go-to-definition and find-references directly, and after every edit it receives the type errors and warnings the change introduced, without running a compiler. That makes them a context-saving tool (one symbol lookup replaces several file reads) and a correctness tool (the type error surfaces in the same turn as the mistake).

The official marketplace has plugins for most mainstream typed languages; each needs that language's server binary installed locally. Install one per language you work in, or enable it for everyone in a repository through the project's plugin settings. Claude Code will often suggest the right one when it notices which language you are using. In a large typed codebase, this is one of the highest-leverage plugins you can add (Chapter 9).

## The Autocomplete Gap

Claude Code does not have streaming inline autocomplete. The kind of experience where you type a few characters and a gray ghost of the suggested completion appears in your editor, updating with each keystroke -- that is not what Claude Code does.

This is not an oversight. It is an architectural consequence. Claude Code's strength is in agentic, multi-step operations that span files and tools. Inline autocomplete is a fundamentally different interaction pattern: low-latency, single-line, character-by-character prediction. Building both well in the same tool requires different model configurations, different infrastructure, and different UX.

The gap matters because inline autocomplete is genuinely useful. For writing boilerplate, filling in function signatures, completing variable names, and the countless small editing tasks that do not warrant a full prompt-and-execute cycle, a good autocomplete tool saves real time.

The recommended approach is a hybrid setup. Use Claude Code for the agentic, multi-step, multi-file work it excels at. Use a separate tool that specializes in inline autocomplete for the character-by-character editing work. The two tools do not conflict. They operate at different scales of interaction -- one handles the forest, the other handles individual leaves.

This hybrid strategy is not a workaround. It is the mature approach. No single tool dominates every interaction scale, and pretending otherwise leads to using a powerful tool badly for tasks where a simpler tool works better.

## When to Use the IDE, When to Drop to Terminal

The decision is simpler than it seems.

**Use the IDE extension when:**
- You need to review multi-file diffs visually
- You are referencing specific code locations with @-mentions
- You want to partially accept or reject changes within a file
- You are doing plan review and want the interactive checklist
- You are working with a teammate who is more comfortable in the IDE

**Drop to the terminal when:**
- You need CLI flags not exposed in the IDE extension
- You are running multiple parallel sessions
- You are piping input or output to other tools
- You need automation or scripting integration
- You want the fastest possible feedback loop
- You are doing exploratory work where conversation flow matters more than diff review

Most sessions start in the terminal. Some move to the IDE for review. Few start in the IDE and stay there. This is not a judgment about IDE quality -- it reflects the fact that agentic coding is primarily a conversation, and conversations flow most naturally in the terminal.

## The Design Tool Pattern

A pattern worth noting: teams report that a visual design tool and Claude Code are open simultaneously roughly 80% of the time. The design tool provides the visual reference -- mockups, layouts, component designs. Claude Code implements what the design shows.

This is not an IDE integration in the traditional sense. It is a workflow where two tools complement each other without being connected. The human is the integration layer, looking at the design and describing to Claude Code what needs to be built.

The pattern works because Claude Code accepts images as input. You can paste a screenshot of a design mockup into the conversation, and Claude can interpret the layout, colors, spacing, and component structure. This is faster than describing a design in words, though it is not a substitute for explicit implementation requirements.

The design-to-code workflow follows the same four-phase pattern described in Chapter 1: explore the existing codebase components, plan the implementation approach, implement the design, and commit. The design tool adds a visual reference to the exploration phase that makes the subsequent phases more grounded.

## Skills as a Cross-Agent Standard

Skills are not exclusive to Claude Code. They are a shared standard supported by multiple AI coding agents. A skill you write for Claude Code works with other agents that support the standard -- and the list is growing. This means your investment in skill development is not locked to a single tool.

The installation workflow uses a package runner command that works across agents:

```
npx skills add <repository-url> --skill <skill-name>
```

Running this command clones the repository, discovers available skills (a typical repository might contain a dozen or more), detects compatible agents, asks which scope to install in (project or user), and creates the skill directory structure. Skills installed this way can be symlinked or copied, and Claude Code's live change detection picks them up without a restart.

The cross-agent compatibility has a practical consequence: teams that use multiple AI coding tools can standardize their workflows through skills rather than building tool-specific configurations. A code review skill, a deployment workflow, or a documentation generator works regardless of which agent runs it. This reduces the cost of experimenting with different tools and prevents workflow lock-in.

## The Hybrid Toolchain

No single AI coding tool dominates every interaction scale. The mature approach is a hybrid setup with each tool handling what it does best.

Use Claude Code for the agentic, multi-step, multi-file work it excels at: repository-wide refactors, complex implementations, codebase exploration, architectural planning, and automated workflows. Layer a dedicated inline completion tool for the character-by-character editing work -- function signatures, variable names, boilerplate, and the countless small editing tasks that do not warrant a full prompt-and-execute cycle.

This is not a workaround for a missing feature. It is an acknowledgment that agentic coding and inline completion are fundamentally different interaction patterns requiring different model configurations, different infrastructure, and different UX. Building both excellently in the same tool requires different optimization targets that inevitably trade off against each other.

The first edition predicted that Claude Code's inline IDE experience would become more aggressive. What actually happened is instructive: the surfaces grew in the *agentic* direction instead -- background sessions, cloud handoff, browser and computer control, visual verification -- not toward keystroke-level completion. The underlying architecture continues to optimize for a different scale of interaction. The hybrid approach is not a temporary compromise. It is the stable end state.

## Terminal First as a Design Philosophy

The terminal-first pattern is not just a preference. It is a design philosophy that shapes how Claude Code evolves.

Because the terminal is the universal surface -- every operating system has one, every IDE has one embedded, every remote server is accessible through one -- building features for the terminal first ensures they work everywhere. An extension-first approach would leave terminal users behind. A terminal-first approach gives everyone the full feature set, with extensions providing visual enhancements on top.

This philosophy has practical implications for how you invest your learning time. Learning the terminal workflow gives you skills that transfer to every surface. Learning the extension workflow gives you skills specific to that extension. If you have limited time, invest in the terminal first and add extension workflows as refinements.

The terminal-first philosophy also explains why Claude Code does not try to replace your IDE. It sits alongside your IDE as a complementary tool, handling the agentic work that IDEs do not do well while leaving the editing, debugging, and navigation work to tools purpose-built for those tasks. The integration is collaborative, not competitive.

---

## Key Takeaways

- Claude Code's engine is surface-independent -- CLAUDE.md, settings, MCP servers, and permissions work identically across terminal, editor extensions, desktop, web, and mobile.
- The terminal remains the most complete surface; the fullscreen renderer, `!` shell mode that Claude responds to, voice dictation, and screen reader mode make it easier to live in.
- The desktop app is now a workbench: parallel Git-isolated sessions, integrated terminal and editor, app previews, a built-in browser, computer use, and its own scheduled tasks -- with sessions moving freely to and from the CLI.
- Cloud sessions run on isolated VMs with credentials kept outside the sandbox; start them with `claude --cloud`, pull them back with `/teleport`, and use projects for multi-day streams of work.
- Remote Control drives a session running on your machine from your phone; Dispatch sends a task from your phone to the desktop app; cloud sessions need neither.
- The browser extension and computer use let Claude verify changes in the running application, not just in tests.
- Code intelligence plugins give Claude go-to-definition and live type errors -- fewer file reads, faster correction.
- Skills are a cross-agent standard: `npx skills add` installs skills that work across multiple AI coding tools, preventing workflow lock-in.
- The inline autocomplete gap is architectural, not accidental; a hybrid setup with Claude Code for agentic work and a dedicated autocomplete tool for editing is the stable end state.
- Invest learning time in terminal workflows first -- they transfer to every surface and every IDE.
