---
name: build-castle
description: Executes READY (default label `ready-for-agent`) issues sequentially using /tdd and /review. Reads tracker adapter config from docs/agents/, sorts by dependencies, runs TDD + review per issue, manages status transitions, auto-commits on success. Run after /to-issues has populated the issue tracker. The /build-castle verb of the Castle (claude-castle) bundle.
---

# Build Castle

Execute all `READY` issues in dependency order using TDD and review. Each issue = one bucket of sand packed into the castle.

## Prerequisites

`docs/agents/tracker-adapter.md` and `docs/agents/build-castle.md` must exist. If either is missing, halt immediately with:

```
Foundation not laid — run /lay-foundation first.
```

This skill talks to the issue tracker only through the tracker contract — see `skills/build-castle/tracker-contract.md` for verb signatures and logical status names. The concrete adapter at `docs/agents/tracker-adapter.md` implements those verbs for this repo's tracker.

## State machine

Every issue moves between logical statuses via the events below. Step 5 narrates *when* each event fires; this table is the single source for *what* each event causes. Status transitions happen by calling `set-status(number, FROM, TO)`.

Retry counter is per-issue, in-memory only. Starts at `0` when an issue first enters `IN_PROGRESS` in this run. `B` = the configured `review_retry_budget` from `docs/agents/build-castle.md`.

| From | Event | To | Side-effect |
|------|-------|-----|-------------|
| `READY` | `start` | `IN_PROGRESS` | run TDD; reset retries to 0 |
| `IN_PROGRESS` | `tdd-failed` | `FAILED` | halt; print "Fix and re-run" |
| `IN_PROGRESS` | `review-failed` (retries `< B`) | `IN_PROGRESS` | increment retries; re-run TDD with findings as follow-up |
| `IN_PROGRESS` | `review-failed` (retries `== B`) | `NEEDS_HUMAN` | halt; print findings |
| `IN_PROGRESS` | `committed` | `COMPLETED` | `git commit`; print short hash |

`B = 0` means a single review failure escalates immediately (no retries). `B = 1` retries once. Resuming an interrupted `IN_PROGRESS` issue on the next run also resets retries to 0.

Issues already in `IN_PROGRESS` at start of `/build-castle` (interrupted prior run) skip the `start` event and resume at TDD directly.

## Models

`/build-castle` invokes two subagents per issue: `/tdd` and `/review`. Both are called via the Agent tool with `subagent_type: general-purpose`. The configured model is passed to the Agent tool. The configured level maps to a thinking keyword that is prefixed onto the subagent prompt.

Level → keyword:

| level   | keyword       |
|---------|---------------|
| `none`  | *(no prefix)* |
| `low`   | `think`       |
| `medium`| `think hard`  |
| `high`  | `think harder`|
| `xhigh` | `ultrathink`  |

Valid models: `opus | sonnet | haiku`. Valid levels: `none | low | medium | high | xhigh`.

Per-repo settings live in `docs/agents/build-castle.md` under a `## Models` section with this exact shape:

```markdown
## Models

| role   | model  | level |
|--------|--------|-------|
| tdd    | sonnet | high  |
| review | opus   | high  |
```

Exactly two rows are required: `tdd` and `review`.

## Process

### 1. Read config

If either of the two files below is missing, halt immediately with `"Foundation not laid — run /lay-foundation first."` — do not partial-read, do not infer defaults.

- `docs/agents/tracker-adapter.md` — how to invoke tracker verbs (`list-by-status`, `read`, `set-status`, `get-dependency-edges`).
- `docs/agents/build-castle.md` — `review_retry_budget` setting and `## Models` table.

Read the `## Models` table from `docs/agents/build-castle.md`. Validate strictly:

- Missing table → halt with `"Models table missing in docs/agents/build-castle.md — re-run /lay-foundation or add manually."`
- Table must contain exactly the roles `tdd` and `review` — one row each. Rows may appear in any order. A duplicate role → halt with `"Duplicate role '<r>' in Models table."` A missing role → halt with `"Missing role '<r>' in Models table. Expected both: tdd, review."`
- `model` value must be one of `opus | sonnet | haiku`. Otherwise halt with `"Invalid model '<x>' for role '<r>' in build-castle.md. Expected: opus, sonnet, haiku."`
- `level` value must be one of `none | low | medium | high | xhigh`. Otherwise halt with `"Invalid level '<x>' for role '<r>' in build-castle.md. Expected: none, low, medium, high, xhigh."`

