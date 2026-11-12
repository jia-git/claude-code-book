# Chapter 5: MCP -- Connecting Claude Code to Everything

## What You'll Learn

Model Context Protocol is how Claude Code reaches beyond the filesystem. It is the structured interface between an AI agent and everything else: databases, APIs, cloud services, market data feeds, internal tools, third-party platforms. Without MCP, Claude Code is capable but isolated -- a very smart process that can read files and run bash commands. With MCP, it becomes a node in your infrastructure.

But MCP is not plug-and-play. The first edition of this book opened with a list of complaints: every server cost context tokens at session start, connections failed silently, tools vanished without warning, and background subagents could not use MCP at all. Most of that list has been fixed -- tool schemas now load on demand, remote servers reconnect on their own, and background subagents keep their MCP tools. What remains is subtler. A chatty tool can still flood your context with results. A project's `.mcp.json` can still connect a server you never vetted. And the structured, auditable interface that makes MCP valuable only pays off if you configure it with discipline.

What follows is the mechanics of MCP in Claude Code: how tool loading works, what it costs, how to configure it for teams, where it breaks, how servers can now push events into a running session, and how practitioners have built production-scale integrations with dozens of tools. You will learn how to reference MCP resources with the `@server:resource` syntax, how managed settings control which servers your organization permits, and how hook patterns like `mcp__server__tool` enable fine-grained logging, validation, and transformation. You will see a sovereign wealth fund running MCP integrations across thousands of portfolio managers, a production trading platform with 41 MCP tools and sub-10ms GPU-accelerated inference, and a marketing team that built an MCP server for campaign analytics in days. When MCP is the right choice over raw bash, and what a real-world connector architecture looks like at scale.

---

## How MCP Tool Loading Works

When a Claude Code session starts, every configured MCP server connects and announces its tools. In the early days, every tool's full definition -- name, description, parameter schema -- loaded into the context window immediately, and you paid for all of them on every request whether you used them or not.

Tool search changed the economics. It is now on by default: at startup Claude sees each server's instructions and the *names* of its tools, and the full schema for a tool is fetched only when Claude decides it needs that tool. Connect ten servers with two hundred tools between them and the idle cost is a list of names. This is the reason MCP can now scale to dozens of servers without crowding out your work.

The search step is not free -- it adds a round trip the first time a tool is used -- but for any realistic catalog it is overwhelmingly the right default. The one place it historically switched itself off was behind a custom API gateway that did not forward the needed request blocks; current versions try it there too and fall back automatically if the gateway rejects it. (`ENABLE_TOOL_SEARCH=true` forces it on for a proxy you know forwards them.)

### Measuring Your MCP Context Cost

`/context all` breaks your context down to the token cost of each loaded MCP tool. `/usage`, on a subscription plan, attributes recent consumption to individual MCP servers -- counting only the requests that actually consumed one of that server's results. Between the two, you can see both what a server costs to have connected and what it costs to use.

With definitions deferred, the expensive part of MCP is now the *results*. A tool that returns a 40,000-token JSON blob costs 40,000 tokens every time it is called. Claude Code warns when a tool result exceeds 10,000 tokens and caps results at 25,000 by default (`MAX_MCP_OUTPUT_TOKENS` raises the cap; a server can declare a larger limit for an individual tool, up to 500,000). Very large text results are saved to a file and Claude reads only what it needs. If you are building MCP servers for your team, the lesson inverts from the first edition: keep schemas clear, and spend your discipline on returning compact, filtered results.

## MCP Resources

MCP servers can expose not just tools but also resources -- structured data that you reference using `@` mentions, the same way you reference files. The syntax is `@server:protocol://resource/path`:

```
> Can you analyze @tracker:issue://123 and suggest a fix?

> Please review the API documentation at @docs:file://api/authentication

> Compare @db:schema://users with @docs:file://database/user-model
```

Type `@` in your prompt to see available resources from all connected MCP servers. Resources appear alongside files in the autocomplete menu. When you reference a resource, it is fetched automatically and included as an attachment in the conversation.

This matters because it puts external data on the same footing as local files. Instead of asking Claude to "use the database MCP tool to fetch the users table schema," you reference it directly as `@db:schema://users` and it appears in context. Multiple resources can be referenced in a single prompt. The model sees them as structured inputs, not as tool call outputs, which often produces better responses because the data is present before reasoning begins rather than being fetched mid-conversation.

