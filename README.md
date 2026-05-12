<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/logo-dark.svg">
    <img src="assets/logo-light.svg" alt="Claude Castle" width="562">
  </picture>
</p>

# claude-castle

> *Build castles without the moat — no Docker, no TypeScript SDK, no infrastructure.*

A Claude Code skill bundle for structured AI-driven development. Wraps the full pipeline from idea to committed code, packing one issue at a time like buckets of sand.

> ⚠ **Hard dependency on [mattpocock/skills](https://github.com/mattpocock/skills).** Four of the pipeline's six steps (`/grill-me`, `/to-prd`, `/to-issues`, `/tdd`) live in Matt Pocock's skill collection and must be installed **before** anything in this repo will work end-to-end. Neither the plugin install nor the manual copy below pulls those skills automatically — see step 1.

## Pipeline

```
/grill-me → /to-prd → /to-issues → /build-castle
```

| Step | What it does |
|------|-------------|
| `/grill-me` | Interviews you about your plan until design is locked |
| `/to-prd` | Converts the grilling session into a PRD, published to your issue tracker |
| `/to-issues` | Breaks the PRD into dependency-ordered, AFK-ready issues |
| `/build-castle` | Executes issues sequentially: TDD → review → commit, per issue |

## vs Sandcastle

[Sandcastle](https://github.com/mattpocock/sandcastle) orchestrates agents in isolated Docker containers via a TypeScript SDK. It is infrastructure-first and deliberately unopinionated about workflow.

This bundle is methodology-first: it runs entirely inside Claude Code with no containers, no SDK, and no build step. The isolation primitive is Claude Code's native git worktree support. The workflow is opinionated — plan, spec, TDD, review, commit — and each stage is a discrete human checkpoint.

Use Sandcastle if you need parallel agents or Docker isolation. Use this if you want a structured planning-to-execution pipeline that runs wherever Claude Code runs.

## Install

### 1. Install dependency skills (from mattpocock/skills) — REQUIRED FIRST

`/build-castle` invokes `/tdd` and `/review` as subagents, and the rest of the pipeline (`/grill-me`, `/to-prd`, `/to-issues`) lives in Matt Pocock's collection. Without these, the Castle bundle has nothing to run. Install them first:

```bash
# In your Claude Code session:
/find-skills grill-me
/find-skills to-prd
/find-skills to-issues
/find-skills tdd
```

Or clone `mattpocock/skills` and copy the relevant skill directories to `~/.claude/skills/`. Source: https://github.com/mattpocock/skills

### 2. Install this bundle

**Option A — as a Claude Code plugin** (recommended):

```
/plugin marketplace add jeroenvdwaal/claude-castle
/plugin install claude-castle@claude-castle
```

Skills become available as `/claude-castle:lay-foundation` and `/claude-castle:build-castle`. Update with `/plugin update claude-castle`.

> The plugin install does **not** pull the mattpocock/skills dependencies. Step 1 above is still required.

**Option B — manual copy**:

```bash
git clone https://github.com/jeroenvdwaal/claude-castle
cp -r claude-castle/skills/lay-foundation ~/.claude/skills/
cp -r claude-castle/skills/build-castle  ~/.claude/skills/
```

Skills are available unnamespaced: `/lay-foundation`, `/build-castle`. The two install modes coexist without conflict (the namespaced ones win on collision).

### Updating

For plugin installs:

```
/plugin marketplace update claude-castle   # refresh marketplace.json from GitHub
/plugin update claude-castle               # install the latest plugin version
```

`marketplace update` is only needed when the marketplace metadata itself changes (new plugin entry, source URL change). For routine plugin upgrades, `plugin update` is enough.

For manual installs: re-run the `cp -r` commands in Option B after `git pull`ing this repo.

### 3. Configure your repo

In any repo you want to use the pipeline:

```
/lay-foundation
```

This sets up your tracker adapter, `review_retry_budget`, and label-map overrides in `docs/agents/`.

## Usage

```
# 1. Grill your idea into a locked design
/grill-me

# 2. Publish a PRD to your issue tracker
/to-prd

# 3. Break the PRD into issues
/to-issues

# 4. Pack the buckets — execute all ready-for-agent issues
/build-castle
```

`/build-castle` executes issues in dependency order. For each issue: TDD (red → green → refactor), review, auto-commit. Halts on failure and resumes cleanly on re-run.

## Configuration

`/lay-foundation` writes to `docs/agents/`:

| File | Controls |
|------|---------|
| `docs/agents/tracker-adapter.md` | Tracker adapter — implements the tracker contract for GitHub / local markdown / other |
| `docs/agents/build-castle.md` | `review_retry_budget` (integer ≥ 0) |

The tracker contract itself lives at `skills/build-castle/tracker-contract.md` — it defines the verbs (`list-by-status`, `read`, `set-status`, `get-dependency-edges`) every adapter implements, plus the logical statuses `READY` / `IN_PROGRESS` / `COMPLETED` / `FAILED` / `NEEDS_HUMAN`. Concrete label strings live in the adapter's Label map section.

Edit these files directly for minor tweaks. Re-run `/lay-foundation` only to switch issue trackers or reset from scratch.

### review_retry_budget

Integer ≥ 0. How many times `/build-castle` passes review findings back to the TDD agent before escalating an issue to `needs-human`.

- `0` (default) — no retries; first review failure halts the run
- `1` — retry once
- `N` — retry up to N times

## Status vocabulary

Issues flow through five logical statuses: `READY` → `IN_PROGRESS` → `COMPLETED` / `FAILED` / `NEEDS_HUMAN`.

- Logical names + verb signatures: `skills/build-castle/tracker-contract.md`
- Concrete label strings (per repo, per tracker): the **Label map** section of `docs/agents/tracker-adapter.md`
- State machine (events + transitions): the **State machine** section of `skills/build-castle/SKILL.md`

## Skills in this bundle

| Skill | Source |
|-------|--------|
| `lay-foundation` | This repo |
| `build-castle` | This repo |
| `grill-me` | [mattpocock/skills](https://github.com/mattpocock/skills) |
| `to-prd` | [mattpocock/skills](https://github.com/mattpocock/skills) |
| `to-issues` | [mattpocock/skills](https://github.com/mattpocock/skills) |
| `tdd` | [mattpocock/skills](https://github.com/mattpocock/skills) |
