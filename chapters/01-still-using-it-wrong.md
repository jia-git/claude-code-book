# Chapter 1: Beyond the Getting-Started Guide

## What You'll Learn

Most developers plateau with Claude Code somewhere around day three. They have the basics down -- type a prompt, get some code, copy it into place -- and they stop there. Not because they are doing anything wrong, but because nobody told them there was more. This chapter is for developers who sense there is a deeper layer but have not found the entrance.

Claude Code is an agentic system. It reads your codebase, plans its approach, executes changes across dozens of files, runs your tests, reads the errors, and tries again. It chains tool calls in loops, not single-shot completions. The moment you internalize this distinction, your entire relationship with the tool changes. You stop asking it questions and start giving it assignments.

This chapter lays out the mental model that separates surface-level use from advanced practice. You will understand the agentic execution loop and why it can be built in a few hundred lines of code, why the context window is the only constraint that matters, how to structure sessions for maximum throughput, and why your instinct to micromanage is the single biggest bottleneck to productivity. You will learn how teams classify tasks for autonomous versus supervised execution, why making Claude Code your first stop on every task changes your workflow, and how to think of the tool as an iterative thought partner rather than a vending machine. You will also confront an uncomfortable number: research shows developers can fully delegate only a small fraction -- somewhere between zero and twenty percent -- of their tasks, and pretending otherwise leads to the kind of silent failures that cost more time than they save.

---

## The Machine You Think You're Using

There is a mental model most developers bring to Claude Code, inherited from years of chatbots and autocomplete tools: type question, receive answer. That mental model undersells the tool by an order of magnitude, and it shapes everything about how they use it.

Claude Code is a loop. Not a request-response pair. A loop. The architecture looks like this: read context, form a plan, take an action, verify the result. Then do it again. And again. A single prompt from you might trigger thirty or forty iterations of this cycle, each one reading files, writing code, running commands, checking outputs, and deciding what to do next.

### The Agentic Loop in 350 Lines

The harness that runs this loop is deceptively simple. A minimal implementation -- the kind of educational prototype that community developers have built to demystify agentic systems -- is roughly 350 lines of code. The core pattern is three functions: call the model, process the tool calls from the response, and loop. If there are tool results, append them to the messages and call the model again. If there are none, the task is done.

```
response = callApi(messages, systemPrompt)
toolResults = processToolCalls(response.content)
if toolResults.length == 0: return   // done
messages.append(response, toolResults)
// loop again
```

Tools are registered in a map -- read a file, write a file, run a command -- and each tool call from the model gets dispatched to the corresponding handler. The model reasons about what tool to call next. The harness executes the tool and feeds the result back. That is the entire architecture.

In production Claude Code, every step is wrapped in permissions checks, context management, hook evaluations, subagent orchestration, and safety constraints. But the core is still that loop. Understanding this matters because it changes what "prompting" means. You are not writing a query. You are briefing a colleague who has access to your entire codebase, your terminal, your test suite, and your build system. That colleague will go explore, make a plan, implement it, and come back when it is done or when it is stuck. The quality of your initial briefing determines everything.

## Context Window: The Only Resource That Matters

Every feature of Claude Code, every architectural decision, every best practice traces back to one constraint: the context window.

Think of it as RAM for the conversation. Every file Claude reads, every command output it captures, every tool result it processes, every message you exchange -- all of it accumulates in this fixed-size buffer. When it fills up, performance does not degrade gracefully. It falls off a cliff. Instructions get forgotten. Work gets dropped. Claude starts solving problems it already solved.

This is not a theoretical concern. It happens in every long session. Auto-compaction kicks in when the window is nearly full, summarizing older content to make room -- but the summarization is lossy. Chapter 3 covers the compaction mechanics and how to survive them.

Here is the counterintuitive part. The current generation of models ships with a million-token window by default -- five times the old ceiling. You would expect the constraint to have disappeared. It did not. It moved. A bigger window means you hit the wall later, but every token in that window is still re-sent on every request, still competes for the model's attention, and still counts against your plan limits. A million tokens of stale exploration is not a resource. It is a very large haystack. The developers who treated the old 200K window as a hard budget kept the habit, and they are the ones whose sessions still feel sharp at hour three.

The practical consequence is that every interaction has a cost measured in context space, and you need to think about that cost the same way you think about compute budgets or memory allocation. Reading a 2,000-line file is expensive. Dumping verbose test output into the conversation is expensive. Having Claude search for something it could find with a targeted glob is expensive.

