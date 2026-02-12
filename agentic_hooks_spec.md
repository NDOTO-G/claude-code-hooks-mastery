# Spec: Agentic Hooks Layer (Claude Code Hooks Mastery–inspired) for Your Fork

**Purpose:** Provide a coding agent enough clarity to produce an implementation plan and then build a working, auditable “agentic layer” in **your fork** (based on *claude-code-hooks-mastery* patterns), with portability targets for **OpenCode** and **Codex**.

---

## 0) High-level outcome

Create a repo-local “agentic layer” that makes agent runs:

- **Deterministic** (can’t mark done unless gates pass)
- **Auditable** (every action logged, every decision traceable)
- **Composable** (commands/prompt templates + agents + validators)
- **Portable-ish** (Claude Code hooks where available; “gates as steps” elsewhere)

This system should let you run:

- A planning command that creates a spec/plan file in a fixed schema
- A builder/validator “team” that implements steps
- Validators that enforce structure, tests, lint, and policy
- A durable task board + JSONL telemetry so progress survives sessions

---

## 1) Non-goals

- Building a full GUI, web app, or hosted service
- Implementing a new LLM provider layer (use whatever the CLI supports)
- Perfect parity across Claude Code / Codex / OpenCode (aim for functional equivalence)

---

## 2) Key concepts and terms

- **Packet / Step:** A smallest independently verifiable unit of work.
- **Gate:** A deterministic check that must pass before advancing (tests/lint/spec schema/policy).
- **Builder agent:** Allowed to edit code; focuses on implementation.
- **Validator agent:** Read-only; produces verification verdicts + evidence.
- **Orchestrator command:** Reads plan/spec, decomposes into packets, dispatches builder/validator, updates task board + logs.
- **Hook (Claude Code):** Pre/Post/Stop intercept points used to run gates and block completion.

---

## 3) Deliverables

### 3.1 Repo additions (source-controlled)

Create an `agentic/` (or `.agentic/`) directory that is entirely repo-local and portable:

```
agentic/
  README.md
  commands/
    plan_w_team.md
    run_plan.md
    validate.md
  agents/
    builder.md
    validator.md
    planner.md
  hooks/                # Claude Code hook configs + scripts (when using Claude Code)
    settings.json       # or equivalent Claude config file
    scripts/
      pre_tool_policy.py
      post_tool_validate.py
      stop_gate.py
  validators/
    validate_new_file.py
    validate_file_contains.py
    validate_markdown_schema.py
    validate_repo_health.sh
    validate_tests.sh
  templates/
    plan_template.md
    packet_run_template.md
  state/
    packet-run.md        # durable task board (append-only)
  logs/
    README.md
    orchestrator.jsonl
    builder.jsonl
    validator.jsonl
    tool.jsonl
```

> Adjust folder names to match your CLI: Claude Code uses `.claude/`; OpenCode uses `.opencode/`. The coding agent should implement *adapters* to generate the right layout per target.

### 3.2 A single “shareable spec file” output by the planner

Planner must generate a plan file (e.g., `specs/<plan_name>.md`) that follows a strict schema.

### 3.3 A runner that executes the plan and enforces gates

- For **Claude Code**: implement real **Stop** gate behavior via hooks (block stopping until validators pass).
- For **OpenCode/Codex**: implement “gates as steps” (runner refuses to advance until gates pass and are logged).

---

## 4) Functional requirements

### FR-1: Plan creation (self-validating)
**User story:** As an operator, I run a command with a short goal statement and receive a plan/spec file that is structurally valid.

**Requirements:**
- Command: `/plan_w_team <plan_name> "<goal>"` (or closest equivalent)
- Must create: `specs/<plan_name>.md`
- Must include required sections (see §6)
- Must be validated automatically by a validator (hook or step)

**Acceptance criteria:**
- If file missing OR schema missing sections → command fails with actionable error.
- If valid → command succeeds, logs include verdict + path.

---

### FR-2: Team orchestration (builder + validator)
**Requirements:**
- Builder has write privileges; validator is read-only.
- Runner dispatches builder tasks per packet.
- After each packet, validator reviews:
  - `git diff` evidence
  - Test/lint outputs
  - Spec compliance

**Acceptance criteria:**
- Every packet yields a validator verdict: PASS/FAIL with evidence links (file paths, diffs, command outputs).

---

### FR-3: Gates (deterministic completion)
**Minimum gates:**
- Spec schema gate (plan contains required sections)
- Repo health gate (clean working tree or explicit allowed dirty state)
- Lint gate (configurable)
- Test gate (configurable)
- Safety/policy gate (deny dangerous commands, secrets reads, etc.)

**Acceptance criteria:**
- A packet cannot be marked DONE unless gates pass.
- A plan cannot be marked COMPLETE unless all packets are DONE and final gates pass.

---

### FR-4: Durable task board (`packet-run.md`)
**Requirements:**
- Append-only or audit-preserving edits.
- Records:
  - Packet ID, description, dependencies
  - Status: TODO / IN_PROGRESS / BLOCKED / DONE / FAILED
  - Builder outputs summary
  - Validator verdict + evidence
  - Git commit or diff references (SHA if committing)

**Acceptance criteria:**
- Runner can resume from existing `packet-run.md` without losing context.

---

