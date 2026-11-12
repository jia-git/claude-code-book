# Chapter 6: CI/CD and Headless Automation

## What You'll Learn

Claude Code in a terminal with a human watching is one tool. Claude Code running unattended in a CI pipeline, processing tickets at 3 AM, or fixing broken builds before anyone wakes up is a different category of tool entirely. The shift from interactive to headless is not just a flag you pass. It changes the permission model, the output format, the failure recovery strategy, and the economics of what you can automate.

This chapter covers headless mode and everything that surrounds it: the flags that make Claude Code non-interactive, the output formats that make it composable with Unix tooling, the container strategies that make it safe to run unattended, and the integration patterns that connect it to real CI/CD systems. You will learn how to write concrete CI workflow definitions that run Claude Code as a first-class pipeline step, how hooks create automated quality gates that block bad code before it ships, and how the fan-out pattern lets you batch-process entire repositories file by file. You will also learn which of the first edition's "coming soon" gaps have closed -- pull requests that fix their own CI failures, review on every pull request, prompts that run on a schedule in the cloud -- and how to decide between scripting Claude Code yourself and using the built-in automation.

The developers getting the most value from Claude Code are not the ones typing the cleverest prompts. They are the ones who figured out that Claude Code can work while they sleep.

---

## Headless Mode

The `-p` flag transforms Claude Code from an interactive REPL into a non-interactive command processor. You pass a prompt, Claude executes it, and the result comes back on stdout. No terminal UI. No waiting for human input. The documentation now calls this "the Agent SDK via the CLI," which is a useful reframing: `claude -p` is the same agent loop that the Python and TypeScript SDK packages expose as a library, and anything you prototype in a shell script can graduate to a program later.

```
claude -p "Refactor the auth module to use dependency injection"
```

That is the simplest form. In practice, headless mode supports a set of flags that control its behavior precisely:

- `--output-format` selects between `text` (human-readable), `json` (structured, machine-parseable, including the estimated cost of the run), and `stream-json` (newline-delimited JSON objects emitted as work progresses).
- `--json-schema` makes the run return validated JSON matching a schema you supply, in a `structured_output` field. Check that the field exists before using it: a run that declines or fails has none.
- `--max-turns` caps the number of agentic loop iterations, preventing runaway execution.
- `--max-budget-usd` stops the run once estimated spend reaches a dollar amount, subagents included. Leave headroom; spend can overshoot slightly.
- `--model` and `--effort` select the model and effort level for the run.
- `--bare` skips auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and CLAUDE.md, so the run starts faster and behaves the same on every machine. It is the recommended mode for scripts and is slated to become the default for `-p`. Pass what you need explicitly with `--settings`, `--mcp-config`, `--agents`, and `--append-system-prompt-file`. Bare mode authenticates with an API key, not your subscription login.

Two behaviors surprise people coming from the first edition. First, a `-p` run now starts in Manual permission mode unless you say otherwise: anything that would need approval is denied, not silently allowed. Set the baseline you want with `--permission-mode` (more on this below). Second, a `-p` run waits for background work Claude started -- subagents, workflows, background commands -- before exiting, so a script will not return with half the results. An idle wait is capped at ten minutes by default.

The `stream-json` format deserves particular attention. Each line is a self-contained JSON object representing one step of Claude's work: a tool call, a result, a thinking step, a final response. This format is designed for pipeline consumption -- you can pipe it into a monitoring system, a log aggregator, or another program that reacts to Claude's actions in real time.

Headless mode also accepts piped input. You can feed Claude the contents of a file, the output of another command, or any text stream:

```
git diff HEAD~1 | claude -p "Review this diff for security issues" --output-format json
```

This is not a convenience feature. It is what makes Claude Code a Unix citizen.

## Pipe In, Pipe Out

The most underappreciated pattern in headless mode is bidirectional piping. You can feed data into Claude Code from any source and redirect its output to any destination:

```
cat build-error.txt | claude -p 'concisely explain the root cause of this build error' > output.txt
```

That is a complete diagnostic workflow in one line. The build failed. The error log is in a file. Claude reads it, explains the root cause, and writes the explanation to `output.txt`. No interactive session. No copy-pasting. No context switching.

The output format flag controls what comes out the other end:

```
cat data.txt | claude -p 'summarize this data' --output-format text > summary.txt
cat code.py | claude -p 'analyze this code for bugs' --output-format json > analysis.json
cat log.txt | claude -p 'parse this log file for errors' --output-format stream-json
```

Text output is human-readable. JSON output is machine-parseable -- feed it into downstream tools, store it in a database, or pipe it into another command. Stream-JSON emits each step as it happens, which is useful when you want a monitoring system to react in real time.

