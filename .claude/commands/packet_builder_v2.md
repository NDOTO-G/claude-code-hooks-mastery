---
description: Execute implementation plan packets with delegated validation and diff-aware orchestration between dependent packets
argument-hint: [path-to-plan] [plan|run]
model: opus
---

# Packet Builder v2

You are a **pure orchestrator**. You coordinate the execution of implementation plans by dispatching **Sonnet builder sub-agents** to implement each packet and **separate validator sub-agents** to independently verify the work. You NEVER write code or validate code directly. Your sole job is to read, reason, compose prompts, and make decisions.

Your key advantage: between every packet, you review what actually happened — the sub-agent's report, the actual diff of changes, and the state of the plan — then compose a precisely tailored prompt for the next sub-agent. This is where quality comes from.

## Variables

Parse `$ARGUMENTS` into these variables:
- **PLAN_PATH**: The first argument — path to the implementation plan file.
- **MODE**: The second argument (optional) — either `plan` or `run`. Default: `run`.
  - `plan` — Parse the plan, analyze the codebase, produce the execution strategy and task board, then STOP. No code is written.
  - `run` — Parse the plan, analyze the codebase, produce the execution strategy and task board, then execute all packets with sub-agents.

## Workflow

Follow these steps exactly, in order.

### Step 1: Load and Parse the Plan

- If no `PLAN_PATH` is provided, STOP immediately and ask the user to provide the path to their implementation plan file.
- Read the plan file at `PLAN_PATH`.
- Parse the plan to identify all **packets** — discrete units of work. Packets are identified by any of these patterns:
  - `### Packet N:` headers
  - `### Phase N:` headers
  - `### Task N:` or `### N.` numbered section headers
  - Any similar structured sections that represent individual coding work items
- For each packet found, extract these fields (use "none" or empty if not present in the plan):
  - **ID**: Assign a short ID (P1, P2, P3, ...) for tracking.
  - **Name**: The packet title/header.
  - **Description**: What needs to be done (the body text under the header).
  - **Files**: Files to create or modify (look for file paths, bullet lists of files).
  - **Criteria**: Acceptance criteria, checklist items, or definition of done (look for `- [ ]` checkboxes, "Criteria:", "Acceptance Criteria:", "Definition of Done:", or numbered requirements). **If criteria are missing or vague, synthesize minimal testable criteria from the packet description** (e.g., "file X exists", "function Y is exported", "tests pass").
  - **Dependencies**: Which packets must complete first (look for "Depends On:", "Dependencies:", "Blocked By:", or "after Packet N"). These will be refined in Step 2 after codebase analysis.
  - **Constraints**: Any scope limits, forbidden changes, or special instructions.
  - **Verification**: Any specific commands to validate this packet (tests, lint, build, compile checks).
  - **Context**: Any code examples, references, patterns, or additional notes.

### Step 2: Analyze Codebase and Determine Execution Strategy

**This step is what turns a static plan into an informed execution.** Before creating the task board or dispatching any agents, you must understand the current codebase and decide which packets can safely run in parallel and which must be sequential.

**Context window discipline:** The codebase may be large. Do NOT read all relevant files yourself — that fills your context with raw source code and leaves less room for reasoning. Instead, dispatch **scout sub-agents** to gather the information you need, then use their condensed reports to make decisions.

#### 2a. Dispatch Codebase Scouts

Dispatch one or more Sonnet sub-agents to scan the codebase. Organize scouts by the information you need:

**Scout strategy — choose based on plan size:**

- **Small plan (1-3 packets, few files):** A single scout can cover everything.
- **Medium plan (4-8 packets):** Dispatch 2-3 scouts in parallel — one for docs/reference files, one for existing source files that packets will modify, one for project structure and conventions.
- **Large plan (9+ packets or many files):** Dispatch scouts by concern area — e.g., one per major directory, one for types/interfaces, one for config/wiring, one for test infrastructure.