The developers who get the most from Claude Code are the ones who treat context like a scarce resource. They use subagents to isolate verbose operations (Chapter 4). They keep CLAUDE.md files concise -- under 200 lines (Chapter 3). They ask throwaway questions with `/btw` so the answer never enters the conversation history. They compact proactively instead of waiting for auto-compaction to butcher their context at the worst possible moment.

## The Four-Phase Workflow

If you have been typing implementation requests directly into Claude Code, you have been skipping the most important step.

The workflow that consistently produces the best results has four phases: explore, plan, implement, commit. Skipping exploration is the single most common mistake, and it leads to Claude confidently solving the wrong problem with technically correct code.

**Explore.** Start in plan mode (`/plan`, or Shift+Tab until the status bar says so). Let Claude read your codebase. Point it at relevant directories. Ask it to understand the existing patterns, the conventions, the architecture. This is cheap -- plan mode keeps Claude researching and proposing rather than editing, so it is figuring out the lay of the land instead of rearranging it.

**Plan.** Still in plan mode. Ask Claude to outline its approach. Review the plan. Press Ctrl+G to open it in your editor and change it directly rather than arguing with it in chat. Push back on parts that do not match your mental model. This is where misunderstandings surface, and fixing a misunderstanding in a plan costs nothing. Fixing it in implementation costs you a revert and a burned context window. The plan is also written to a file, and Claude Code re-injects that file after every compaction -- so on a long task the plan survives where the conversation that produced it may not.

**Implement.** Approve the plan, and Claude Code offers to switch you into auto mode, accept-edits mode, or manual mode for execution. Claude now has the context from exploration and a reviewed plan. Implementation quality is dramatically higher because Claude is working from understanding rather than assumption. Let it run.

**Commit.** Claude will offer to commit when implementation is complete. Review the diff. If you have been doing this well, the diff should be unsurprising. Commit frequently. Small commits are cheap insurance. The checkpoint-and-rollback strategy (discussed later in this chapter) depends on having clean commit boundaries to revert to.

This four-phase workflow is not about being cautious. It is about being efficient. Exploration and planning are cheap. Implementation and debugging are expensive. Front-loading cheap work to reduce expensive work is not caution. It is engineering.

The corollary: if you could describe the diff in one sentence, skip the plan. Renaming a variable does not need a design review. Plan mode earns its overhead when you are uncertain about the approach, the change spans multiple files, or the code is unfamiliar.

### First-Stop Workflow Planning

One team at Anthropic adopted a practice that sounds trivially simple but changed how they worked: they made Claude Code their first stop for every task. Before reading code manually, before searching through the repository, before asking a colleague -- they opened Claude Code and asked it to identify which files to examine for whatever they were working on. Bug fix, feature development, analysis -- it did not matter.

This replaced the traditional time-consuming process of manually navigating the codebase and gathering context before starting work. Claude could scan the entire repository structure, identify relevant files, explain complex interactions between modules, and surface dependencies that a human would need fifteen minutes of directory browsing to find. The team reported that this single habit -- always starting with Claude Code rather than treating it as an optional accelerator -- was the biggest shift in their daily workflow.

## Permission Modes Are Workflow Selectors

Most people think of permission modes as a safety dial. Crank it up for danger, crank it down for trust. That framing misses the point.

Permission modes shape how work happens. They are workflow selectors, not just guardrails.

**Plan mode** keeps Claude researching and proposing instead of editing. Use it for exploration and research. You are not being cautious; you are being intentional about which phase of the workflow you are in.

**Manual mode** (the config value is still `default`) asks before anything beyond a read. Use it for surgical changes where you want to approve every edit and command.

**Accept-edits mode** lets Claude write files and run common filesystem commands inside the working directory without asking, but still prompts for other shell commands -- and, notably, still prompts before writing files that can execute code. This is the sweet spot when you trust the plan but want to supervise system-level operations.

**Auto mode** is the one that changed everything, and it is now where an interactive terminal or editor session starts by default. Instead of you approving each action, a separate classifier model reviews it in the background. Routine work runs without interruption; anything that escalates beyond what you asked for, touches infrastructure the classifier does not recognize, or looks like it was steered by hostile content in a file or web page gets blocked. It is the middle ground that used to not exist: between clicking "approve" two hundred times a day and turning every check off.

**Don't-ask mode** is for locked-down scripts: anything that would prompt is simply denied. **Bypass mode** turns the checks off entirely and belongs only in disposable containers. The permission model (Chapter 2) governs the full scope of what each mode allows.