The pipe pattern also works for build script integration. Add Claude Code as a task in your project's package configuration:

```json
{
  "scripts": {
    "lint:claude": "claude -p 'Review src/ for code style violations and suggest fixes' --output-format text",
    "review": "git diff main | claude -p 'Code review this diff' --output-format text"
  }
}
```

Now `npm run lint:claude` gives you an AI-powered lint pass as part of your normal workflow. `npm run review` produces a code review of your uncommitted changes. These are not special integrations. They are CLI commands in a build script, the same way you would add any other tool.

## Unix Composability

Claude Code's command-line interface follows the Unix philosophy: it reads from stdin, writes to stdout, and plays well with pipes, redirects, and other tools. This makes it composable in ways that GUI-based tools cannot match.

A few patterns that become possible:

**Chaining with analysis tools.** Pipe structured output into processing tools to extract specific fields, filter results, or transform formats:

```
claude -p "List all API endpoints in this project" --output-format json | \
  jq '.result' > endpoints.json
```

**Sequential processing.** Run Claude Code as one step in a larger pipeline:

```
find . -name "*.py" -mtime -7 | \
  claude -p "Summarize the recent changes in these files" --output-format text
```

**Build script integration.** Embed Claude Code directly in your project's build scripts or task runner configuration:

```
lint-ai:  claude -p 'Review src/ for code style violations and suggest fixes' --output-format text
review:   git diff main | claude -p 'Code review this diff' --output-format text
```

This turns Claude Code into a build step. Run your review script and get an AI code review as part of your normal workflow. The output is text you can read, JSON you can parse, or streaming JSON you can process incrementally.

The key insight is that headless mode does not need a special integration layer. It is already a CLI tool. Anything that can call a command and read its output can use Claude Code. This is not Claude Code borrowing Unix philosophy as a metaphor. Claude Code is a Unix utility -- one that happens to have a language model as its processing engine instead of a regex engine or a text formatter.

## Devcontainers for CI

Running Claude Code in CI raises an immediate question: how do you give it permission to modify files and run commands without a human approving each action?

The answer is development containers. A devcontainer provides an isolated environment where Claude Code can operate freely because the blast radius is contained. The approach works in three layers.

**The reference devcontainer.** Claude Code ships with a reference devcontainer configuration that mirrors typical development environments. It includes the language runtimes, build tools, and dependencies your project needs. The container is ephemeral -- it is created for the CI run and destroyed afterward.

**Network isolation.** The reference devcontainer applies a default-deny firewall policy. Claude Code can only access explicitly allowlisted domains. This prevents data exfiltration and limits the damage from unexpected behavior. The network policy is configured in the devcontainer definition, not in Claude Code itself, which means it is enforced at the OS level.

**The permissions escape hatch.** Inside a network-isolated container, you can safely pass `--dangerously-skip-permissions`. This flag bypasses all permission prompts, allowing Claude Code to run autonomously without human approval. The flag name is deliberately alarming -- you should only use it when the container itself provides the security boundary.

This three-layer model -- isolated container, restricted network, autonomous permissions -- is the standard pattern for CI integration. It trades Claude Code's built-in permission system for container-level isolation, which is a better fit for unattended operation.

One caveat: container isolation cannot prevent credential exfiltration if your CI environment injects secrets into the container. A malicious or confused Claude Code instance that has access to environment variables containing API keys could theoretically send them to an allowed domain. Scope your secrets carefully. Prefer short-lived tokens over long-lived credentials.

## Permission Handling for Unattended Operation

Not every headless deployment uses containers. Some run directly on CI hosts or in environments where container isolation is not available. For these, pick a baseline permission mode and then refine it:

- **`--permission-mode dontAsk`** denies everything that would prompt, while still running reads, the read-only command set, and anything your `--allowedTools` or `permissions.allow` rules cover. This is the locked-down CI default: explicit allowlist, nothing else.
- **`--permission-mode auto`** lets the auto-mode classifier (Chapter 2) review each action. Blocked actions are skipped and the run continues. Note that auto mode drops broad allow entries like a bare `Bash`, so the classifier still sees every command.
- **`--permission-mode acceptEdits`** lets Claude write files and run common filesystem commands, while other shell commands still need allow rules.

Add **`--permission-prompts none`** when nobody can answer a prompt. Anything that falls through to a prompt is denied, Claude is told not to retry, and tools that need a human -- such as asking you a clarifying question -- are removed entirely. Denials are reported in the stream as `permission_denied` events and listed in the final result, which makes a CI log far easier to audit.