Launch scouts using the Task tool:
  - `subagent_type`: `"general-purpose"`
  - `model`: `"sonnet"`
  - `run_in_background`: `true` (launch all scouts in parallel)

**Scout Prompt Template:**

```
You are a codebase scout. Your job is to read files and return a structured summary. Do NOT modify anything.

## Files to Analyze
{list of file paths to read}

## What to Report
For each file, provide:
1. **Path**: the file path
2. **Purpose**: what this file does (1 sentence)
3. **Exports/API surface**: what it exports — function names, class names, type names, with their signatures
4. **Patterns**: coding conventions observed — naming, error handling, file structure, import style
5. **Dependencies**: what it imports from other project files (not external packages)
6. **Relevant details**: anything a developer building new code alongside this would need to know

## Format
Return a structured report. Be thorough but concise — summarize, don't paste entire files. Your report will be used by an orchestrator to plan execution order, so focus on information that reveals dependencies and patterns.
```

Wait for all scouts to return before proceeding.

#### 2b. Synthesize Scout Reports and Map File Overlaps

Using the scout reports (NOT by reading the files yourself), build your understanding:

- **Project conventions**: What patterns are established? Naming conventions, file structure, error handling, import style.
- **File overlap matrix**: For each packet, list every file it will touch (create or modify). Cross-reference:
  - **File conflicts**: Do any two packets modify the SAME file? If yes, they CANNOT run in parallel — one will overwrite the other's changes or cause merge conflicts.
  - **Directory conflicts**: Do packets create multiple new files in the same directory with related concerns? They may need ordering so the first establishes the pattern.
- **API surface map**: What types, functions, and classes exist? What does each packet's target file currently export?

#### 2c. Identify Implicit Dependencies

Using the synthesized understanding from 2b, look for dependencies beyond what the plan declares:
- **Pattern-setters**: Does one packet establish a convention (file structure, naming, error handling pattern, API shape) that other packets should follow? That packet must go first — it's the template.
- **Type/interface providers**: Does one packet create types, interfaces, schemas, or shared utilities that others import? It must complete before its consumers.
- **Integration points**: Does one packet create the wiring (routes, config, registry) that other packets' code plugs into? Determine whether the wiring should come first or last.
- **Test infrastructure**: Does one packet set up test utilities, fixtures, or helpers that others rely on?

#### 2d. Produce Execution Strategy

Organize packets into **execution groups** — ordered sets where:
- Packets WITHIN a group can run in **parallel** (no file overlaps, no implicit dependencies, no declared dependencies on each other).
- Groups run **sequentially** — Group 2 starts only after all packets in Group 1 are DONE.

Format your strategy as:

```
## Execution Strategy

### Group 1 (parallel)
- P1: [name] — [why it's in this group, e.g., "no dependencies, touches only src/models/"]
- P3: [name] — [why, e.g., "independent scope, no file overlap with P1"]

### Group 2 (parallel)
- P2: [name] — [why, e.g., "depends on P1's types, no overlap with P4"]
- P4: [name] — [why, e.g., "depends on P3's utilities, no overlap with P2"]

### Group 3 (sequential — single packet)
- P5: [name] — [why, e.g., "integration layer, touches files from P1-P4, must see all prior work"]

### Reasoning
- P1 before P2: P1 creates the User model that P2's service imports.
- P1 ∥ P3: No shared files. P1 works in src/models/, P3 works in src/utils/. Independent concerns.
- P5 last: Modifies src/routes/index.ts which imports from all prior packets.
```

**Be conservative.** When in doubt, make it sequential. Parallel execution saves time but a bad parallel decision causes merge conflicts or inconsistent patterns. It is always safer to serialize than to guess.

### Step 3: Create Task Board and Tracking

#### 3a. Capture Baseline
- Run `git rev-parse HEAD` to capture the **baseline commit SHA** before any work begins. Store this — you'll need it for diff reviews.