Cycling between these modes mid-session with Shift+Tab is not a sign of indecision. It is how experienced users move through the explore-plan-implement-commit workflow without starting new sessions. You explore in plan mode, review the plan, approve it into auto mode, and let Claude execute.

### Autonomous Loops and the 80% Handoff

The product development team at Anthropic took the hands-off workflow further. Their engineers set up autonomous loops where Claude writes code, runs the test suite, reads the failures, and iterates continuously. They hand Claude abstract problems they are not familiar with, let it work autonomously, and then review the result when it reaches roughly eighty percent completion. The last twenty percent -- the judgment calls, the design refinements, the edge cases that require domain knowledge -- they finish themselves.

What used to take a hand-rolled script is now two built-ins. Auto mode removes the per-action prompts. The `/goal` command removes the per-turn prompts: you state a completion condition -- "all tests in the auth package pass and lint is clean" -- and a separate evaluator model checks it after every turn, sending Claude back to work until the condition holds or it judges the goal impossible. Chapter 8 covers how to write conditions that hold up across many turns.

This workflow depends on two things: starting from a clean git state so you can revert the entire run if it goes sideways, and committing as Claude goes so you have intermediate checkpoints. The security engineering team at Anthropic distilled this into a single directive they give Claude at the start of autonomous sessions: "commit your work as you go." Instead of asking targeted questions for code snippets, they let Claude work autonomously with periodic check-ins, and the incremental commits create a trail they can review, cherry-pick, or revert as needed.

## The Collaboration Paradox

Here is the uncomfortable number: research shows developers use AI-assisted tools in roughly sixty percent of their work. The fraction they can fully delegate -- hand off the problem and walk away -- is between zero and twenty percent.

This is not a failure of the tooling. It is a feature of the work.

Software engineering is a judgment-intensive activity. Even when Claude Code handles the implementation, a human still needs to define the problem correctly, evaluate whether the solution is appropriate, decide what tradeoffs are acceptable, and catch the cases where technically correct code is strategically wrong. The sixty percent usage figure means Claude is present and useful for most of the day. It does not mean sixty percent of the work is automated. AI serves as a constant collaborator, but using it effectively requires thoughtful setup, active supervision, validation, and human judgment -- especially for high-stakes work.

The practitioners who struggle most are the ones who expect full delegation and get frustrated when it does not materialize. The practitioners who thrive are the ones who embrace the iterative partner model: Claude proposes, you react, Claude adjusts, you approve. It is a conversation, not a handoff.

This applies even within a single task. You might delegate the initial implementation, then supervise the debugging, then take over for a tricky edge case, then hand it back for test coverage. The boundary between human work and machine work is not a clean line. It is a continuous negotiation, and the negotiation itself is part of the work.

## The Thought Partner Model

There is another way to think about what Claude Code does, one that extends beyond writing code. Consider the experience of someone who used Claude Code not to build software but to build a financial plan. They fed it spreadsheet exports from multiple accounts, described their goals and constraints, and worked with it iteratively over several sessions to produce a phased optimization plan -- complete with tax impact analysis, fund recommendations, and before-and-after allocation grids. The kind of deliverable you would pay a professional advisor to produce.

The interaction model was not "type a question, get an answer." It was more like working with an analyst who has all the data but has not lived with the accounts. You clarify, reframe constraints, and Claude updates every calculation, table, and recommendation in real time. It is iterative thought partnership applied to analysis, not just code.

This model -- Claude as iterative analyst, not one-shot oracle -- applies to any domain where the work involves processing data, applying constraints, and refining outputs through feedback. Code is the most common use case, but the mental model transfers directly to data analysis, technical writing, infrastructure planning, and anything else where the quality of the output depends on the quality of the dialogue.

The key insight: the best practitioners give Claude a rough proposal as a starting point rather than a blank slate. A sketch of the approach, a strawman architecture, even a half-formed idea. Claude refines a proposal faster and more accurately than it generates one from nothing. If you have an opinion, state it. If you have a suspicion, share it. The output quality tracks directly to the specificity of the input.

## The 70-80% Coverage Gap

Claude Code can build roughly 70-80% of a production stack today. The remaining 20-30% is where experienced developers earn their keep.

### Component-Level Assessment

The mistake developers make is thinking about this gap in the abstract. It is not abstract. It is component-specific, and the fit varies dramatically depending on what you are building.

Consider a production application with typical frontend, backend, and infrastructure layers. At the component level, the picture looks something like this:

**Where Claude Code excels (build now, ship to production):**
- Frontend scaffolding -- component architecture, forms, dashboards, state management. Standard patterns with well-established conventions.
- API scaffolding -- CRUD endpoints, route definitions, middleware, request validation. The patterns are predictable and verifiable.
- Database schema design -- migrations, models, indexing strategies, relationship definitions. Claude generates schemas with proper constraints and immutability patterns.
- Test generation -- unit tests, integration tests, test fixtures. Clear success criteria make this ideal for the agentic loop.
- DevOps configuration -- container definitions, CI pipeline definitions, deployment scripts. Template-driven work with verifiable outputs.
- Documentation -- API docs, README files, code comments, changelog entries. Claude reads the code and describes what it does.

**Where Claude Code needs a human partner (hybrid approach):**
- Real-time systems with connection management -- Claude scaffolds the structure, but you review connection lifecycle, backpressure handling, and failover logic.
- Performance-critical paths -- Claude generates correct code, but optimizing for tight latency budgets requires measurement and iteration that benefits from human judgment.
- Security-sensitive logic -- authentication flows, encryption, access control. Claude handles the boilerplate, but the security review is yours.
- Domain-specific protocols and standards -- specialized protocols require domain expertise that Claude may approximate but not guarantee.

**Where to proceed with caution:**
- Novel algorithms without established patterns. Claude will produce something, but without reference implementations it may be subtly wrong.
- Ultra-low-latency components where microseconds matter. Claude does not profile. You do.
- Anything where the failure mode is "it works but it is wrong in a way that is expensive to discover later."

### The Quick Decision Matrix

When evaluating whether to build a specific component with Claude Code, run through this checklist:

- **Does it have clear success criteria?** Tests pass, types check, linter is happy? Build it now.
- **Does it require real-time data handling?** Claude builds the component; you own the connection management.
- **Is the latency budget tight?** Claude scaffolds; you profile and optimize.
- **Is there an established pattern?** Claude excels. No established pattern? Proceed carefully.
- **Is correctness non-negotiable and hard to verify?** Human review is mandatory, not optional.

Knowing which category you are looking at is itself a skill. The developers who build intuition for this classification ship faster because they do not waste time fighting Claude on tasks where human judgment is non-negotiable, and they do not waste time hand-writing code that Claude could generate in seconds.

### Build Now vs. Wait For

The first edition of this book had a "wait for it" list: native CI auto-fix, issue-to-pull-request automation, visual dashboards for parallel agents. Every item on it shipped within months. That is the real lesson of this section -- the line between "hybrid" and "native" moves fast enough that a workaround you build today is likely to be a built-in next quarter.

**Build with Claude Code right now:** Repository-wide refactors, API scaffolding, test generation, database migrations, deployment configurations, documentation, component architecture, and any task where verification is automated. Add to that list what used to be gaps: browser-based end-to-end verification (Claude drives your real, logged-in browser through the browser extension, or launches and pokes your app with `/run` and `/verify`), pull requests that fix their own CI failures and review comments (`/autofix-pr` hands a branch to a cloud session that watches it), automated review on every pull request, and dispatching and monitoring a dozen parallel sessions from one screen (`claude agents`, covered in Chapter 4).

**Use a hybrid toolchain for now:** Inline code suggestions while typing (complement Claude Code with an editor-integrated autocomplete tool), GUI-only workflows on platforms where computer use is not available, and anything where the verification step requires a human's eyes on production data.

**Treat as experimental:** Agent teams (still behind an environment variable), agent view and cloud scheduled tasks (research previews), and anything labeled preview in the changelog. They work, and some are excellent, but their interfaces and limits are still moving.

The developers who ship fastest are the ones who use Claude Code aggressively for what it handles well today, complement it with other tools for current gaps, and check the release notes (`/release-notes`) before building a fragile workaround for something that may already be a native feature.

## Task Classification: Async vs. Sync

Not every task deserves the same level of attention. Experienced teams develop an intuition for which tasks to run asynchronously -- fire and forget -- and which require synchronous supervision.

**Async candidates.** Test generation for existing code. Documentation updates. Boilerplate scaffolding. Migration scripts with clear patterns. Dependency updates. Code formatting and lint fixes. These tasks have unambiguous success criteria and low risk of subtle errors. Dispatch them as background sessions in auto mode and check the result when they are done -- `claude --bg "..."` returns your terminal immediately, and agent view (Chapter 4) tells you which ones finished, failed, or need you.