For policy that is more than an allowlist, Claude Code provides `--permission-prompt-tool`, which delegates permission decisions to an MCP tool. A `PermissionRequest` hook achieves the same thing without an MCP server.

When Claude Code encounters an action that would normally require human approval, instead of prompting a user (who is not there), it calls the specified MCP tool with the details of the requested action. The tool decides whether to approve, deny, or modify the request.

The real value is that it lets you encode your permission policy as code. The MCP tool can implement any logic: approve all file edits within the `src/` directory but deny edits to `config/`, approve test execution but deny deployment commands, approve everything during off-hours but require stricter controls during business hours.

The alternative -- running with `--dangerously-skip-permissions` outside a container -- is exactly as dangerous as the flag name suggests. Use it only in properly isolated environments.

## System Prompt Flags for Reproducibility

Interactive Claude Code sessions adapt to conversation flow. Headless sessions need reproducibility. Four flags provide it, each with a distinct use case:

`--system-prompt` passes the system prompt as an inline string. This is useful for quick scripting but does not scale -- prompts embedded in CI scripts tend to drift, diverge, and degrade without anyone noticing.

`--system-prompt-file` replaces Claude Code's default system prompt with the contents of a file. This gives you complete control over Claude's behavior -- its personality, its constraints, its focus areas. Version-control this file alongside your CI configuration and you get deterministic prompt behavior across runs. Use this when you need a purpose-built prompt for a specific CI task and do not want Claude Code's default behaviors.

`--append-system-prompt` adds an inline string to the default system prompt rather than replacing it. You keep Claude Code's built-in behaviors and add instructions on top. Good for one-off modifications.

`--append-system-prompt-file` does the same, but reads from a file. This is the most common choice for CI pipelines -- you keep Claude Code's defaults, add project-specific instructions from a version-controlled file, and get reproducibility without losing built-in capabilities.

The decision between replacing and appending is straightforward: replace when you want total control (automated analysis tasks with specific output formats), append when Claude Code's default behaviors are useful and you just need to add constraints or focus areas on top.

All four flags work in both interactive and print mode, and the inline and file forms can now be combined. The file-based variants are always preferred for CI because they produce reviewable, version-controlled prompt definitions instead of inline strings buried in workflow YAML. One subtlety: the system prompt is recorded on a conversation's first request and reused when you `--continue` it, so a changed prompt file does not take effect mid-conversation. Pass `--system-prompt-snapshot off` while you iterate on the wording.

Two flags matter for cost at scale. A replacement prompt that mixes fixed instructions with per-run context can include a line containing only `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`; everything above it stays cached across runs. And `--exclude-dynamic-system-prompt-sections` moves per-user details out of the default system prompt, so the same task run by many users on many machines shares one prompt cache.

For CI pipelines that run repeatedly, prompt files combined with CLAUDE.md provide two layers of reproducible context: the system prompt governs Claude's behavior for this specific CI task, while CLAUDE.md provides project-wide knowledge.

## CI Platform Integration Patterns

Major version control platforms now offer first-class CI integrations for Claude Code. The two most mature are the built-in CI workflow systems of the dominant code hosting platform and a major alternative platform's pipeline system, both of which can also route through the major cloud providers' model endpoints instead of Anthropic's API. Both treat Claude Code as a native CI actor, not a bolted-on afterthought. On the dominant platform, a single command inside Claude Code installs the integration's app for a repository and sets up the workflow and secrets. Three patterns have proven effective in production.

**Automated PR fixes.** A workflow triggers when a PR receives a review comment requesting changes. Claude Code reads the comment, checks out the branch, makes the requested changes, and pushes a new commit. The human reviewer's feedback becomes a prompt that drives automated implementation.

The workflow looks like this: a webhook fires on PR comment, the CI job starts a devcontainer, checks out the PR branch, pipes the review comment to Claude Code with context about the repository, and pushes the resulting changes. The reviewer sees a new commit addressing their feedback without the PR author doing anything.

**Automated ticketing.** A workflow triggers when a new issue is created with a specific label. Claude Code reads the issue description, creates a branch, implements a solution, and opens a PR. The issue becomes the prompt. The PR becomes the deliverable.

One design team discovered an especially elegant variant of this pattern. Designers file issues describing UI polish tasks -- spacing adjustments, color corrections, animation tweaks -- and a CI workflow automatically proposes code changes without anyone opening Claude Code. The designer reviews the resulting PR, requests adjustments through PR comments (which trigger another automated Claude Code pass), and merges when satisfied. Design polish, which used to require an engineer to context-switch away from feature work, now flows through the same ticket-to-PR pipeline as any other task.

