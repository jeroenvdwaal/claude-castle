---
name: lay-foundation
description: Configures a repo for the Castle (claude-castle) workflow. Sets up the tracker adapter (implements the tracker contract), review_retry_budget, and label-map overrides in docs/agents/. Run before first use of /build-castle, or if /build-castle appears to be missing context about the tracker adapter or label map.
disable-model-invocation: true
---

# Lay Foundation

Configure this repo for the Castle (claude-castle) development pipeline. Before any sand can be packed, the foundation has to be flat.

This skill writes:

- **Tracker adapter** — implements the tracker contract for this repo's issue tracker
- **`review_retry_budget`** — how many automatic TDD retries `/build-castle` attempts when `/review` finds problems before escalating to a human
- **Label strings** — concrete labels mapped from logical statuses, written into the adapter's Label map

This is a prompt-driven skill. Explore current state, present findings, confirm with user one decision at a time, then write.

## Process

### 1. Explore

Read current repo state — don't assume:

- `git remote -v` — GitHub remote present? (suggests GitHub issue tracker)
- `CLAUDE.md` and `AGENTS.md` at root — which exists? Is there already an `## Agent skills` section?
- `docs/agents/tracker-adapter.md` — prior tracker adapter config?
- `docs/agents/build-castle.md` — prior build-castle config?
- `issues/` directory at root — local markdown issue tracker in use?
- `.scratch/` — alternative local markdown convention

### 2. Present findings and ask

Summarise what's present and what's missing. Walk user through three decisions **one at a time** — present each, get answer, then move to next.

**Decision A — Tracker adapter**

> The tracker adapter tells `/build-castle` how to talk to your issue tracker. It implements the tracker contract (`skills/build-castle/tracker-contract.md`) — a fixed set of verbs (`list-by-status`, `read`, `set-status`, `get-dependency-edges`) — using commands appropriate for the chosen tracker. Pick where you actually track work.

- **GitHub** — issues in GitHub Issues (uses `gh` CLI). Recommended if `git remote` points to GitHub.
- **Local markdown** — issues as `.md` files under `issues/` at repo root. Good for solo projects or repos without a remote.
- **Other** (Jira, Linear, etc.) — describe your workflow in a paragraph; skill records it as freeform prose.

Default: GitHub if remote points to GitHub, local markdown otherwise.

**Decision B — `review_retry_budget`**

> When `/build-castle` runs `/review` on completed TDD work and review finds problems, how many times should it pass the findings back to the TDD agent and retry before escalating to a human?

Integer ≥ 0. Default: `0`.

- `0` — no retries; first review failure marks the issue `needs-human` and halts the run
- `1` — retry once; if review still fails, escalate
- `N` — retry up to N times before escalating

**Decision C — Label strings**

> `/build-castle` works with logical statuses (`READY`, `IN_PROGRESS`, `COMPLETED`, `FAILED`, `NEEDS_HUMAN`). The tracker adapter maps each logical status to a concrete label string in its "Label map" section. These strings must match labels that exist (or will be created) in your issue tracker.

Defaults (override any that conflict with existing labels):

| Logical | Default label |
|---------|---------------|
| `READY` | `ready-for-agent` |
| `IN_PROGRESS` | `in-progress` |
| `COMPLETED` | `completed` |
| `FAILED` | `failed` |
| `NEEDS_HUMAN` | `needs-human` |

Ask if user wants to override any. If issue tracker has no existing labels, defaults are fine. Overrides land in the adapter's Label map section.

### 3. Confirm and edit

Show user a draft of:

- The `## Agent skills` block to add to CLAUDE.md / AGENTS.md
- Contents of `docs/agents/tracker-adapter.md`
- Contents of `docs/agents/build-castle.md`

Let them review and request edits before writing.

### 4. Write

**Pick file to edit:**

- If `CLAUDE.md` exists → edit it
- Else if `AGENTS.md` exists → edit it
- If neither → ask user which to create; don't pick for them

Never create `AGENTS.md` when `CLAUDE.md` already exists (or vice versa). If `## Agent skills` block already exists in the chosen file, update in-place — don't append a duplicate.

The block to add/update:

```markdown
## Agent skills

### Tracker adapter

[one-line summary of tracker]. See `docs/agents/tracker-adapter.md`. Contract: `skills/build-castle/tracker-contract.md`.

### Build-castle

review_retry_budget: [N]. See `docs/agents/build-castle.md`.
```

Write config files using the seed templates in this skill folder:

- [tracker-adapter-github.md](./tracker-adapter-github.md) — GitHub Issues via `gh`
- [tracker-adapter-local.md](./tracker-adapter-local.md) — local markdown under `issues/`
- [build-castle.md](./build-castle.md) — review_retry_budget config

Copy the chosen template to `docs/agents/tracker-adapter.md` in the target repo. If the user overrode any defaults in Decision C, edit the "Label map" section of the copied adapter accordingly.

For "other" trackers, write `docs/agents/tracker-adapter.md` from scratch. Use one of the existing templates as a structural model — same `##` headings (`list-by-status`, `read`, `set-status`, `get-dependency-edges`, `Label map`) — and fill each section with the user's description.

**If `docs/agents/` files already exist:** update only fields that changed. Don't overwrite sections the user has customised.

### 5. Done

Tell user setup is complete. Mention:

- They can edit `docs/agents/*.md` directly for minor tweaks — no need to re-run this skill
- Re-run only if switching issue trackers or resetting from scratch
- Full pipeline: `/grill-me` → `/to-prd` → `/to-issues` → `/build-castle`
- Dependencies from `mattpocock/skills`: `grill-me`, `to-prd`, `to-issues`, `tdd`
- From this package: `lay-foundation`, `build-castle`
