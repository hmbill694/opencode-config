# Agent System

This directory contains four cooperating agents that form a structured, human-in-the-loop software development pipeline. Every feature request flows through **Ideation → Planning → Writing → Testing**, with human approval gates at the key decision points.

---

## Agents

### 🎯 Orchestrator (`orchestrator.md`)
**Mode:** Primary &nbsp;|&nbsp; **Bash:** Ask

The Orchestrator is the user's main interface. It drives the entire workflow, gates all approvals, and delegates work to the three subagents. It never writes code itself.

### 📐 Planner (`plan.md`)
**Mode:** Subagent &nbsp;|&nbsp; **Write/Edit/Bash:** Denied

A read-only research agent. It inspects the repository's architecture and produces a step-by-step, checklist-formatted implementation plan. It returns the draft to the Orchestrator and never touches the filesystem.

### ✍️ Writer (`writer.md`)
**Mode:** Subagent &nbsp;|&nbsp; **Bash:** Denied (escalate to Orchestrator)

Executes the approved implementation plan step by step using `write`, `edit`, and `patch` tools. It tracks its own progress in a `_progress.md` file and hands off to the Tester when done. It cannot run bash commands — it must request permission from the Orchestrator first.

### 🧪 Tester (`tester.md`)
**Mode:** Subagent &nbsp;|&nbsp; **Write/Edit:** Denied

Validates the build after every Writer handoff. It auto-detects the project type, runs the appropriate build command, and optionally runs linters and test suites. It feeds failures back to the Writer and escalates to the user after 3 consecutive failures.

---

## Workflow

```
User
 │
 ▼
Orchestrator ──── workshops idea, confirms requirements
 │
 │  writes: agent-docs/plans/<slug>_requirements.md
 ▼
Planner ─────────── reads repo, drafts implementation plan
 │
 ▼
Orchestrator ──── presents plan, waits for approval
 │
 │  writes: agent-docs/plans/<slug>_implementation.md
 ▼
Writer ──────────── implements plan step by step
 │                  writes: agent-docs/plans/<slug>_progress.md
 ▼
Tester ──────────── runs build, lint, tests
 │
 ├── FAIL  ──────── returns feedback to Writer  [up to 3x]
 │                  reads/writes: agent-docs/plans/<slug>_state.json
 │
 ├── WARN  ──────── reports warnings, returns to Orchestrator
 │
 └── SUCCESS ─────► Orchestrator asks user: done or refine?
```

### Approval Gates

| Gate | Who approves |
|---|---|
| Requirements | User (via Orchestrator `question` tool) |
| Implementation plan | User (via Orchestrator `question` tool) |
| Bash commands (Writer) | User (via Orchestrator `question` tool) |
| Circuit breaker (3 failures) | User (via Tester `question` tool) |

---

## Artifact Files

All generated files are written under `agent-docs/plans/` using a slug + timestamp prefix (e.g. `user_auth_170870`) to prevent overwrites across sessions.

| File | Written by | Purpose |
|---|---|---|
| `<slug>_requirements.md` | Orchestrator | Approved feature requirements |
| `<slug>_implementation.md` | Orchestrator | Approved step-by-step plan |
| `<slug>_progress.md` | Writer | Tracks completed steps (survives context resets) |
| `<slug>_state.json` | Tester | Tracks consecutive failure count for circuit breaker |

---

## Circuit Breaker

If the Writer→Tester loop fails **3 times in a row**, the Tester halts, presents a full error summary, and asks the user to choose:

1. **Retry fresh** — reset counter, try again
2. **Abort task** — stop entirely
3. **Fix manually** — user fixes the issue, then asks Tester to re-run

The state file is deleted after the user responds so a fresh retry starts at Attempt 1.

---

## Tester Result Hierarchy

| Result | Meaning | Action |
|---|---|---|
| `FAIL` | Build doesn't compile | Loop back to Writer (up to 3×) |
| `WARN` | Lint / type / test failures | Pass with warnings to Orchestrator |
| `SUCCESS` | All validations pass | Return to Orchestrator for wrap-up |