## Centralized Configuration

MCP servers are configured in `.mcp.json` at the project root. This file should be committed to version control. One person figures out the MCP configuration; the whole team benefits.

```json
{
  "mcpServers": {
    "analytics": {
      "command": "node",
      "args": ["./mcp-servers/analytics/index.js"],
      "env": {
        "DB_CONNECTION": "${ANALYTICS_DB_URL}"
      }
    }
  }
}
```

Environment variable interpolation -- `${VAR}`, or `${VAR:-default}` -- lets the configuration reference credentials without embedding them. The `.mcp.json` file contains the structure; actual secrets live in each developer's environment. One defensive detail: in a remote server's URL and headers, variables holding Claude Code's own credentials read as empty, so a project cannot name a server that quietly receives your API key.

### Transports and Scopes

Three transports matter. **HTTP** is the recommended choice for remote servers (`claude mcp add --transport http notes https://...`). **stdio** runs a local process (`claude mcp add --transport stdio name -- command args`). The older server-sent-events transport is deprecated; current versions try HTTP first and fall back. WebSocket servers can be configured in JSON.

Servers live at three scopes. **Local** (the default) is stored in your user configuration but applies only to the current project. **Project** is the committed `.mcp.json`. **User** applies to every project. When two scopes define the same name, local beats project beats user, which beats plugin-provided servers and connectors from your Claude account. Organization-provided servers rank above all of them.

Authentication for remote servers is OAuth. Complete it from the `/mcp` panel inside a session, or from the shell with `claude mcp login <name>` (and `claude mcp logout` to clear stored credentials) -- the shell form works over SSH, where it prints the authorization URL instead of opening a browser. For servers that need short-lived tokens, a `headersHelper` command generates headers at connection time and is re-run when the server rejects a request.

### Connectors from Your Claude Account

If you use connectors in the Claude apps, they are available in Claude Code too, at the lowest precedence. That has a practical upside -- one sign-in on the web and the connector works in your terminal, in cloud sessions, and in published artifacts -- and a governance implication: your organization's connector policy now reaches your coding sessions. Connectors an administrator marks as "ask" always prompt, whatever permission mode you are in.

### Managed MCP: Allowlists, Denylists, and Auto-Approval

For enterprises, managed settings can restrict which MCP servers are permitted. An allowlist (`allowedMcpServers` in managed settings) specifies the only servers that may be configured. A denylist (`deniedMcpServers`) blocks specific servers. This prevents developers from connecting arbitrary external services to their Claude Code sessions -- a meaningful control when the organization has data classification policies.

The allowlist/denylist operates at the configuration level. If a server is not on the allowlist, Claude Code will not connect to it, regardless of what appears in the project's `.mcp.json`. This is the same override pattern as the permission scope hierarchy described in Chapter 2: managed settings win.

Three user-level settings control the approval flow for project-defined MCP servers:

**`enableAllProjectMcpServers`** automatically approves all MCP servers in project `.mcp.json` files. Set this to `true` when you trust the repositories you work in and want zero friction. Set it to `false` (the default) when you want to approve each server individually.

**`enabledMcpjsonServers`** is a list of specific server names to auto-approve: `["memory", "tracker"]`. Servers on this list connect without prompting. Servers not on this list still prompt for approval.

**`disabledMcpjsonServers`** is a list of specific servers to reject: `["filesystem"]`. Servers on this list never connect, even if the project `.mcp.json` defines them.

These settings matter for organizational deployment. A team can commit `.mcp.json` to version control with all the servers the project needs, then configure managed settings to allowlist those servers and denylist everything else. New team members clone the repo, get the MCP configuration automatically, and the managed settings ensure no rogue servers connect. Two newer controls round this out: managed configuration can *provide* servers to every user rather than merely permit them, and a managed MCP file can take exclusive control so that no other configuration source is read at all.

One trust caveat. In an untrusted folder, approvals committed to the project's settings are ignored and the server waits at "pending approval" until you trust the workspace. But in a non-interactive `-p` run there is no approval prompt, so a repository's `.mcp.json` servers connect without asking. When scripting over repositories you did not write, use `--strict-mcp-config` with your own `--mcp-config`, or `--bare`. And if you change your mind about a project's servers, `claude mcp reset-project-choices` clears your earlier approvals.

## Where MCP Breaks

MCP connections are not as reliable as local tool calls. Understanding the failure modes prevents confusion when things go silent.