**Sync candidates.** Core business logic. Security-sensitive changes. Architectural decisions. Performance optimizations. Anything where the failure mode is silent -- code that runs but produces wrong results. These tasks need you watching, reacting, and course-correcting in real time.

### The Product Team's Classification Rubric

The product development team at Anthropic formalized this intuition into a rubric. Tasks on the product's periphery -- a new settings panel, a prototype feature, a peripheral UI component -- run unattended. Abstract problems the developer is unfamiliar with are also good candidates, because Claude's exploratory attempts in autonomous mode often surface the right approach faster than human speculation.

Tasks touching core business logic, critical user-facing features, or anything where style guide compliance and architectural consistency matter? Synchronous supervision. The developer watches, interrupts when needed, and keeps Claude aligned with the broader design intent.

The classification is not fixed. A task that starts async might become sync when Claude encounters an unexpected edge case. A task that starts sync might become async once you are satisfied with the approach and just need it applied across many files. The ability to toggle between these modes -- both in your own attention and in Claude's permission settings -- is what makes the workflow fluid.

## Try One-Shot First, Then Collaborate

The reinforcement learning engineering team at Anthropic distilled their workflow into a simple escalation pattern: give Claude a quick prompt and let it attempt the full implementation first. If it works -- about one-third of the time -- you have saved significant time. If it does not, switch to a more collaborative, guided approach.

This sounds obvious, but the default instinct for many developers is the opposite: over-specify the prompt, try to anticipate every edge case, and front-load excessive detail. The one-shot-then-collaborate pattern works better because it lets you discover what Claude actually struggles with rather than guessing. The first attempt produces data -- you see where Claude's understanding breaks down, and your subsequent guidance is targeted instead of speculative.

The one-third success rate is not a quality metric. It is a feature of the workflow. When the one-shot succeeds, you saved thirty minutes of planning. When it fails, you spent two minutes and gained information about where to focus your collaboration. The expected value is strongly positive in both cases.

## Session Strategy

Sessions are not persistent environments. Each new session starts with a fresh context window. The conversation, the files Claude read, the dead ends it explored -- gone.

Not quite everything, anymore. Auto memory (Chapter 3) means Claude now keeps its own notes per repository -- your corrections, your preferences, the build quirk it tripped over last Tuesday -- and loads the index of those notes at the start of every session. But notes are not context. A session still begins without the working state of the last one, and that is a design constraint that shapes how you should work.

**Continue.** Resume the last session in the current directory (`claude -c`). Use this when you were interrupted and need to pick up where you left off. The full conversation history is restored into the context window.

**Resume by ID or name.** Useful for long-running projects with multiple parallel workstreams. Name sessions with `-n` at launch or `/rename` mid-session, then `claude -r auth-refactor` gets you back. Paste a pull request URL into `/resume` and it finds the session that created that PR.

**Branch.** `/branch` splits the conversation at this point and switches you into the copy; the original stays intact and reachable from `/resume`. Use it when you want to try an alternative approach without losing your current progress. Its sibling, `/fork`, copies the conversation into a new *background* session that works on something else while you keep going here.

**Fresh session.** Start clean. This is often the right choice when your previous session's context is mostly spent on exploration that is no longer relevant. Persistent knowledge goes in CLAUDE.md (Chapter 3) and in your project's task system, not in session history. And if you `/clear` and immediately regret it, the rewind menu offers the pre-clear conversation as its top entry for as long as the process stays open.

The key insight is that session management is context management. A session that has been running for hours is a session with a full context window, which means degraded performance. Starting fresh with a good CLAUDE.md file is often faster than continuing a bloated session, because Claude gets your critical instructions at full fidelity instead of through the haze of auto-compacted summaries.

## The Iterative Partner Model

One-shot expectations are the root of most frustration with Claude Code. You type a detailed prompt, expect perfect output, and feel disappointed when the result is 80% correct instead of 100%.

Flip the expectation. Plan for iteration.

The first pass is a draft. Review it. Tell Claude what is wrong. Tell Claude what is right. Give it the failing test output. Show it the linter errors. The agentic loop is designed for this. Each correction sharpens Claude's understanding of what you actually want, and because the conversation history is in the context window, Claude does not repeat its mistakes within a session.

Multi-session iteration works the same way, just with CLAUDE.md and auto memory as the persistence layers instead of conversation history. Auto memory captures the small stuff on its own. The rules you want enforced for the whole team still need a human decision: after a productive session, ask Claude to summarize what it learned and suggest updates to your CLAUDE.md file. The next session starts smarter. Over five or ten sessions, the CLAUDE.md file accumulates enough project-specific knowledge that first-pass quality improves dramatically.