### 2. Fetch issues

Call `list-by-status([READY, IN_PROGRESS, COMPLETED, FAILED, NEEDS_HUMAN])`. The first two drive execution; the others are needed for the full dependency graph.

### 3. Topological sort

Call `get-dependency-edges()`. Sort fetched issues so every blocker comes before every blocked.

If a `READY` or `IN_PROGRESS` issue's transitive blocker is `FAILED` or `NEEDS_HUMAN`, mark it blocked in the plan — do not execute.

If a dependency cycle is detected, report it and halt.

### 4. Print execution plan

Before executing anything, print the full ordered plan:

```
Execution plan (N issues):
  [1] #001 example bucket A                          ✓ already done
  [2] #002 example bucket B                          ✓ already done
  [3] #003 example bucket C                          ready
  [4] #005 example bucket E                          ⚠ blocked by #004 (failed)
```

If no issues are `READY` AND no issues are `IN_PROGRESS`, halt with:

```
No READY issues found. Nothing to execute.
```

Otherwise ask user to confirm before proceeding.

### 5. Execute each issue in order

For each `READY` or `IN_PROGRESS` issue in sorted order (skip `COMPLETED`; halt if a required blocker is `FAILED`/`NEEDS_HUMAN`):

#### 5a. Start

Fire `start` event: `set-status(number, READY, IN_PROGRESS)`. (For an `IN_PROGRESS` issue being resumed, skip — already in state.)

Print header:

```
[N/Total] <issue title> (#<number>)
```

#### 5b. Run TDD

Invoke `/tdd` via the Agent tool: `subagent_type: general-purpose`, `model: <tdd.model>` (from the Models table). Prompt body:

```
<keyword>

Run the /tdd skill on this issue:

<full issue body>
```

`<keyword>` is the level→keyword mapping from the `## Models` section. Omit the keyword line entirely if `tdd.level == none`.

Stream step-level output (model + level visible on entry):

```
  ▶ TDD (<tdd.model>, <tdd.level>) — writing failing tests...
  ✓ TDD green — tests pass
  ✓ TDD refactor — clean
```

**On TDD failure:** fire `tdd-failed` event. `set-status(number, IN_PROGRESS, FAILED)`. Print:

```
  ✗ TDD failed — <error summary>
  ⚠ Halted. Issue #<number> marked failed. Fix and re-run /build-castle.
```

Stop all execution.

#### 5c. Run review

Invoke `/review` via the Agent tool: `subagent_type: general-purpose`, `model: <review.model>` (from the Models table). Prompt body:

```
<keyword>

Run the /review skill on the changes just made for this issue. Inspect the working-tree diff (`git diff`) to see what was changed.

Issue context:

<full issue body>
```

`<keyword>` mapped from `review.level` (omit the line if `review.level == none`).

Stream output:

```
  ▶ Review (<review.model>, <review.level>)...
```

**On review pass:** continue to 5d.

**On review failure** — fire `review-failed`. Compare in-memory retry counter against `review_retry_budget` (`B`):

- **`retries < B`**: increment counter; pass findings verbatim back to TDD agent as a follow-up prompt; retry the full TDD cycle. Status stays `IN_PROGRESS`.
- **`retries == B`**: `set-status(number, IN_PROGRESS, NEEDS_HUMAN)`. Print findings; stop all execution.

  ```
    ✗ Review failed — <findings summary>
    ⚠ Halted. Issue #<number> marked needs-human. Fix and re-run /build-castle.
  ```

#### 5d. Commit

Fire `committed`. Auto-commit all changes with the issue number in the message:

```
git add -A && git commit -m "<issue title> (#<number>)"
```

`set-status(number, IN_PROGRESS, COMPLETED)`. Print:

```
  ✓ Review passed
  ✓ Committed: <short-hash>
```

### 6. Summary

After all issues processed (or on halt), print summary:

```
Done. N issues completed, N commits made.
```

Or on halt:

```
Stopped at issue #<number>. N issues completed before halt.
Re-run /build-castle after fixing the issue to resume.
```

## Resumability

On re-run, `COMPLETED` issues are skipped. `IN_PROGRESS` issues (interrupted mid-run) keep their status and restart from the beginning of TDD — no `set-status` call needed before restarting.

The execution plan (step 4) always shows the full picture including already-completed issues with `✓ already done`.