### Disconnection

An MCP server can disconnect mid-session. For remote servers this is now mostly self-healing: Claude Code retries a dropped HTTP connection up to five times with exponential backoff, and retries a failed first connection a few times before giving up. Local stdio servers that crash are the remaining hazard -- they do not restart on their own, and their tools are simply gone until you reconnect them from `/mcp`.

The symptom is still subtle: Claude stops using a capability it was using moments ago, or it starts doing something manually that it was doing through a tool. If Claude suddenly begins writing raw HTTP requests instead of using your API connector, check your MCP connections. `/mcp` shows current connection status; `/mcp reconnect all` retries every failed server in one step. Make it a reflex when behavior changes unexpectedly.

### Server Startup Failures

MCP servers can fail to start when a session begins. A missing dependency, a bad environment variable, a port conflict -- any of these can prevent a server from initializing. Claude Code will start the session without those tools, and you may not notice their absence until you need them. Slow servers are less of a problem than they were: servers can finish connecting in the background, and Claude can wait for a still-connecting server when a request needs its tools.

The diagnostic path: run `/mcp` at session start, or `claude mcp list` from the shell. Confirm all expected servers are connected and their tool counts match expectations. In scripted runs, the session's initial metadata event lists each server's status and any configuration entries that were skipped as invalid -- a CI job can fail fast on a non-empty error list instead of discovering the missing tool twenty minutes in. This takes five seconds and prevents thirty minutes of confusion.

### Long-Running Calls

A tool call that is still running after two minutes in the main conversation now moves to the background on its own, so a slow query no longer freezes the session. The result arrives when it is ready. Servers that support it can also run tasks on the server side that survive the session; resume the session to pick the result back up.

### Tools with Identical Names

If two MCP servers expose tools with the same name, behavior is undefined. One will shadow the other. Which one wins may depend on server initialization order, which you do not control. Name your tools with server-specific prefixes to avoid collisions.

## MCP in Subagents

The first edition called this "a hard architectural constraint, not a configuration option": background subagents could not use MCP tools. That constraint is gone. Background subagents -- now the default (Chapter 4) -- keep their MCP tools, and when one needs a permission to call a tool, the prompt surfaces in your main session instead of being auto-denied.

What remains is scoping. A custom subagent's `mcpServers` field decides which servers it can reach, so a database-reader agent gets the database server and a code reviewer gets none. In its `tools` list, `mcp__db__*` grants one server's tools; in `disallowedTools`, `mcp__*` withholds them all. The old workaround -- fetch the data in the main session and hand it to the worker -- is still useful, but now for a different reason: when twenty parallel workers would each hit the same rate-limited API, one fetch is kinder to the server than twenty.

## MCP in Plugins

Plugins can bundle MCP server configurations that auto-start when the plugin is enabled. From the user's perspective, the plugin's MCP tools appear as standard tools -- indistinguishable from built-in capabilities.

This is the distribution mechanism for MCP. Rather than asking every team member to configure MCP servers manually, a plugin packages the server binary, its configuration, and the auto-start logic into something that activates with a single enablement step.

Plugin MCP servers start with every session where the plugin is enabled. With tool search their idle context cost is small, but they are still processes running on your machine with your permissions. The `/plugin` manager's stats and `/usage` attribution show which plugins actually get used; disable the rest. Note the naming: a plugin's tools are namespaced as `mcp__plugin_<plugin>_<server>__<tool>`, and permission rules and hook matchers must use that full form.

## MCP Tool Hooks

Hooks interact with MCP tools through a naming convention: `mcp__servername__toolname`. This pattern allows PreToolUse and PostToolUse hooks to match specific MCP tools with the same regex-based matching used for any other tool.

A hook matching `mcp__analytics__.*` intercepts every tool from your analytics MCP server. A hook matching `mcp__.*__write.*` intercepts any write operation across all MCP servers. A hook matching `mcp__memory__.*` catches all operations on a memory server specifically. The pattern matching is flexible enough to express server-level, tool-level, or cross-server policies.

The double-underscore convention (`mcp__server__tool`) is not arbitrary -- it is the exact format Claude Code uses internally to namespace MCP tools. When Claude calls a tool from your analytics server named `get_price_history`, the internal tool name is `mcp__analytics__get_price_history`. Your hook matcher targets this full namespaced name, which means you can write hooks that distinguish between tools of the same name on different servers. Permission rules follow the same names, with one precision worth knowing: a rule names one server exactly, so `mcp__analytics__*` will not accidentally cover a server called `analytics_v2`.