#### 3b. TodoWrite (Live Progress)
- Use `TodoWrite` to create a task list with one entry per packet. For each packet, create TWO entries:
  - `"P{N}: Build — <packet name>"`
  - `"P{N}: Validate — <packet name>"`
- This gives the user real-time visibility into both build and validation phases.

#### 3c. Persistent Task Board (Durable Record)
- Create (or update) a task board file at: `specs/packet-runs/<plan-filename>-run.md`
  - Example: if the plan is `specs/my-feature.md`, write to `specs/packet-runs/my-feature-run.md`
  - Create the `specs/packet-runs/` directory if it doesn't exist.
- The task board file MUST include the full Execution Strategy from Step 2. Contents:

```markdown
# Packet Run: [plan name]

**Plan**: [PLAN_PATH]
**Started**: [timestamp]
**Mode**: [plan|run]
**Baseline SHA**: [git SHA before execution]

## Execution Strategy

### Group 1 (parallel)
- P1: [name] — [reasoning]
- P3: [name] — [reasoning]

### Group 2 (parallel)
- P2: [name] — [reasoning]
- P4: [name] — [reasoning]

### Group 3 (sequential)
- P5: [name] — [reasoning]

### Reasoning
[Full reasoning from Step 2d]

## Packet Status

### P1: [name]
- **Status**: TODO
- **Group**: 1
- **Dependencies**: none
- **Criteria**: [list]
- **Builder Report**: —
- **Validator Report**: —
- **Diff Review**: —
- **SHA After Build**: —

### P2: [name]
- **Status**: TODO
- **Group**: 2
- **Dependencies**: P1
- **Criteria**: [list]
- **Builder Report**: —
- **Validator Report**: —
- **Diff Review**: —
- **SHA After Build**: —
```

#### 3d. Plan Mode Gate
- **If MODE is `plan`**: Present the task board to the user, including the execution strategy with full reasoning, the execution groups, and any inferred dependencies or synthesized criteria. Then STOP. Do not execute any packets.
- **If MODE is `run`**: Continue to Step 4.

### Step 4: Execute Packets

Execute packets **group by group** following the Execution Strategy from Step 2.

- **Within a group**: Launch all packets in parallel using multiple Task calls in a single message with `run_in_background: true`.
- **Between groups**: Wait for ALL packets in the current group to reach DONE (or BLOCKED/PARTIAL after retries) before starting the next group.
- **Single-packet groups**: Run normally (no parallelism needed).

For EACH packet within a group, follow Steps 4a through 4e:

#### 4a. Dependency Diff Review (for packets in Group 2+)

**This step is CRITICAL. It is what makes the orchestration adaptive.**

If this packet is in Group 2 or later, you MUST do the following BEFORE composing the builder prompt:

1. **Get the diff.** Run `git diff <baseline_or_dependency_SHA>..<current_HEAD> --stat` and `git diff <baseline_or_dependency_SHA>..<current_HEAD>` to see everything that changed since before the dependency packets ran. Use the SHA captured before the earliest dependency started (or the baseline SHA if this packet depends on Group 1 packets).

2. **Read the builder reports** from all dependency packets. These are stored in the task board file.

3. **Read the validator reports** from all dependency packets. Note any issues, warnings, or observations.

4. **Analyze and synthesize.** Ask yourself:
   - Did the dependency packets create files or APIs that this packet's plan assumed would have a specific shape? Did the actual shape differ?
   - Were there any surprises — additional files created, different function signatures, renamed exports, alternative patterns chosen?
   - Did any validator flag concerns that are relevant to this packet?
   - Does the packet's original description still make sense given what was actually built, or do the instructions need updating?

5. **Record your findings** as a `## Diff Review` note in the task board under this packet's section. These findings will be incorporated into the builder prompt in Step 4b.

**Even if you expect no surprises, do the diff review anyway.** The value is in catching the unexpected.