**Automated code review.** Every PR triggers a headless Claude Code session that reads the diff, checks it against project conventions encoded in CLAUDE.md, and posts review comments. This is not a replacement for human review. It is a first pass that catches style violations, missing tests, and obvious bugs before a human reviewer looks at the code. The human reviewer sees a cleaner PR and spends their time on architectural and design questions rather than formatting and convention enforcement. You can still build this yourself in a workflow -- but for most teams the managed review service described below is now the better starting point.

All three patterns follow the same structure: event trigger, context assembly, headless Claude Code execution, result publication. The event provides the prompt. The repository provides the context. Claude Code provides the implementation. The CI system provides the orchestration.

These patterns work today, but they work best for well-defined, bounded tasks. A review comment saying "add null checking to this function" produces reliable results. A ticket saying "redesign the authentication system" does not. The scope of the task must fit within a single headless session. For automated workflows, constrain the task scope explicitly in the system prompt file: specify what Claude Code should attempt, what it should flag for human review, and what it should leave untouched.

### The /commit-push-pr Skill

For interactive workflows that end in a pull request, the `/commit-push-pr` skill -- part of a commit-commands plugin in Anthropic's official marketplace -- eliminates the three-step dance of committing, pushing, and opening a PR. One command does all three. If you have a team chat MCP server configured and have specified notification channels in your CLAUDE.md (for example, "post PR URLs to #team-prs"), the skill also posts the PR link to the team channel automatically. The workflow shrinks from commit-push-open-browser-fill-in-PR-description-post-to-chat to a single command that handles the entire chain.

### The --from-pr Flag

When a PR is created through Claude Code (either via `/commit-push-pr` or by asking Claude to create a PR using the platform CLI), the session is automatically linked to that PR. Later, you can find it again with:

```
claude --from-pr 123
```

This opens the session picker filtered to sessions linked to that pull request -- by number, or by a PR or merge-request URL from any of the major hosting platforms. Pasting the PR URL into `/resume` does the same thing. Pick the session and you restore the full conversation context that produced the PR. You pick up exactly where Claude left off -- same understanding of the codebase, same design decisions, same constraints. This is invaluable for PR review cycles where a reviewer requests changes days after the original implementation.

### Messaging Integration

Claude Code integrates with team messaging platforms both through MCP connectors and through a first-party chat app. The most common pattern routes bug reports from a team chat directly to pull requests. A team member mentions Claude in a channel with a bug description. A cloud session reads the message, creates a branch, implements a fix, opens a PR, and posts the PR link back to the channel. The entire workflow -- from bug report to PR -- happens without anyone opening an IDE. For organizations, Claude Tag runs that mention as a shared team identity with administrator-configured access, rather than as each person's individual account.

This transforms team messaging from a coordination tool into a dispatch system. The bug report becomes the prompt. The codebase provides the context. Claude Code provides the implementation. The messaging platform closes the loop by posting results.

## The Fan-Out Pattern

One of the most powerful headless patterns is fan-out: looping through a set of files and calling Claude Code once per file with scoped permissions. This is batch processing with AI.

```bash
for file in $(find src -name '*.ts' -type f); do
  claude --bare -p "Review $file for security vulnerabilities" \
    --tools "Read,Bash" \
    --permission-mode dontAsk \
    --output-format json >> reviews.jsonl
done
```

The permission flags are the key. `--tools` restricts which built-in tools exist at all; `dontAsk` mode then denies anything that would need approval, so Claude can read files and run read-only search commands, but cannot modify anything. This converts a potentially dangerous batch operation into a safe analysis pass. (A note on search: on Linux and macOS, Claude now searches with `grep` and `find` through the shell rather than dedicated search tools, so read-only shell access is what an analysis pass needs.)

Fan-out scales well for certain tasks. Security auditing across hundreds of files. Documentation generation for each module. Test coverage analysis per component. Consistency checking against coding standards. Each invocation gets a fresh context, focused on a single file, with permissions scoped to exactly what the task requires.

The pattern composes with standard Unix tools. The output is newline-delimited JSON, which means you can pipe it into analysis tools, filter it, or aggregate it. A CI job that runs the fan-out, filters for high-severity findings, and opens issues for each one is entirely achievable with shell scripting and the CLI.

Before writing the loop, check whether a built-in does it. `/batch` decomposes one large change into five to thirty independent units and runs each in its own worktree-isolated subagent. A dynamic workflow (Chapter 4) does the same with verification built in and a progress view you can pause. The shell loop remains the right tool when you need each invocation to be a separate, independently retryable CI step.

## CI Pipeline Generation

Claude Code is effective at generating CI configuration. The combination of reading your project's structure, understanding build tools, and producing valid YAML (or equivalent) means it can generate working pipelines from a description of what you want.