The relationship now runs both ways. A hook can itself *be* an MCP call: the `mcp_tool` handler type invokes a tool on a configured server when an event fires, which is the natural shape for sending every tool invocation to an audit service that already speaks MCP. And when a server asks the user for input mid-call -- an elicitation -- the Elicitation hook can answer it programmatically, which is what lets such servers work in unattended runs.

### Logging

The most immediately valuable MCP hook is a logging hook. Every MCP tool invocation -- server, tool, parameters, timestamp -- written to a file or sent to a monitoring service. This creates an audit trail for external service access that is otherwise invisible.

For regulated industries, this is not optional. If your MCP servers connect to financial data sources or customer databases, an auditable log of every query Claude made is a compliance requirement. A PreToolUse hook on `mcp__.*` with a logging command handles this in a few lines of configuration.

### Transformation

PostToolUse hooks can transform MCP tool output before Claude processes it, by returning a replacement result. This is useful when an MCP tool returns more data than Claude needs. A hook that filters a large API response to the relevant fields reduces context consumption and improves Claude's ability to focus on what matters.

The opposite transformation is also valuable: enriching sparse MCP output with additional context. A hook that appends metadata or formatting to a tool's output can improve Claude's interpretation without modifying the MCP server itself.

### Validation

PreToolUse hooks on MCP tools can enforce input validation before the request reaches the external service. A hook that checks whether a database query MCP tool is receiving a SELECT statement (allowed) versus a DROP TABLE statement (denied) prevents catastrophic mistakes without modifying the MCP server's code.

This is defense in depth. The MCP server should have its own input validation. The hook is a second layer that catches what the server might not.

## Channels: When the Server Talks First

Every MCP interaction described so far starts with Claude. Claude decides to call a tool; the server answers. Channels reverse the direction. A server that declares the channel capability can *push* a message into a running session -- a CI failure, a chat message, a monitoring alert, a webhook -- and Claude reacts to it as a new turn, even while you are away.

Start a session with `--channels` naming the channel plugins to listen to. A channel can also expose a reply tool, so Claude can answer back through the same pipe: a chat message arrives asking why the nightly build failed, Claude investigates and replies in the thread. Channel plugins gate who can send, and permission prompts can be relayed out through the channel so a human can approve from wherever the message came from.

This turns a running session into something closer to an on-call engineer than a tool: it sits idle, costs nothing, and wakes up when the event it was waiting for arrives. It is still a research preview, and only allowlisted channels load by default. The polling alternative -- `/loop` checking a status every few minutes -- works everywhere, but every poll re-sends your full context. A channel only spends tokens when something actually happens.

## Domain-Specific Connector Patterns

The most instructive MCP deployments are in domains with rich external data requirements. Financial data analysis provides a useful case study because it involves multiple data sources, real-time feeds, structured and unstructured data, and strict security requirements.

### The Connector Ecosystem

A production analytics platform in financial services might integrate over a dozen data connector categories via MCP, each a separate server wrapping a specific data provider:

- **Fundamental equity data**: company financials, rankings, screening tools
- **Alternative data**: non-traditional datasets aggregated from diverse sources
- **Fund research**: ratings, analysis, fund comparison data
- **Private markets data**: venture, private equity, M&A intelligence
- **Earnings transcripts**: real-time call transcripts and investor event coverage
- **Expert intelligence**: interview transcripts, company research, industry analysis
- **Document governance**: secure data room access with permission controls
- **Live market data**: real-time pricing for fixed income, FX, equities, macro indicators
- **Credit ratings**: entity ratings and research across millions of entities
- **News feeds**: global multi-asset news in real time
- **Regulatory filings**: SEC filings, international regulatory submissions

Each connector is an MCP server that exposes tools for querying its specific data source. The pattern is consistent across all of them:

1. The MCP server wraps an API client for the data source.
2. Tools expose domain-specific queries (get price history, search filings, fetch fundamentals).
3. Authentication is handled at the server level, not passed through Claude.
4. Response schemas are structured for Claude to interpret without additional parsing.

This architecture keeps credentials out of Claude's context (they live in the MCP server's environment), provides structured access patterns instead of raw API calls, and makes every data access auditable through hooks.

### Pre-Built Agent Skills for Domain Workflows

