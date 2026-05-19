---
name: ship
description: Full feature development lifecycle — PRD, critical review, task breakdown, implementation, and quality gates. Use when the user provides a feature description and wants to go all the way from idea to working, reviewed, tested code.
argument-hint: <feature description>
disable-model-invocation: true
allowed-tools: Read Glob Grep Bash(find *) Bash(cat *) Bash(ls *)
---

# Ship

Feature: $ARGUMENTS

You are driving this feature from idea to fully implemented, reviewed, and tested code. Apply the
`code-quality` skill to all code written in this workflow. Do not stop until every task is complete
and all quality gates pass. Follow every phase in order.

---

## Phase 1: Discovery

Run these in parallel before writing anything:

1. Read `CLAUDE.md` at the project root (if it exists) — internalize every convention, naming rule,
   and forbidden pattern. These are law for the rest of this workflow.

2. Detect the project stack:
```!
echo "=== package.json ===" && (cat package.json 2>/dev/null | head -30 || echo "none")
echo "=== pyproject.toml ===" && (cat pyproject.toml 2>/dev/null | head -20 || echo "none")
echo "=== Cargo.toml ===" && (cat Cargo.toml 2>/dev/null | head -10 || echo "none")
echo "=== go.mod ===" && (cat go.mod 2>/dev/null | head -5 || echo "none")
echo "=== Makefile targets ===" && (grep -E "^[a-z].*:" Makefile 2>/dev/null | head -15 || echo "none")
echo "=== biome ===" && (cat biome.json 2>/dev/null || cat biome.jsonc 2>/dev/null || echo "none")
echo "=== eslint ===" && (ls .eslintrc* eslint.config.* 2>/dev/null | head -3 || echo "none")
```

3. From the discovery output, record:
   - `DETECTED_LANG` — primary language/framework
   - `TYPE_CHECK_CMD` — type checker command (e.g. `tsc --noEmit`, `mypy .`), or "none"
   - `TEST_CMD` — test runner (e.g. `npm test`, `pytest`, `cargo test`)
   - `FORMAT_CMD` — `biome format --write .` if Biome configured, else `prettier --write .`, else "none"
   - `LINT_CMD` — `biome check --apply .` if Biome configured, else `eslint .`, else from package.json, else "none"

4. Create `.claude/ship/` in the project if it does not exist.

5. Create a TodoWrite task list: Discovery, PRD Draft, PRD Review, Task Breakdown, Approval Gate,
   plus one placeholder per implementation task (fill in after Phase 4).

---

## Phase 2: PRD Draft

Write `.claude/ship/PRD.md`. Be concrete — no vague language, no filler.

Required sections:

**Problem Statement** — What problem this solves, who is affected, why it matters now.
**Goals** — Concrete, measurable outcomes.
**Non-Goals** — Explicit scope limits — what this does NOT do.
**User Stories** — At least 3: "As a <role>, I want <action> so that <outcome>."
**Technical Approach** — Implementation strategy, which parts of the codebase are touched, data flow.
**Edge Cases & Risks** — Failure modes, error states, security implications, performance concerns.
**Open Questions** — Unresolved decisions that need input before or during implementation.

Mark PRD Draft complete in TodoWrite.

---

## Phase 3: PRD Critical Review

Launch a subagent with this exact prompt:

> "You are a harsh senior engineering critic. Read `.claude/ship/PRD.md` and poke holes in it. For
> each section identify: missing error cases, ambiguous requirements, scope creep risks, security or
> performance implications, unstated assumptions, and anything a junior engineer would misinterpret.
> Return a structured bullet list of critiques. Be thorough and unsparing."

After the agent returns:
1. Address every substantive critique by updating `.claude/ship/PRD.md`.
2. Append `## Revision Notes` at the bottom — list each critique and how it was resolved.

Mark PRD Review complete in TodoWrite.

---

## Phase 4: Task Breakdown

Write `.claude/ship/TASKS.md`. Rules:
- Tasks ordered so dependencies flow forward (nothing requires code from a later task).
- Each task is one focused implementation unit.
- Every task block includes a dedicated test subtask — never combine implementation and tests.
- **Sweep for docs the implementation will make factually wrong.** Before finalizing the
  task list, grep the repo's existing docs (top-level `README.md`/`CLAUDE.md`, anything
  under `docs/`, `agent.md` / `AGENTS.md` files, `.env.example`, feature-flag tables,
  roadmaps, integration-status files) for content the planned changes will invalidate —
  closed todos, "no way to do X" claims that will no longer be true, method enumerations
  that will be incomplete, env-var lists that will be missing entries. Add a single
  doc-update task covering all of them, scheduled after the implementation tasks but
  before the verification gate so wording can cite as-shipped behavior. Do NOT create new
  docs or add doc-touches for the sake of completeness — only fix existing inaccuracies.
  If the sweep finds nothing to update, skip the task; do not invent work.