### FR-5: Telemetry logging (JSONL)
**Requirements:**
- `logs/orchestrator.jsonl`: plan start/end, packet dispatch, status changes
- `logs/tool.jsonl`: shell command runs + exit codes
- `logs/builder.jsonl`: prompts (hashed), outputs summary, files touched
- `logs/validator.jsonl`: verdict, evidence list

**Acceptance criteria:**
- A third party can reconstruct what happened from logs + git history.

---

## 5) Target platform adapters

### 5.1 Claude Code (native hooks)
Implement:
- `PreToolUse` policy checks (block risky commands)
- `PostToolUse` validations (lint/test on relevant file changes)
- `Stop`/`SubagentStop` gates to prevent concluding until validators pass

**Note:** Hooks should emit structured JSON that can block/continue where supported.

### 5.2 OpenCode
Implement:
- Commands in `.opencode/command/*.md` using `$ARGUMENTS` and file injection
- Agent roles in `.opencode/agent/*`
- Permissions rules mirroring policy gates

### 5.3 Codex (CLI or SDK)
Implement:
- A runner script that performs gates as explicit steps
- Optional: a future adapter using Codex SDK for structured orchestration
- Approval modes / rules as safety layer where available

---

## 6) Plan/spec schema (required sections)

Every plan file `specs/<plan_name>.md` MUST contain:

1. **Title**
2. **Goal / Success Criteria** (bullet list)
3. **Assumptions**
4. **Constraints / Policies**
5. **Context / Existing System Notes**
6. **Architecture Sketch**
7. **Work Breakdown (Packets)**
   - Each packet has: `id`, `description`, `inputs`, `outputs`, `dependencies`, `gates`, `tests`
8. **Validation Strategy**
9. **Risk Register**
10. **Open Questions**
11. **Done Definition**

Validator scripts must enforce presence + minimal structure.

---

## 7) Safety and policy requirements

### SR-1: Secrets and sensitive files
- Deny reading `.env`, credentials, SSH keys unless explicitly whitelisted.
- Log policy violations.

### SR-2: Destructive commands
- Deny `rm -rf`, `dd`, disk formatting, etc.
- Deny network calls by default unless explicitly enabled.

### SR-3: Doom loop guard
- Detect repeated identical tool calls; break and require human input or re-plan.

---

## 8) Implementation outline (what the coding agent should produce)

The coding agent should output:

1. **Repo analysis**: current structure in your fork, what exists, what needs adding.
2. **Adapter decision**: which primary runtime (Claude Code vs OpenCode vs Codex) you’re targeting first.
3. **File-by-file plan**: create/modify files in `agentic/` + platform folders.
4. **Validator implementation plan**:
   - Python validators (`validate_new_file.py`, `validate_file_contains.py`, schema validator)
   - Shell validators (tests/lint wrappers)
5. **Runner/orchestrator plan**:
   - How it reads plan, writes task board, dispatches builder/validator, retries on failure.
6. **Hook/permissions plan**:
   - Claude hooks config OR OpenCode permissions rules OR Codex approval modes.
7. **Demo scenario**:
   - A small sample plan with 2–3 packets
   - Run command → produce spec → execute packets → pass gates → logs created
8. **Acceptance test checklist** (see §9)

---

## 9) Acceptance tests (definition of “working”)

### AT-1: Plan generation gate
- Running planner produces `specs/demo-plan.md`
- Schema validator passes

### AT-2: Builder/validator split
- Builder changes a file
- Validator (read-only) reports PASS with evidence

### AT-3: Stop-gate / no false completion
- Introduce a failing test
- Runner refuses to mark DONE; logs record FAIL; builder re-runs and fixes

### AT-4: Logs + task board are complete
- `packet-run.md` updated
- JSONL logs exist with at least one record per packet

### AT-5: Resume behavior
- Interrupt mid-run; re-run; system resumes at correct packet

---

## 10) UX requirements (operator workflow)

- “One command to plan”
- “One command to run”
- “One command to validate”
- Clear, minimal output with pointers to:
  - plan file
  - packet-run.md
  - failing validator output

---

## 11) Suggested initial MVP scope

Implement first for **one platform** (recommended: the CLI you’ll use daily) with:

- Plan schema validator
- Task board + runner
- Builder/validator roles (even if manual dispatch initially)
- Lint/test gates

Then add:
- Claude native hooks (if you use Claude Code)
- OpenCode command templates + permissions
- Codex adapter

---

## 12) Notes for the coding agent (how to proceed)

- Prefer **small tracer-bullet slices**: get plan → one packet → validate → log.
- Keep everything repo-local and versioned.
- Make validators **fast** and deterministic.
- Make logs **append-only** where possible.
- Avoid complex frameworks until MVP works.

---

## 13) Appendix: example packet format

```yaml
Packet:
  id: P1
  description: "Add plan schema validator"
  inputs:
    - specs/<plan>.md
  outputs:
    - agentic/validators/validate_markdown_schema.py
  dependencies: []
  gates:
    - "python -m agentic.validators.validate_markdown_schema specs/<plan>.md"
  tests:
    - "python -m pytest -q"
```

---

## 14) Next step prompt (copy/paste to a coding agent)

**Task:** Produce an implementation plan + file-level changes for the above spec inside *this fork*.  
**Constraints:** Keep changes repo-local; prefer Python for validators and a small runner script; add adapters for Claude Code + OpenCode where feasible.

**Deliverables:** A PR-ready set of files + a `docs/IMPLEMENTATION_PLAN.md` describing the rollout in phases, plus a runnable demo.