Beyond raw connectors, some domains have developed pre-built agent skills that combine MCP data access with standardized analysis workflows. In financial services, for example, skills exist for comparable company analysis (valuation multiples and operating metrics producing benchmark datasets), discounted cash flow modeling (projections, discount rate calculations, sensitivity tables), due diligence data processing (extracting structured data from document rooms), earnings analysis (pulling metrics, guidance changes, and management commentary from transcripts), and coverage initiation reports (industry analysis with valuation frameworks).

These skills are not MCP servers themselves -- they are packaged workflows that consume data from MCP connectors and produce structured output. The pattern generalizes: any domain with recurring analytical tasks can build skills that standardize the workflow while pulling data from MCP connectors. The MCP server handles data access; the skill handles the analytical process.

### Sovereign Wealth Fund: MCP at Institutional Scale

The most striking MCP deployment in the public record involves a sovereign wealth fund managing over a trillion dollars in assets. The fund built custom MCP integrations connecting Claude to internal portfolio company data systems. Approximately 9,000 portfolio managers query these integrations daily, receiving AI-generated insights on portfolio company performance, metrics, and trends.

The scale is instructive. This is not a developer tool or a prototype. It is infrastructure serving thousands of users in a regulated financial context. The MCP architecture makes it work because credentials and data access policies are server-side, every query is auditable, and the integration points are structured -- portfolio managers interact with Claude, Claude interacts with MCP tools, MCP tools query the fund's internal systems. No credentials traverse the conversation context. No unstructured API calls risk malformed queries against production databases.

### Production Neural Trading Platform: 41 MCP Tools

The ceiling of what is possible with MCP integration is visible in a production trading platform that combines neural forecasting with real-time market analysis. The system integrates 41 MCP tools across multiple domains:

- **Real-time analytics**: market data feeds, price history, volume analysis
- **Neural forecasting**: interface to GPU-accelerated prediction models
- **Backtesting**: Monte Carlo simulation and historical strategy evaluation
- **Risk management**: position sizing, drawdown analysis, correlation monitoring
- **News and sentiment**: real-time news analysis and sentiment scoring

The neural forecasting engine runs state-of-the-art time series models with sub-10ms prediction latency, GPU-accelerated with a 6,250x speedup via custom GPU kernel optimization. Claude Code generated the forecasting engine code and the MCP integration layer. The MCP tools provide the structured interface between Claude's reasoning and the GPU-accelerated inference -- Claude decides what to predict and how to act on predictions, while the heavy numerical computation runs server-side where GPU acceleration matters.

At this scale, every configuration discipline described earlier in this chapter becomes mandatory. Server-per-domain grouping isolates failures. Deferred loading prevents context exhaustion. Naming conventions keep 41 tools navigable. Health monitoring catches disconnections before they cascade.

### The Prompt Router Pattern

A recurring architectural pattern in production MCP deployments is the prompt router: a classification layer that sits between user input and MCP tool dispatch. When a user sends a message, the router classifies intent before dispatching to the appropriate processing pipeline.

The pattern looks like this in practice: an inbound request arrives. A router determines whether the user wants to create something, analyze something, query data, or perform an action. Based on the classification, the system invokes specific MCP tools in a structured sequence. A "create strategy" intent triggers a pipeline that creates a portfolio object, loops through strategy creation, breaks down conditions, builds structured objects, and sends the result. An "analyze" intent triggers a different pipeline that fetches data via MCP, runs analysis, and returns findings.

This is not specific to any domain. Any system that exposes many MCP tools to Claude benefits from a router that narrows the tool set before Claude starts reasoning. Instead of Claude choosing from 41 tools on every turn, the router reduces the candidate set to the 5-8 tools relevant to the classified intent. This reduces both token cost (fewer tool descriptions in context) and error rate (fewer irrelevant tools to mistakenly invoke).

### The Three-Layer Architecture

A production deployment pattern that combines MCP with user-facing applications uses three layers:

**Layer 1: AI-powered creation.** Users describe what they want in natural language. Claude processes the intent, invokes MCP tools to access data and computation, and produces structured output. This layer is token-expensive but handles the hard part: translating human intent into structured data.

**Layer 2: Visual display.** The structured output from Layer 1 renders in a standard UI. Users see results, configurations, and data in a readable format. This layer costs nothing in AI tokens -- it is pure frontend rendering.