Self-critique prompting (Chapter 8) accelerates this loop further -- asking Claude to evaluate its own output surfaces issues the initial generation missed.

The iterative model also changes how you think about failure. A wrong first attempt is not a failure. It is data. It tells Claude something about what you want that your original prompt did not. The developers who internalize this stop feeling frustrated by imperfect output and start feeling productive because each iteration is fast and each one gets closer.

## Commit Frequently, Revert Without Hesitation

The single most important operational practice for working with Claude Code is this: start from a clean git state and commit frequently. Small commits are cheap insurance that give you rollback points when Claude goes down the wrong path.

Why? Because reverting is cheaper than correcting. When Claude makes a wrong turn, your instinct is to explain the problem and ask it to fix its own work. Often, this makes things worse -- each correction attempt consumes context, accelerating degradation. A revert to the last good commit and a fresh prompt is almost always faster. Chapter 10 covers the full checkpoint-and-rollback recovery strategy and the "slot machine" workflow that teams use to formalize this practice.

## One-Third Reality

Roughly one-third of tasks succeed on the first attempt without additional guidance. This is not a quality problem -- it is the expected behavior of an iterative system. The correction-and-retry cycle is fast, and the first-attempt success rate improves as your project's verification infrastructure (tests, types, linting) matures. Chapter 10 covers this statistic in depth and its implications for how you should structure your workflow.

## Prompt Suggestions: The Hints You're Ignoring

After Claude finishes a response, grayed-out follow-up suggestions appear below the output. Most developers ignore them. That is a mistake.

These suggestions are not random. They are generated based on Claude's assessment of what the logical next step should be, given the current conversation context and the state of the codebase. They surface actions you probably want but might not think to ask for: running the tests after a refactor, checking for edge cases after an implementation, updating related files after a change to a shared interface.

The suggestions are particularly valuable early in a session, when you are still orienting to the problem. They function as a lightweight checklist of things Claude noticed but did not act on unprompted. Hitting Tab to accept a suggestion is faster than formulating the next prompt yourself, and the suggestion is often better-scoped than what you would have typed because Claude has the full context of what it just did.

The anti-pattern is accepting suggestions blindly. They are hints, not orders. Read them. If the suggestion matches what you would have done next, accept it. If it does not, ignore it and direct Claude yourself. The value is in the time saved when the suggestion is right, not in deferring your judgment entirely.

## When Simplicity Beats Sophistication

Claude has a tendency toward over-engineered solutions -- a failure mode covered in detail in Chapter 10. The short version: tell Claude to keep it simple, explicitly constrain it toward straightforward implementations, and put simplicity directives in your CLAUDE.md file.

This connects to a broader principle: Claude Code responds to constraints better than it responds to freedom. A vague prompt produces vague output. A prompt with specific constraints -- which files to touch, which patterns to follow, which tradeoffs to make, which frameworks to avoid -- produces focused, appropriate code. More on this in the prompt craft chapter (Chapter 8).

---

## Key Takeaways

- Claude Code is an agentic loop (read-plan-act-verify) built on a pattern so simple it fits in 350 lines -- but wrapped in production-grade safety, permissions, and context management that makes the difference.
- The context window is the single most important constraint -- every file read, command output, and message consumes finite space, and performance degrades sharply when it fills.
- The explore-plan-implement-commit workflow front-loads cheap work (exploration, planning) to reduce expensive work (debugging, reverting).
- Permission modes are workflow selectors: plan mode for exploration, auto mode (now the interactive default) for trusted implementation, manual for surgical changes. Shift+Tab cycles between them mid-session.
- A million-token window moves the context wall; it does not remove it. Treat context as scarce anyway.
- Research shows developers can fully delegate only 0-20% of tasks; the real value is iterative collaboration, not fire-and-forget automation.
- Assess Claude Code fit at the component level: frontend scaffolding and test generation are immediate wins; real-time systems and security-critical logic need human partnership.
- Try one-shot first, then collaborate -- the one-third immediate success rate is a feature, not a bug, and failed first attempts produce the information you need for targeted guidance.
- Commit frequently and revert without hesitation; reverting to a clean state and retrying is almost always faster than patching a wrong approach.
- Explicitly constrain Claude toward simplicity; without constraints, it defaults to over-engineered solutions.
