# Recreating the Claude Code Hooks Mastery “agentic layer” across Claude Code, Codex, and OpenCode

## Why this repo works as an “agentic layer” blueprint

The core engineering move in **claude-code-hooks-mastery** is to treat “agent workflows” as *first-class, versioned artifacts* that live alongside your application code—commands, agents/subagents, validation scripts, permissions, logs, and status UX—so the system becomes repeatable and inspectable rather than “vibes in a terminal.” citeturn20view0turn27view0

Two design choices dominate the repo’s leverage:

First, it uses **hooks as deterministic control points**. Hooks run *outside* the model’s discretionary loop and can validate/block/augment behavior using exit codes and structured JSON outputs. That makes it possible to turn “I hope the agent did X” into “the workflow cannot complete until X is true.” citeturn4search0turn22search0turn25search0

Second, it uses **multi-agent orchestration (builder/validator “teams”)** to scale compute and trust. In the repo’s team-based workflow, a planning meta-prompt produces an explicit plan, then specialized “builder” and “validator” agents execute and verify work in coordinated tasks. citeturn27view0turn25search0

Those two together (deterministic gates + compute-scaled verification) are the essence you want to port to entity["company","OpenAI","ai company"]’s entity["product","Codex","openai coding agent"] and entity["product","OpenCode","ai coding agent"].

## What Claude Code Hooks Mastery actually ships as reusable primitives

The repo is structured to demonstrate an end-to-end “hooked” development environment: a `.claude/` layer (hooks, commands, agents, output styles, status lines) plus `logs/` and planning specs. citeturn20view0turn25search0

From the README’s enumerations and walkthrough sections, the reusable primitives break down into five categories:

The first category is hookable lifecycle observability and enforcement. The repo’s README explicitly frames hook events as a lifecycle with structured JSON payloads, and it logs hook executions to `logs/` as append-only JSON artifacts while warning that `chat.json` only retains the most recent conversation snapshot. citeturn25search0turn20view0turn27view0

The second category is a “skills” mindset (in your words: prompts that create programmatic control). Here, the “skills” are effectively implemented as: (a) **commands** that act like parameterizable templates, (b) **agents** with explicit tool scopes and descriptions that trigger proactive delegation, and (c) validators and policies that constrain both. citeturn27view0turn25search0turn4search0

The third category is self-validating prompts. The repo’s `/plan_w_team` is explicitly described as “not an ordinary prompt” because it embeds hook definitions/validators in its command frontmatter to validate the artifact it generates. citeturn27view0

The fourth category is orchestration via Claude Code tasks. The README lists the task-tool primitives (`TaskCreate`, `TaskUpdate`, `TaskList`, `TaskGet`) and describes dependency-aware coordination as the mechanism for long-running parallel work without brittle “sleep loops.” citeturn27view0

The fifth category is UX scaffolding (status lines, output styles) that reduces operator overhead while preserving traceability. citeturn20view0turn27view0

## How Claude Code hooks deliver deterministic control

### The official hook contract you must port conceptually

In entity["product","Claude Code","anthropic cli coding agent"], hooks are configured in settings files (user, project, local project), organized by **event name** and (when applicable) a **matcher** that filters which tool calls trigger the hook. citeturn10search3turn22search0

Claude Code’s hook system provides two “control channels”:

Exit codes:
- Exit code **0** = success (with important nuances: for `UserPromptSubmit` and `SessionStart`, stdout is injected into context; for other hooks, stdout is mainly visible in transcript mode). citeturn4search0turn22search0  
- Exit code **2** = blocking error, with hook-type-specific behavior (e.g., blocks tool execution for `PreToolUse`, blocks stopping for `Stop`, blocks prompt processing for `UserPromptSubmit`). citeturn4search0turn22search0

Structured JSON output:
- Hooks can emit JSON fields like `continue`, `stopReason`, `systemMessage`, and—critically for “don’t let the agent finish yet”—decision controls for Stop/SubagentStop and other events. citeturn4search0turn25search0

This matters because the repo’s “self-validating prompts” depend on **Stop** semantics (prevent concluding until validators pass). citeturn27view0turn4search0

### The repo’s `/plan_w_team` demonstrates three hook-powered patterns

The README describes `/plan_w_team` as having three components: **self-validating**, **agent orchestration**, and **templating**. citeturn27view0

Self-validating: the prompt embeds Stop hooks that run validators like `validate_new_file.py` and `validate_file_contains.py` against `specs/*.md`. The README explains the intent: ensure a spec file is created in the right directory and contains required sections; if validation fails, the agent receives feedback and must continue until it passes. citeturn27view0