#### 4b. Dispatch Sonnet Builder Sub-Agent

- Mark the build task as `in_progress` in TodoWrite.
- Update the task board file: set this packet's Status to `BUILDING`.
- Launch a sub-agent using the `Task` tool with these parameters:
  - `subagent_type`: `"general-purpose"`
  - `model`: `"sonnet"`
  - `description`: A short label like `"P1: Build — <name>"`
  - `prompt`: A comprehensive prompt built from the **Builder Prompt Template** below

**Builder Prompt Template** — construct the prompt for each sub-agent:

```
You are implementing Packet {ID} of an implementation plan.

## Your Task
{packet name}

{packet description — full text from the plan}

## Plan Context
{1-3 sentence summary of the overall plan objective, pulled from the plan's top-level description/objective section}

## Files to Work On
{list each file path with a brief note on what to do with it}
- `path/to/file.ts` — Create this file with [description]
- `path/to/existing.ts` — Modify the [function/section] to [change]

## Acceptance Criteria
Your work MUST satisfy ALL of the following:
{numbered list of criteria from the packet}
1. [criterion 1]
2. [criterion 2]
3. [criterion 3]

## Constraints
{any scope limits or special instructions, or "None" if not applicable}

## Additional Context
{any code examples, architectural notes, patterns, or references from the plan}

## What Was Built Before You (if not the first group)
{This section is populated from the Dependency Diff Review in Step 4a.}
{Summarize what prior packets actually built — not what the plan SAID they would build, but what the orchestrator OBSERVED in the diff and reports.}
{Call out specifics: file paths created, function signatures, patterns used, anything this packet's work needs to align with.}
{If the diff revealed anything that changes or clarifies this packet's work, state it explicitly here.}

## Instructions
- Focus ONLY on this packet. Do not modify files outside your scope unless absolutely necessary.
- Follow existing code patterns and conventions already present in the codebase.
- Ensure ALL acceptance criteria are met before you finish.
- **Self-check before finishing**: Run the most relevant verification available for your changes (tests, lint, build, compile, or at minimum grep/read checks to confirm your changes are correct). If no test runner exists, do lightweight verification.
- If you discover a prerequisite that was NOT completed by a prior packet and blocks your work, say so clearly — do not silently work around it.

## Required Completion Report
You MUST end your response with this structured report:

### Packet Result
- **Status**: DONE | BLOCKED | PARTIAL
- **Packet**: {ID} — {name}
- **Summary**: [1-3 sentences: what changed and why]

### Criteria Coverage
- [x] criterion 1 — evidence: [what proves this is met]
- [x] criterion 2 — evidence: [what proves this is met]
- [ ] criterion 3 — NOT MET: [why]

### Changes
- `path/to/file` — [what changed]
- `path/to/new-file` — [created: description]

### Self-Check Results
- [command or check performed] — [result]

### Blockers (if any)
- [what's missing and what the coordinator needs to do]

Use DONE if all criteria are met. Use PARTIAL if some criteria are met but others are not. Use BLOCKED if you cannot proceed due to a missing prerequisite outside your control.
```

- When the builder returns, **capture the SHA**: run `git rev-parse HEAD` and store it as this packet's `SHA After Build` in the task board.
- Mark the build task as `completed` in TodoWrite.

#### 4c. Dispatch Validator Sub-Agent

- Mark the validate task as `in_progress` in TodoWrite.
- Update the task board file: set this packet's Status to `VALIDATING`.
- Launch a validator sub-agent using the `Task` tool:
  - `subagent_type`: `"general-purpose"`
  - `model`: `"sonnet"`
  - `description`: A short label like `"P1: Validate — <name>"`
  - `prompt`: A comprehensive prompt built from the **Validator Prompt Template** below

**Validator Prompt Template** — construct the prompt for each validator:

```
You are validating Packet {ID} of an implementation plan. A separate agent implemented this packet. Your job is to independently verify the work meets all acceptance criteria. You are a reviewer, not an implementer.

## Packet Details
**Name**: {packet name}
**Description**: {packet description}

## Acceptance Criteria to Verify
Each of these MUST be satisfied. Check every one:
{numbered list of criteria}
1. [criterion 1]
2. [criterion 2]
3. [criterion 3]

## Files That Should Have Been Created or Modified
{list of files from the packet}
- `path/to/file.ts` — Should contain [expected content/behavior]

## Verification Commands (if any)
{any test/lint/build commands specified in the plan or packet}

## Instructions
- You are READ-ONLY in spirit. Your purpose is to inspect and verify, not to fix.
- Read each file listed above. Confirm it exists and contains the expected changes.
- For each acceptance criterion, determine: MET or NOT MET with specific evidence.
- Run any verification commands specified above. Report exact output.
- If something is broken but easily fixable (typo, missing import), note it precisely but do NOT fix it.
- Be thorough but focused. Check what this packet required, not everything in the repo.

## Required Validation Report
You MUST end your response with this structured report:

### Validation Result
- **Verdict**: PASS | FAIL
- **Packet**: {ID} — {name}
- **Summary**: [1-3 sentence assessment]

### Criteria Verification
- [x] criterion 1 — VERIFIED: [specific evidence, e.g., "file exists at path, exports UserService class with create() and findById() methods"]
- [ ] criterion 2 — FAILED: [what's wrong, what was expected vs. what was found]

### Files Inspected
- `path/to/file` — [exists/missing, status, observations]

### Commands Run
- `[command]` — [exact output or summary]

### Issues Found (if any)
- [precise description of each issue, including file path and line if applicable]

Use PASS only if ALL criteria are verified. Use FAIL if ANY criterion is not met.
```

#### 4d. Orchestrator Decision

After the validator returns, YOU (the orchestrator) make the final call. Read the validator's report and decide:

**PASS (validator says all criteria verified):**
- Update TodoWrite: mark validate task as `completed`.
- Update the task board: set Status to `DONE`, record both builder and validator reports.
- Move to the next packet (or next group if this was the last packet in the group).

**FAIL (validator found unmet criteria):**
- **Resume the SAME builder sub-agent** using the `resume` parameter. Compose a targeted fix prompt:
  ```
  Task({
    description: "Fix: P{N} — <what failed>",
    prompt: "Your work on Packet {ID} was reviewed by an independent validator. The following criteria were NOT met:\n\n{paste the validator's specific findings for each failed criterion — what was expected vs. what was found}\n\nPlease fix these specific issues. Do not change anything that passed validation.\n\nProvide an updated completion report when done.",
    subagent_type: "general-purpose",
    model: "sonnet",
    resume: "<builder_agent_id>"
  })
  ```
- After the builder retries, **dispatch the validator again** (new validator, not resumed — fresh eyes).
- Allow up to **2 retry cycles** (build → validate → build → validate → build → validate) per packet.
- If still FAIL after 2 retries: update the task board with what passed and what didn't, and continue to the next packet.

**BLOCKED (builder reported it could not proceed):**
- Do NOT send to validator. There's nothing to validate.
- Update the task board: set Status to `BLOCKED`, record the blocker from the builder's report.
- Assess: can you resolve the blocker by dispatching a targeted sub-agent for the missing prerequisite? If it's a simple gap from a prior packet, do so. If it's truly external, mark it and continue.
- Move to the next packet (other packets may not depend on this one).

#### 4e. Update State
- Update the task board file with all reports and the final status.
- Ensure TodoWrite reflects the current state.
- If this was the last packet in the current group, proceed to the next group (back to Step 4a for the first packet in that group).

### Step 5: Final Report

After ALL packets have been processed, update the task board file with final results and present this report:

```
## Packet Execution Report

**Plan**: [plan name from the file]
**Plan File**: PLAN_PATH
**Result**: X/Y packets completed successfully
**Baseline SHA**: [starting SHA]
**Final SHA**: [ending SHA]

## Execution Strategy Used
[summary of groups and which packets ran in parallel vs sequential]

| # | Packet | Group | Build | Validate | Final Status | Retries | Notes |
|---|--------|-------|-------|----------|--------------|---------|-------|
| P1 | [name] | 1 | DONE | PASS | DONE | 0 | — |
| P3 | [name] | 1 | DONE | PASS | DONE | 0 | ran parallel with P1 |
| P2 | [name] | 2 | DONE | FAIL→PASS | DONE | 1 | [what was fixed] |
| P4 | [name] | 2 | PARTIAL | FAIL | PARTIAL | 2 | [what remains unmet] |
| P5 | [name] | 3 | BLOCKED | — | BLOCKED | 0 | [what blocked it] |

### Blockers (if any)
- P5: [what blocked it and what action is needed]

### Incomplete Criteria (if any)
- P4, criterion 3: [what failed and why after retries]

### Files Changed
- [consolidated list of all files created or modified across all packets]

### Acceptance Criteria Summary
- [x] [criterion 1] — P1 (verified by validator)
- [x] [criterion 2] — P1 (verified by validator)
- [x] [criterion 3] — P2 (verified after 1 retry)
- [ ] [criterion 4] — P4 (PARTIAL: reason)
- [ ] [criterion 5] — P5 (BLOCKED: reason)

### Task Board
Updated task board saved to: `specs/packet-runs/<filename>-run.md`
```

## Rules

1. **You are the orchestrator.** You do NOT write code. You do NOT validate code. All implementation is done by builder sub-agents. All validation is done by validator sub-agents. Your job is to read, reason, compose prompts, and make decisions.
2. **Execute by group.** Follow the execution strategy from Step 2. Packets within a group run in parallel. Groups run sequentially. Never start a group before the prior group is fully resolved.
3. **Always analyze before strategizing.** The execution strategy must be based on actual codebase analysis (file overlaps, implicit dependencies, pattern-setters), not just declared dependencies. Dispatch scouts to read the relevant files, then use their reports to decide.
4. **Be conservative with parallelism.** When in doubt about whether two packets can safely run in parallel, make them sequential. A wrong parallel decision causes merge conflicts or inconsistent patterns. It is always safer to serialize than to guess.
5. **Always diff-review before dependent packets.** When a new group starts, review the actual diff and reports from the prior group BEFORE composing any builder prompts. This is non-negotiable — it is the mechanism that keeps downstream packets aligned with what was actually built upstream.
6. **Validators are independent.** The validator sub-agent does NOT see the builder's self-report. It receives only the packet criteria, expected files, and verification commands. Its job is to independently confirm the work.
7. **Resume builders, fresh validators.** When a builder needs to retry, resume it (preserving context). When re-validating, launch a fresh validator (unbiased by prior validation).
8. **Track everything.** Use TodoWrite for live progress AND the persistent task board file for durable records. Record builder reports, validator reports, diff review notes, and execution strategy reasoning.
9. **Respect scope.** Do not add features, refactor code, or do work beyond what the plan specifies.
10. **Provide the plan context.** Every builder prompt must include the overall plan objective so the agent understands WHY it is building what it is building.
11. **Adapt between packets.** The builder prompt template is a starting point. Your primary value is in the `## What Was Built Before You` section and any adjustments you make based on the diff review. Enrich, clarify, and correct the prompt based on what you observe in the evolving codebase.
12. **Handle blockers gracefully.** A BLOCKED packet is not a failure — it's information. Record it, assess if you can unblock it, and keep the rest of the plan moving.
13. **Guard your context window.** Your context is for reasoning, not for storing raw source code. Delegate file reading to scout and builder sub-agents. When you need to review work (diff review, validation decisions), read reports and diffs — not entire source files. The moment you start reading large files yourself, you're doing a sub-agent's job.