Format:
```
# Task Breakdown: $ARGUMENTS

## Task 1: [Title]
- [ ] 1.1: [Specific action]
- [ ] 1.2: [Specific action]
- [ ] 1.3: Write tests for Task 1

## Task 2: [Title]
- [ ] 2.1: [Specific action]
- [ ] 2.2: Write tests for Task 2
```

Add one TodoWrite entry per task. Mark Task Breakdown complete.

---

## APPROVAL GATE — STOP HERE

**Do not write any code or modify any project files until the user explicitly approves.**

Present this to the user:

---
**Feature:** $ARGUMENTS

**PRD:** `.claude/ship/PRD.md`
**Tasks:** `.claude/ship/TASKS.md`

[Print the full contents of TASKS.md]

**Stack:** `DETECTED_LANG` | Type check: `TYPE_CHECK_CMD` | Tests: `TEST_CMD` | Lint: `LINT_CMD`

Reply with:
- **`approved`** — begin implementation
- **`edit prd [instructions]`** — revise, then re-present for approval
- **`stop`** — cancel
---

Wait for the user's reply before proceeding.

---

## Phase 5: Implementation Loop

After the user replies `approved`, mark Approval Gate complete in TodoWrite and begin.

**For each task in TASKS.md, in order:**

### Step A — Implement

Launch a subagent:

> "Implement [Task N: title] for this feature: $ARGUMENTS. PRD: `.claude/ship/PRD.md`. Tasks:
> `.claude/ship/TASKS.md`. Implement only the subtasks under Task N. Follow CLAUDE.md conventions
> if present. Apply code-quality standards: names reveal intent, no implicit `any`, no swallowed
> errors, no floating promises, no magic values, small focused functions, no dead code. After all
> subtasks, write tests for the new functionality. Return a list of all files created or modified."

### Step B — Quality Gates (run in order — fix before advancing)

**Gate 1 — Type check** (skip if TYPE_CHECK_CMD is "none"):
Run `TYPE_CHECK_CMD`. Fix all errors.

**Gate 2 — Test suite**:
Run `TEST_CMD`. All tests must pass. Fix failures — never comment out or skip tests.

**Gate 3 — New tests**:
Inspect files modified by the implementation agent. Verify new test code covers the new
functionality (not just that existing tests still pass). Write tests now if missing.

**Gate 4 — Parallel code review** (3 agents simultaneously, only changed files):
- **Agent A (Bugs):** "Review these files for bugs, incorrect logic, and silent failures. Flag only
  confirmed issues. Files: [list]"
- **Agent B (Conventions):** "Review these files for CLAUDE.md compliance. Quote the exact rule for
  each violation. Files: [list]"
- **Agent C (Edge cases):** "Review these files for uncovered edge cases — null inputs, empty
  collections, large payloads, concurrent access, auth failures. Flag only real gaps. Files: [list]"

Fix any confirmed, clearly real issue. Log unfixed issues as known trade-offs in TASKS.md.

**Gate 5 — Format + Lint**:
Run `FORMAT_CMD` if set. Run `LINT_CMD` if set. All lint errors must be zero. Do not suppress rules
without an explanatory comment.

### Step C — Mark complete

Mark all subtasks `[x]` in TASKS.md. Mark task complete in TodoWrite. Proceed to the next task.

---

## Phase 6: Final Summary

After all tasks complete, print:

```
## Ship Complete: $ARGUMENTS

### What was built
[Bullet list of implemented capabilities]

### Files modified
[All files touched across all tasks]

### Tests added
[Names or descriptions of new test cases]

### Quality gates
- Type check: [passed / skipped — reason]
- Test suite: [passed — N tests]
- Code review: [N issues found, M fixed, K logged as trade-offs]
- Format/Lint: [passed / skipped — reason]

### Open concerns
[Issues flagged by review agents that were not fixed, with rationale]
```

Mark all TodoWrite items complete.