Agent orchestration: the prompt leverages Claude Code’s task system, listing `TaskCreate`, `TaskUpdate`, `TaskList`, and `TaskGet`, and describing a dependency-driven flow where tasks unblock automatically when prerequisites complete. citeturn27view0

Templating: `/plan_w_team` is explicitly framed as a “template meta prompt”—it generates plans in a predictable schema with placeholders (Plan Name → Team Orchestration → Step-by-Step Tasks) so you can anticipate and automate downstream execution. citeturn27view0

### The “team” pattern is builder/validator with tool-scoped roles

In the repo, the builder and validator are distinct agents: a builder with full tools and a validator that is read-only (explicitly no Write/Edit), used to “increase compute to increase trust.” citeturn27view0turn25search0

Separately, code-quality validators are described as PostToolUse validators that can enforce lint/type checks (example: Ruff/Ty on `.py` files, blocking when errors are found). citeturn27view0turn4search0

This combination is your portable core: **(1) orchestrate work**, **(2) validate independently**, **(3) block completion until verified**.

## Generalizing the prompt strategy into a PacketBuilder-style workflow

You described “PacketBuilder” as the prompt that made your parallelized implementation-plan execution click, with emergent adaptation: the orchestrator refines downstream agent prompts based on completed work. The repo already hints at the key enabling mechanisms:

- A **plan format** with explicit, chunkable steps (templating). citeturn27view0  
- A **task system** with dependency modeling (for ordering and parallelism). citeturn27view0  
- **Hooks/validators** that prevent “done” unless hard conditions are met. citeturn27view0turn4search0  
- **Role-separated agents** (builder vs validator) that allow unbiased verification. citeturn27view0

A PacketBuilder-style extension adds two more orchestration-quality upgrades that are especially relevant when porting to systems that don’t match Claude Code’s task UX:

- A durable external “source of truth” task board (`task.md`, `packet-run.md`) so the orchestration survives model/session limitations. (This is the conceptual equivalent of the repo’s plan spec plus task-system visibility.) citeturn27view0turn24search1  
- A mandatory “what changed” step (diff review) between dependent packets, so the orchestrator can update the next builder prompt based on *reality*, not assumptions. You can implement this in any CLI that can run `git diff`, even if it lacks first-class Stop hooks. citeturn24search1turn21search1

In other words: if Claude Code gives you “Stop hook blocks completion,” you can approximate it elsewhere by: **always diff + always update task board + always run validator agent + refuse to progress on failures**.

## Porting the system to OpenCode with minimal feature loss

OpenCode is structurally close to Claude Code in the ways that matter for this port: it has agents (including subagents), custom commands with prompt templates and argument substitution, CLI invocations for programmatic use, and—importantly—an explicit permissions system. citeturn7search0turn7search2turn24search1turn23search1

### Recreating “slash commands + prompt variables” in OpenCode

OpenCode supports custom commands defined as markdown files with frontmatter (description/agent/model) and supports `$ARGUMENTS` plus positional arguments (`$1`, `$2`, …). citeturn23search1turn23search2

This directly maps to your “drop in the prompt and run it” requirement: create a project command like `.opencode/command/packet_builder.md` whose content is your orchestrator prompt, and pass the plan path as the argument.

OpenCode also supports template injection of shell output and file content (e.g., `!`command`` and `@filename`) which is a powerful equivalent to “context injection” hooks for many workflows—because you can force inclusion of `git diff`, `AGENTS.md`, or your plan file at invocation time. citeturn23search1

### Recreating “multi-agent + tool-scope roles” in OpenCode

OpenCode has **primary agents** (like Build/Plan) and **subagents** (like General/Explore), and you can invoke them via `@` mentions. citeturn7search0turn7search4

The builder/validator split ports cleanly:

- Builder agent: allow edits + bash (or “ask” with broad allow rules depending on your risk tolerance).  
- Validator agent: deny edit/write; allow read/grep/diff/test commands; optionally deny web fetch to prevent prompt-injection via remote content.

OpenCode’s permission system is explicit and granular. It supports `allow/ask/deny` rules with patterns for bash commands, and it even includes safety-guard permissions like `external_directory` and a `doom_loop` guard that triggers on repeated identical tool calls. citeturn24search1turn24search0

That means you can enforce much of what Claude Code hooks are used for (blocking dangerous commands, preventing `.env` reads, stopping repeated loops) via config—without building a separate hook runner. citeturn24search1turn23search4