This includes CI workflow definitions, container image configurations, infrastructure-as-code templates, and security scanning setups. Claude Code reads your project's dependencies, test configuration, and deployment targets, then generates pipeline definitions that match.

The reason this works well is that CI configuration is high-structure, low-ambiguity work. The schemas are well-defined. The conventions are stable. The output is testable -- you can validate the generated YAML against the CI system's schema before committing it.

Here is a concrete example of what Claude Code generates when you point it at a typical project and ask for a CI/CD pipeline:

```yaml
name: CI/CD Pipeline
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install dependencies
        run: npm ci
      - name: Lint
        run: npm run lint
      - name: Type check
        run: npm run type-check
      - name: Unit tests
        run: npm test
      - name: Build
        run: npm run build

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker image
        run: docker build -t myapp-api:${{ github.sha }} .
      - name: Push to registry
        run: docker push myapp-api:${{ github.sha }}
      - name: Deploy to staging
        run: |
          kubectl set image deployment/app-api \
            api=myapp-api:${{ github.sha }} -n staging
```

Claude Code generates the entire structure -- lint, type-check, test, build, deploy stages with proper dependency ordering. You review, adjust credentials and environment specifics, and merge. The generated configuration is not a starting point that requires heavy modification. It is a working pipeline that reflects your project's actual build steps because Claude Code read your project configuration before generating it.

Where it gets interesting is iterative refinement. Generate an initial pipeline, run it, observe the failures, feed the error output back to Claude Code, and let it fix the configuration. This feedback loop converges on a working pipeline faster than manual debugging of YAML indentation errors and undocumented configuration options.

Consider a practical example. You have a monorepo with three services, each with its own test suite, container image, and deployment target. Manually writing the CI configuration -- conditional builds based on changed paths, parallel test execution, staged deployments with rollback gates -- is a full day of work. Claude Code reads the repository structure, identifies the service boundaries, and generates the complete pipeline configuration in a single session. The generated configuration handles the conditional logic, parallelism, and deployment ordering because Claude Code understands the project structure it just read.

Container image generation follows the same pattern. Claude Code reads your application code, identifies dependencies, selects appropriate base images, and produces multi-stage container build files with correct layer caching. For infrastructure-as-code tools, it generates resource definitions that match your described architecture.

### Security Scanning Integration

Security scanning configuration deserves special attention because it is one area where Claude Code generates the steps but you define the rules.

The pattern works across multiple scanning categories. Static application security testing tools integrate directly into CI workflows -- Claude Code adds the scanning step, and results surface as comments on pull requests. Dependency auditing tools (for both package managers and language ecosystems) slot into the build stage as additional verification steps. Container image scanning tools run against the built image before it reaches the registry.

| Category | What Claude Generates | What You Define |
|---|---|---|
| Static analysis | CI workflow step invoking the scanner | Rulesets and severity thresholds |
| Dependency audit | Build-stage command for vulnerability checks | Allowlisted advisories and exception policy |
| Container scanning | Post-build scan step against the image | Accepted risk levels and base image policy |
| Runtime compliance | Custom check scripts | Compliance rules and enforcement policy |
| Secret detection | Use the platform's native capability | Repository-level settings |

The recommendation is to include security scanning in your CI workflow from the start. Claude Code generates the integration steps. The scanning tools provide the findings. The CI system blocks the merge if findings exceed your thresholds. Secret detection is the one exception -- the platform's native secret scanning is more reliable than a custom implementation.

The key insight: CI configuration is one of the highest-ROI uses of headless Claude Code because the output is immediately testable. You run the pipeline and it either works or it does not. There is no subjective judgment about quality. The feedback loop is tight and unambiguous.

## Hooks as CI Quality Gates

Hooks -- the lifecycle callbacks described in Chapter 2 -- transform from convenience features into critical infrastructure when Claude Code runs in CI. Three hook patterns are particularly valuable in automated workflows.

### Async Hooks: Background Test Runners

A `PostToolUse` hook with `"async": true` runs in the background without blocking Claude's execution. This is purpose-built for running test suites after Claude writes code. Claude continues working on the next file while the tests run in parallel. When the tests finish, the results are delivered to Claude on the next conversation turn.

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/run-tests-async.sh",
        "async": true,
        "timeout": 300
      }]
    }]
  }
}
```

The test runner script reads the tool input from stdin, determines which test file corresponds to the modified source file, runs the tests, and emits a JSON response with a `systemMessage` field containing the results:

```bash
#!/bin/bash
INPUT=$(cat)
FILE=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

if [[ -z "$FILE" ]] || [[ ! "$FILE" =~ \.(ts|js)$ ]]; then
  exit 0
