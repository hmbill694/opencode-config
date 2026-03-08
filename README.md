This is my current opencode config. It's an ongoing WIP but it has shown nice results for me so far.


# Agent System

This directory contains four cooperating agents that form a structured, human-in-the-loop software development pipeline. Every feature request flows through **Ideation → Planning → Writing → Testing**, with human approval gates at the key decision points.

---

## Agents

### 🎯 Orchestrator (`orchestrator.md`)
**Mode:** Primary &nbsp;|&nbsp; **Bash:** Ask &nbsp;|&nbsp; **Write:** Allow

The Orchestrator is the user's main interface. It drives the entire workflow, gates all approvals, and delegates work to the three subagents. It never writes code itself — only workflow artifacts (`_requirements.md`, `codebase.md`).

### 📐 Planner (`plan.md`)
**Mode:** Subagent &nbsp;|&nbsp; **Write:** Allow &nbsp;|&nbsp; **Edit/Bash:** Denied

A research agent. It inspects the repository's architecture and produces a step-by-step, checklist-formatted implementation plan, writing it **directly** to `agent-docs/plans/<slug>_implementation.md`. It returns control to the Orchestrator once the file is ready.

### ✍️ Writer (`writer.md`)
**Mode:** Subagent &nbsp;|&nbsp; **Bash:** Ask (runtime prompts user)

Executes the approved implementation plan step by step using `write` and `edit` tools. It tracks its own progress in a `_progress.md` file and hands off to the Tester when done. Bash commands (e.g. installing packages) trigger a native runtime approval prompt to the user — no manual relay needed.

### 🧪 Tester (`tester.md`)
**Mode:** Subagent &nbsp;|&nbsp; **Bash:** Allow &nbsp;|&nbsp; **Write:** Allow

Validates the build after every Writer handoff. It auto-detects the project type, runs the appropriate build command, and optionally runs linters and test suites. It feeds failures back to the Writer (reading `_progress.md` as the source of truth for completed steps) and escalates to the user after 3 consecutive failures.

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
Planner ─────────── reads repo, drafts + writes implementation plan
 │                  writes: agent-docs/plans/<slug>_implementation.md
 ▼
Orchestrator ──── reads plan, presents to user, waits for approval
 │
 ▼
Writer ──────────── implements plan step by step
 │                  writes: agent-docs/plans/<slug>_progress.md
 ▼
Tester ──────────── runs build, lint, tests
 │
 ├── FAIL  ──────── reads _progress.md, returns feedback to Writer  [up to 3x]
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
| Bash commands (Writer) | User (via runtime `bash: ask` prompt) |
| Circuit breaker (3 failures) | User (via Tester `question` tool) |

---

## Artifact Files

All generated files are written under `agent-docs/plans/` using a slug + Unix epoch timestamp prefix (e.g. `user_auth_1741306200`) to prevent overwrites across sessions. The slug epoch is obtained by running `date +%s` at requirements approval time.

`agent-docs/codebase.md` is committed to the repo (only `agent-docs/plans/` is gitignored) so the codebase map persists across sessions and clones.

| File | Written by | Purpose |
|---|---|---|
| `agent-docs/codebase.md` | Orchestrator | Persistent map of all source files (committed) |
| `<slug>_requirements.md` | Orchestrator | Approved feature requirements |
| `<slug>_implementation.md` | Planner | Approved step-by-step plan |
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