### Recreating “programmatic CLI execution” in OpenCode

OpenCode documents a non-interactive CLI command `opencode run "..."` and also supports flags like `--model` and `--agent`, which is what you need for “press start, run the workflow, log output.” citeturn7search2turn7search0

A practical pattern is:

1. `opencode run` (or `opencode --prompt ...`) executes the orchestrator command. citeturn7search2turn23search1  
2. The orchestrator writes/updates a persistent `specs/packet-runs/<plan>-run.md` task-board file (your durable `TaskList`). (This parallels the repo’s emphasis on structured plans and logs.) citeturn27view0turn25search0  
3. The orchestrator triggers subagents for builder/validator work using `@` mentions or `subtask`-configured commands. citeturn23search1turn7search0  
4. Permissions ensure the validator cannot accidentally change code, turning “review” into a real constraint. citeturn24search1turn7search0

## Porting the system to Codex with safe approximations for missing features

OpenAI’s Codex ecosystem (Codex cloud agent + Codex CLI + Codex app) is explicitly designed around multi-agent parallelism, work isolation (worktrees), and safety controls (sandboxing, approvals, rules). citeturn8search3turn8search4turn21search1turn8search5

### What Codex provides that aligns with Claude’s orchestration model

Codex is positioned as a cloud-based software engineering agent that can run many tasks in parallel, and OpenAI recommends assigning well-scoped tasks to multiple agents simultaneously. citeturn8search0turn8search3

Codex CLI is a local terminal agent with approval modes, and OpenAI describes updates including progress tracking with a to-do list and tools like web search and MCP. citeturn21search1turn8search5

Codex app emphasizes multi-agent workflows with built-in worktrees so agents can work on isolated copies of a repo without conflicts. citeturn8search3turn8search4

These features match your goal of “parallelize work on an implementation plan,” especially if your Packet/Phase decomposition is good.

### Where Codex differs and how to recreate “hooks” behavior

Claude Code hooks provide **pre/post/stop interception** at tool boundaries with structured payloads and block/continue semantics. citeturn4search0turn22search0

Codex’s public materials emphasize sandboxing and approvals (read-only vs auto-edit vs full-auto in the CLI) and configurable rules for elevated permissions, rather than an explicit per-tool hook API in the same style. citeturn21search1turn21search4turn8search5

So, to recreate “hook-like” behavior in Codex CLI, the most robust portable approach is to promote your “hooks” into **first-class workflow steps** that the orchestrator must execute:

- **Pre-step gates**: before a code-editing packet, run a *validator phase* that checks policy (no `.env` reads, no `rm -rf`, etc.) and populates a “constraints” section in the task board. This mirrors what `PreToolUse`/`PermissionRequest` hooks accomplish in Claude Code. citeturn22search0turn27view0turn21search1  
- **Post-step validators**: after each packet, run tests/lint/diff inspections and update the task list; if failing, the orchestrator explicitly re-queues a “fix” packet for that same area. This mirrors `PostToolUse` validation and the repo’s “builder/validator” compute scaling. citeturn27view0turn4search0turn8search5  
- **Stop-gate equivalent**: “do not finish” becomes “do not advance to next packet / do not mark packet DONE.” This is a social/flow contract enforced by the orchestrator, rather than a native Stop hook. The task board file becomes your objective reality. citeturn27view0turn21search1

### Programmatic control: prefer Codex SDK when you need orchestration beyond the terminal

OpenAI explicitly describes a **Codex SDK** for embedding “the same agent that powers the Codex CLI” into applications and workflows, with structured outputs and context management to resume sessions. citeturn21search0turn21search3

If your end goal is “press start → orchestrator runs everything → logs + artifacts land in repo,” the SDK is the cleanest way to build that without screen-scraping or pseudo-tty automation.

## Programmatic implementations and logging strategy

You’re asking for both (a) “drive the CLI like a user” and (b) “drive the agent programmatically.” The cleanest long-term engineering path is to treat CLIs as the *operator UX*, but build the deterministic orchestration in code using official SDKs/APIs when available.

### Claude Code programmatic entrypoints

Anthropic documents a **Claude Code SDK** approach where `claude -p` runs non-interactively and prints final results, with flags for tool permissions and working directory. citeturn22search1turn22search4

A minimal “run orchestrator prompt over a repo” pattern looks like:

```bash
claude -p "Run /plan_w_team on specs/my-plan.md, then execute the resulting plan with builder/validator agents."
```