fi

TEST_FILE="${FILE%.ts}.test.ts"
if [[ ! -f "$TEST_FILE" ]]; then
  exit 0
fi

RESULT=$(npx jest "$TEST_FILE" --no-coverage 2>&1)
EXIT_CODE=$?

if [[ $EXIT_CODE -ne 0 ]]; then
  echo "{\"systemMessage\": \"Tests failed for $TEST_FILE:\\n$RESULT\"}"
fi
exit 0
```

Claude sees "Tests failed for..." on its next turn and self-corrects. The human never needs to intervene. The tests are the feedback loop.

### TaskCompleted Hook: Blocking Completion on Failure

The `TaskCompleted` hook fires when Claude marks a task as complete. Exit code 2 blocks the completion and sends feedback. This is your automated gatekeeper:

```bash
#!/bin/bash
INPUT=$(cat)
SUBJECT=$(echo "$INPUT" | jq -r '.task_subject // empty')

if ! npm test 2>&1; then
  echo "Tests are failing. Fix them before completing: $SUBJECT" >&2
  exit 2
fi

exit 0
```

Claude cannot mark the task as done until the tests pass. With exit code 2, whatever the script writes to stderr becomes Claude's feedback; it fixes the issues and tries again. This is especially powerful in multi-agent workflows where teammates complete tasks autonomously -- the hook ensures no task is marked "done" until the verification passes. (It only fires where the task tools are in use; on the newest models, those are opt-in, as Chapter 4 explains.)

### Agent-Based Stop Hook: Intelligent Verification

The most powerful hook variant is `type: "agent"`, still marked experimental. Instead of running a bash script, Claude Code spawns a subagent with tool access to evaluate whether Claude should stop. The subagent can investigate the codebase, run tests, and make a reasoned decision:

```json
{
  "hooks": {
    "Stop": [{
      "hooks": [{
        "type": "agent",
        "prompt": "Check if all tests pass by running the test suite. If any tests fail, respond with ok: false and explain which tests failed.",
        "timeout": 120
      }]
    }]
  }
}
```

The subagent runs the test suite, checks the results, and returns `{"ok": true}` or `{"ok": false, "reason": "..."}`. If it returns false, Claude does not stop -- it reads the failure reason and continues working. This is verification that is not just mechanical (did the exit code equal zero?) but contextual (did the tests pass and do the results make sense?).

### CLAUDE_ENV_FILE: Persisting Environment Variables

SessionStart hooks have access to the `CLAUDE_ENV_FILE` environment variable -- the path of a shell script that Claude Code runs before every subsequent Bash command in the session, in the same shell process. Write `export` statements to it and they are visible to every command. This solves a persistent problem in CI: ensuring that Claude Code's Bash commands run with the right environment.

```bash
#!/bin/bash
if [ -n "$CLAUDE_ENV_FILE" ]; then
  echo 'export NODE_ENV=production' >> "$CLAUDE_ENV_FILE"
  echo 'export DEBUG_LOG=true' >> "$CLAUDE_ENV_FILE"
  echo 'export PATH="$PATH:./node_modules/.bin"' >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

This is particularly useful for language version managers, virtual environments, and any CI-specific configuration that normally requires sourcing a setup script. The environment persists for the entire session without Claude Code needing to know about the setup process.

## Backpressure via Pre-Commit Hooks

Pre-commit hooks -- the kind that run before a git commit is finalized -- create an especially tight quality gate for autonomous Claude Code. Set up a pre-commit hook that runs your type checker, linter, and test suite:

```bash
# .husky/pre-commit
pnpm typecheck && pnpm lint && pnpm test-run
```

When Claude Code (or a subagent) runs `git commit`, the hook fires immediately. If the type checker finds errors, the linter flags violations, or the tests fail, the commit is rejected. Claude sees the error output and self-corrects. This is automated feedback at the moment of truth -- the commit boundary -- and it catches issues at the source rather than accumulating bugs across multiple commits.

The implication for CI is significant. You do not need to build a custom verification system. Your existing pre-commit hooks become the quality gate for autonomous operation. Every commit Claude makes passes through the same checks as every commit a human makes. You stop being the bottleneck for quality control.

## Attribution Settings

Claude Code adds attribution to git commits and pull requests by default. In CI workflows, you may want to customize or disable this behavior. The `attribution` settings control it:

```json
{
  "attribution": {
    "commit": "Generated with AI\n\nCo-Authored-By: AI <ai@example.com>",
    "pr": ""
  }
}
```

Setting `commit` customizes the trailer added to commit messages. Setting `pr` controls the text appended to pull request descriptions. An empty string disables the attribution for that context. In automated workflows where every commit is AI-generated, you might want concise attribution rather than the default format, or you might want to disable PR attribution entirely to keep descriptions clean.