**Layer 3: No-code refinement.** Users manually adjust the AI-generated output through a visual interface -- toggling parameters, adjusting thresholds, fine-tuning configurations. This layer also costs zero AI tokens. Changes feed back into the same data model that Layer 1 produced.

The economic rationale is clear: AI handles ideation (expensive but high-value), the UI handles display (free), and no-code editing handles refinement (also free). MCP sits in Layer 1, providing the data access and computation that Claude needs to generate structured output. The three-layer split means you pay for AI tokens only during creation, not during the entire user workflow.

### Why Not Just Curl?

A developer could accomplish the same thing by giving Claude bash access and letting it run curl commands against data APIs. This works and is worse in every measurable way.

With curl through bash, credentials appear in command strings (visible in context, logs, and potentially in API calls to the model provider). Access patterns are unstructured (parsing curl output is fragile). Audit logging requires parsing bash command strings. Rate limiting must be handled by the model's judgment rather than server-side logic.

With MCP, credentials are server-side. Access is structured. Logging is automatic with hooks. Rate limiting is enforced by the server. The initial setup cost is higher. The operational security is categorically better.

## Architecture Patterns

### The Go + JavaScript Pattern

A recurring architecture in production MCP deployments uses Go for the heavy computation and JavaScript for the MCP server interface. The rationale is practical and practitioner-motivated: Go handles concurrent data processing, complex algorithms, and performance-sensitive operations. JavaScript is what Claude Code writes most fluently when generating MCP server code.

One practitioner described the reasoning bluntly: Go for heavy lifting because of existing infrastructure expertise, JavaScript for the MCP server because it is the easiest language for Claude to write. The architecture works like this: the Go backend exposes a local API (HTTP or a binary RPC protocol). The JavaScript MCP server translates between MCP protocol and the Go API. Claude interacts with the MCP server; the MCP server delegates to Go.

The token-saving angle is significant. A codebase analysis tool built with this architecture reported 80-90% token savings compared to having Claude analyze code directly. The Go backend does the heavy parsing and analysis, returning structured results through the MCP server. Claude receives pre-digested information rather than raw source files, consuming a fraction of the context.

This is not the only valid architecture. Python MCP servers work. Rust MCP servers work. But the Go + JavaScript split exploits a specific strength: when you ask Claude Code to build or modify your MCP server, it produces cleaner JavaScript than it does Go. If your MCP server is a thin translation layer, quality of AI-generated code matters more for the interface than for the computation.

### MCP Beyond Engineering: Marketing Campaign Analytics

MCP is not limited to developer tools and data APIs. A growth marketing team built an MCP server integrated with a major social media advertising platform's API to query campaign performance, spending data, and ad effectiveness directly within Claude. Instead of switching between the ad platform's dashboard and Claude for analysis, they query everything in one place.

The MCP server wraps the advertising API, exposes tools for fetching campaign metrics, and returns structured data that Claude can reason about. The team uses it to answer questions like "which campaigns had the highest cost per acquisition this week" or "compare performance of headline variant A versus variant B across all active campaigns." Every query is auditable through hooks, and the API credentials never enter Claude's context.

This is the MCP pattern generalized beyond engineering: any workflow that involves querying an external service, analyzing the response, and making decisions based on the analysis is a candidate for an MCP integration. The effort to build the server is a one-time cost. The ongoing benefit is structured, secure, auditable access to external data from within Claude's reasoning loop.

### Scaling to 41+ Tools

Production deployments exist with over 40 MCP tools across multiple servers. At this scale, configuration discipline becomes non-negotiable.

The patterns that make large integrations work:

- **Server-per-domain grouping**: One MCP server per data domain or service. Analytics tools on one server, portfolio tools on another, backtesting tools on a third. This makes failures isolated -- a crashed analytics server does not take down portfolio management.
- **Deferred loading by default**: With 40+ tools, eager loading would consume a prohibitive fraction of context. Tool search, now the default, handles this -- just make sure nothing in your environment turns it off.
- **Naming conventions**: Every tool prefixed with its domain: `analytics_price_history`, `portfolio_current_holdings`, `backtest_run_strategy`. Claude can reason about tool selection better when names are self-documenting.
- **Minimal schemas and compact results**: Each tool's parameter schema describes exactly what is needed and nothing more, and each result returns only what Claude needs. Rich usage guidance goes in a skill that teaches Claude how to use the tools well, not in schema descriptions that load every time a tool is fetched.
- **Health monitoring**: A startup hook or script that verifies all MCP servers are responding before beginning work. Thirty seconds of verification prevents an hour of debugging why tools are missing.