You’ll typically want to select allowed tools / permission modes for safety, because Claude Code’s hook system can run arbitrary shell commands and can block/continue at multiple lifecycle points. citeturn22search0turn4search0turn22search1

For direct API usage (when you want Claude *without* Claude Code), Anthropic documents client SDKs; the Python SDK example shows `client.messages.create(...)` with a `model` string and messages array. citeturn22search2turn22search3

The tradeoff is that **Claude Code orchestration features** (tasks, subagents, commands, hooks) are product-level primitives; if you go direct to Messages API you must implement orchestration and tool execution yourself.

### OpenCode programmatic entrypoints

OpenCode’s CLI supports programmatic interaction via `opencode run "..."` and supports flags that matter for your “choose another model for coder” requirement: `--model` and `--agent`. citeturn7search2turn7search0

```bash
opencode run --agent build --model opencode/gpt-5.1-codex "Execute packet plan in specs/packet-plan.md"
```

(Exact model IDs depend on your provider configuration; OpenCode documents that model IDs use `provider/model-id` format.) citeturn7search0turn23search4

Additionally, OpenCode’s permissions system is configurable per agent and per command pattern, which is the closest built-in analogue to “hook-enforced safety.” citeturn24search1turn24search0

### Codex programmatic entrypoints

OpenAI documents Codex CLI installation (`npm i -g @openai/codex`) and describes running it locally with different approval modes, and it also describes the Codex SDK for embedding the agent in apps. citeturn21search1turn21search0turn8search1

For lower-level orchestration (tool calling, traces, etc.), OpenAI points to its Responses API and Agents-related tooling (web search, file search, MCP support) and also documents function calling/structured outputs. citeturn8search8turn8search7

### Logging: implement “event logs + durable task board” everywhere

Claude Code Hooks Mastery logs hook events to JSON files and treats those as auditable artifacts; the README explicitly enumerates hook-specific log files and their intent. citeturn25search0turn27view0

To recreate this across Codex and OpenCode (and also in direct API mode), you want two layers of logs:

- **Human audit trail**: append-only `packet-run.md` task board that records each packet, its status, validator results, and the commit SHAs/diffs relevant to that packet. This mirrors the repo’s “templated plan + coordinated tasks” philosophy and gives you a deterministic “TaskList” substitute when the platform lacks one. citeturn27view0turn21search1  
- **Machine telemetry**: JSONL logs per event class such as `orchestrator.jsonl` (packet start/end, diff summary), `builder.jsonl` (agent prompt hash + outputs), `validator.jsonl` (verdict + evidence), and `tool.jsonl` (shell commands run, exit codes). This is the portable replacement for platform-specific hook logs.

If you want near parity with Claude Code’s hook granularity, treat **every packet boundary** as your “Stop hook boundary,” and treat **every shell command** as your “PreToolUse/PostToolUse boundary.”

## The practical “bring it all together” architecture

A portable version of what you’re building looks like this:

A single “orchestrator command” (your PacketBuilder family) that:
- reads a plan/spec file,
- decomposes it into packets with dependencies,
- dispatches builders and validators,
- and writes durable state (`packet-run.md`, `task.md`) so the system stays coherent across sessions and across tools.

In Claude Code, you can amplify determinism with real hooks (Stop validators, tool boundary gates) and with the task system and subagents described in the repo. citeturn27view0turn22search0

In OpenCode, you map:
- Claude commands → OpenCode commands with `$ARGUMENTS` and templating, citeturn23search1turn23search4  
- Claude tool gating via hooks → OpenCode permission rules + agent-specific permissions, citeturn24search1turn24search0  
- Claude subagents → OpenCode subagents and `subtask`-invoked commands. citeturn7search0turn23search1

In Codex, you map:
- Claude tasks/subagents → multi-agent parallelism and worktrees (Codex app) or multiple concurrent agent threads (SDK), citeturn8search3turn21search0  
- Claude hooks → orchestrator-enforced “gates” + approval/rules configurations + validator passes, citeturn21search1turn21search4turn4search0  
- durable task state → `.md` artifacts in repo + structured outputs from SDK. citeturn21search0turn8search7

If you implement the orchestrator as a small library (and treat each CLI as a front-end), you can preserve the exact behavior you liked—especially “the orchestrator adapts prompts for subagents based on completed work”—because that behavior is fundamentally **an orchestrator property**, not a Claude-only property. citeturn27view0turn8search0turn7search0