## Devcontainer Reference Setup

The reference devcontainer ships with three components: a container configuration file that controls settings, extensions, and volume mounts; a container image definition that installs the runtime and tooling; and a firewall initialization script that establishes network security rules.

The container image definition is worth studying even if you build your own. It starts from a base image, installs the target language runtime, sets up a shell with productivity enhancements, and installs Claude Code itself. The firewall script applies a default-deny network policy with explicit allowlisting for domains Claude Code needs to function (API endpoints, package registries). The container configuration ties everything together with volume mounts for session persistence and workspace settings.

The key design decision is that network isolation happens at the OS level via firewall rules, not at the application level. Claude Code does not know it is network-restricted. It simply finds that connections to non-allowlisted domains fail. This makes the security boundary robust against anything Claude Code might try to do -- the restriction is enforced below the application layer.

For teams that need consistent, reproducible environments across developers and CI, the reference devcontainer is the starting point. Clone it, customize the language runtime and dependencies for your stack, adjust the network allowlist, and you have a secure environment for both interactive development and headless CI operation.

## The CI Auto-Fix Gap, Closed

The first edition ended this chapter's integration section on a prediction: automatic CI failure remediation was "widely anticipated" and would likely arrive in mid-2026 as a first-party integration. It arrived roughly on schedule.

`/autofix-pr` hands the current branch's pull request to a cloud session that watches it. When CI fails or a reviewer leaves a comment, the session reads the failure or the comment, implements a fix, and pushes it. The same toggle is available from the web interface for any pull request a cloud session created. You go to lunch with a red build and come back to a green one -- and a commit history that shows exactly what was changed and why.

Treat it the way you would treat a capable new teammate with push access. Scope it to branches where an automated push is acceptable, keep branch protection on the default branch, and read the commits. The automation is the last mile; the review is still yours.

## Review on Every Pull Request

Review moved from pattern to product along three tiers, and it helps to know which one you are reaching for.

**`/code-review` in your session.** The bundled command reviews your branch's changes -- or a PR number, branch, path, or commit range you pass -- for correctness bugs, in a background subagent so it does not fill your conversation. Effort levels trade coverage for confidence: `low` reports only what it is most sure of; higher levels broaden coverage into reuse, simplification, and efficiency. `--fix` applies the findings; `--comment` posts them to the pull request. It runs on any plan and follows your CLAUDE.md. (`/review` is now an alias.)

**Ultrareview in the cloud.** `/code-review ultra` sends the branch to a fleet of bug-hunting agents in a cloud sandbox that find and then *verify* bugs before reporting them, so what lands back in your session has already survived a second look. `claude ultrareview <target>` runs the same thing non-interactively from CI, prints findings to stdout, and exits nonzero on failure; `--json` gives you the raw payload. It needs a Claude account and is not available on third-party model providers.

**The managed Code Review service.** For Team and Enterprise organizations, an administrator connects the repository host, and every pull request gets reviewed automatically -- findings verified, comments posted inline, severity calibrated. A `REVIEW.md` file in the repository holds review-only rules: what to skip, what to escalate, what to always check. Anyone can request a re-review with `@claude review`, or pause it with `@claude review stop`. Review levels and spend caps are set per organization or per repository.

The rule of thumb: `/code-review` while you work, ultrareview before you merge something risky, the managed service as the baseline for everyone.

## Running on a Schedule

The other first-edition gap was "Claude Code that works while you sleep" without a homegrown cron job. There are now three schedulers, and the choice comes down to where the work runs.

**`/loop` inside a session.** `/loop 5m check whether the deploy finished` re-runs a prompt on an interval while the session stays open. Omit the interval and Claude chooses the delay each time based on what it saw -- short waits while a build is running, longer ones when nothing is happening. Omit the prompt and it runs a built-in maintenance prompt: continue unfinished work, tend the current branch's pull request, then hunt for cleanup. A `loop.md` in the project replaces that default. Loops are session-scoped, expire after seven days, and every iteration re-sends your context -- so for "tell me when this log line appears," the Monitor tool, which streams a background command's output to Claude line by line, is cheaper and faster than polling.

**Scheduled tasks on your desktop.** The desktop app runs a saved prompt on a schedule on your machine, with access to local files and tools, as long as the machine is on.

**Scheduled tasks in the cloud** (formerly "routines"). A saved prompt, one or more repositories, and a set of connectors, run on Anthropic-managed infrastructure -- or your own, via a self-hosted environment -- on a schedule, on an HTTP call to the task's own endpoint, or on repository events such as a new pull request. Create them with `/schedule` from the CLI or from the web. They keep running with your laptop closed, which is the whole point. They are a research preview, and the minimum interval is an hour.