## MCP Over CLI for Data Security

The security argument for MCP over raw CLI access deserves emphasis because it contradicts the instinct to "just use bash."

Bash is universal. Any API with a CLI client or a curl endpoint is accessible through bash. MCP requires building and maintaining a server. The effort difference is real.

But the security properties are not equivalent. When Claude runs a bash command, the full command -- including any embedded credentials, API keys, or tokens -- exists in the conversation context. That context is sent to the model provider's API. Your credentials transit through a third-party service.

When Claude uses an MCP tool, the tool name and parameters exist in context. The credentials exist in the MCP server's environment, which never leaves your machine. The MCP server makes the authenticated request locally. Only the response data enters Claude's context.

For organizations with data classification policies, this distinction matters. It is the difference between "our API keys appear in third-party API calls" and "our API keys stay on our infrastructure." If your security team has opinions about where credentials can exist -- and they should -- MCP is the architecture that satisfies those opinions.

This is not about trust in any particular model provider. It is about architectural hygiene. Credentials belong in server environments, not in conversation contexts, regardless of who is on the other end.

A data infrastructure team put it concisely: use MCP servers rather than the CLI for sensitive data to maintain better security control, especially for data with logging and privacy concerns. The CLI approach works for internal, non-sensitive operations. For anything with compliance requirements -- database queries against production, API calls to financial data providers, access to customer data systems -- MCP is the architecture that satisfies auditors.

### Enterprise Data Privacy Patterns

In regulated environments, MCP is one component of a broader data handling strategy:

**Local filesystem for sensitive data.** Financial data, customer records, and proprietary datasets stay on the local machine. Claude Code accesses them through standard file operations. The data never transits through external APIs beyond what the model provider's API requires for the conversation context.

**MCP for restricted system integrations.** When Claude needs to query a database, access an internal API, or interact with a governed data platform, MCP servers mediate the connection. Credentials stay server-side. Access patterns are structured and auditable. The MCP server can enforce rate limits, query restrictions, and data filtering before results enter Claude's context.

**Context window for non-sensitive research.** Large context windows (100,000+ tokens) are useful for processing non-sensitive documents -- research papers, public filings, open-source documentation. These can be loaded directly without MCP overhead.

The pattern is: classify your data first, then choose the access architecture based on sensitivity. Not everything needs MCP. Not everything should bypass it. The classification determines the architecture, not the other way around.

---

## Key Takeaways

- Tool search is on by default: idle MCP servers cost little more than a list of tool names. The expensive part of MCP is now tool *results* -- design servers that return compact, filtered output.
- `/context all` and `/usage` show what each server costs to keep connected and to use.
- MCP resources (`@server:resource` syntax) put external data on the same footing as local files, enabling direct references in prompts.
- Use HTTP for remote servers, stdio for local ones; scopes resolve local, then project, then user, then plugins and account connectors, with organization-provided servers above all. Authenticate from the shell with `claude mcp login`.
- Remote servers reconnect on their own; crashed local servers do not. `/mcp` and `/mcp reconnect all` are your first moves when Claude's behavior changes unexpectedly.
- Background subagents now keep their MCP tools; scope which servers each agent can reach with `mcpServers` and exact-server permission rules.
- Channels let a server push events into a running session, so Claude can react to CI failures, chat messages, and alerts without polling.
- Hook patterns like `mcp__server__tool` enable logging, validation, and transformation of MCP tool invocations; hooks can also call MCP tools directly and answer server elicitations.
- MCP servers keep credentials out of Claude's context, making them categorically more secure than equivalent bash commands for sensitive data access.
- Managed settings control which servers are allowed, denied, or provided to everyone; remember that `-p` runs connect a repository's `.mcp.json` without asking.
- Commit `.mcp.json` to version control so one person's configuration effort benefits the entire team.
- The three-layer architecture (AI creation, visual display, no-code refinement) minimizes token costs by restricting AI to the ideation phase, with MCP providing structured data access in that layer.
- At scale (40+ tools), server-per-domain grouping, deferred loading, and strict naming conventions prevent context bloat and tool collisions.
- For regulated environments, classify data first then choose the access architecture: local filesystem for sensitive data, MCP for restricted system integrations, direct context for non-sensitive research.