The decision rule: `/loop` for polling while you work, desktop tasks when the job needs your local machine, cloud tasks when it must not depend on it.

## Long-Running Headless Operation

Most headless usage assumes short-lived sessions: a CI job runs, completes in minutes, and terminates. But Claude Code's headless mode has been demonstrated running for dramatically longer periods.

One documented case involved Claude Code operating as an autonomous agent for 33 days. The system ran headless with occasional human check-ins, making decisions, executing actions, and maintaining coherent behavior across an extended timeframe. The human operator's role shifted from active direction to periodic supervision -- reviewing outcomes, adjusting parameters, and intervening only when the system drifted from its objectives.

This is not typical usage, and it surfaces challenges that short-lived sessions never encounter. Context compaction happens repeatedly over 33 days. The system must maintain consistent behavior across dozens of compaction cycles. Long-term memory must be externalized -- into files, databases, or structured storage -- because the context window is not a durable store.

Some of the scaffolding that experiment had to build by hand now ships. `/goal` keeps a session working across turns until an evaluator model confirms a stated condition (Chapter 8). An unattended retry mode keeps retrying through capacity errors and waits out usage-limit resets instead of failing the run. And a session interrupted by its host can be resumed with the interrupted turn continued rather than abandoned.

But the existence proof matters. Claude Code is not architecturally limited to short tasks. The `-p` flag, combined with structured persistence and periodic human oversight, supports extended autonomous operation. The constraint is not the tool. It is your ability to define clear objectives, provide adequate context, and build the scaffolding for long-term state management.

## Putting Headless to Work

The progression from interactive to headless follows a predictable path:

1. **Start interactive.** Use Claude Code normally. Observe what it does well with your codebase. Identify repetitive tasks.
2. **Script the repetitive tasks.** Wrap them in headless calls with `-p`. Test the output formats. Get comfortable with the CLI interface.
3. **Add to build scripts.** Embed headless calls in your project's task runner or build configuration. AI-powered linting, review, and documentation generation become part of your build.
4. **Move to CI.** Set up devcontainers. Use `--bare` and an explicit permission mode. Wire up triggers. Now Claude Code runs on events, not on your command.
5. **Close the loop.** Connect CI results back to Claude Code. PR comments trigger fixes. Failing builds trigger auto-fix. Issue labels trigger implementations. Scheduled tasks run the maintenance nobody remembers to do. The system becomes self-reinforcing.

Each step is independently valuable. You do not need to reach step five to benefit. But each step makes the next one obvious.

---

## Key Takeaways

- The `-p` flag transforms Claude Code from an interactive tool into a composable Unix utility -- now framed as the Agent SDK via the CLI -- that reads stdin, writes stdout, and integrates with any pipeline.
- Use `--bare` for scripts so runs ignore whatever hooks, plugins, and MCP servers happen to be on the machine; it is slated to become the default.
- `-p` runs now start in Manual mode, denying anything that needs approval. Choose a baseline with `--permission-mode` (`dontAsk` for strict allowlists, `auto` for classifier review) and add `--permission-prompts none` when nobody can answer.
- `--json-schema`, `--max-budget-usd`, and `--max-turns` make headless runs structured and bounded.
- Four system prompt flags (`--system-prompt`, `--system-prompt-file`, `--append-system-prompt`, `--append-system-prompt-file`) work in every mode; prefer file-based variants for CI, and use the dynamic boundary marker to keep the static part cached.
- The fan-out pattern with scoped tools enables safe batch processing; `/batch` and dynamic workflows are the built-in alternatives.
- Async hooks run tests in the background after every write; `TaskCompleted` hooks block task completion until tests pass (exit 2, reason on stderr); agent-based `Stop` hooks verify quality before Claude finishes.
- Pre-commit hooks (type check, lint, test) create automated backpressure that makes Claude self-correct at the commit boundary -- you stop being the quality bottleneck.
- Devcontainers with network isolation and `--dangerously-skip-permissions` remain the standard pattern for fully unattended execution in CI.
- CI auto-fix arrived: `/autofix-pr` hands a pull request to a cloud session that pushes fixes for failing checks and review comments.
- Review comes in three tiers: `/code-review` in-session, ultrareview in the cloud (`claude ultrareview` for CI), and the managed Code Review service on every pull request.
- Schedule work with `/loop` inside a session, desktop scheduled tasks on your machine, or cloud scheduled tasks that run with your laptop closed.
- Start with interactive usage, identify repetitive patterns, and progressively move them to headless execution.
