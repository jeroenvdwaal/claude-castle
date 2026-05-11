# Tracker adapter: Local markdown

Implements the tracker contract (`skills/build-castle/tracker-contract.md`) for issues stored as `.md` files under `issues/` at repo root.

## File convention

`issues/<NNN>-<slug>.md`. `<NNN>` zero-padded to 3 digits. Slug lowercase-hyphenated.

Frontmatter on every issue:

```markdown
---
title: "Short issue title"
type: AFK | HITL
status: ready-for-agent | in-progress | completed | failed | needs-human
blocked-by: none | "<NNN>" | "<NNN>,<NNN>"
---
```

`status` value is a concrete label string; map via Label map below.

## Label map

| Logical | Label (= `status:` value) |
|---------|---------------------------|
| `READY` | `ready-for-agent` |
| `IN_PROGRESS` | `in-progress` |
| `COMPLETED` | `completed` |
| `FAILED` | `failed` |
| `NEEDS_HUMAN` | `needs-human` |

## list-by-status

- **Input**: `statuses[]` (logical names)
- **Output**: list of `{ number, title, body, status }`

Translate each logical name to its label string. Then:

```sh
grep -l -E "^status: (<label1>|<label2>|...)$" issues/*.md
```

For each match, read the file. `number` = the `NNN` from the filename. `title` from frontmatter. `body` = file contents after the closing `---`. `status` = the frontmatter `status:` value mapped back through the Label map.

## read

- **Input**: `number`
- **Output**: `{ number, title, body, status }`

```sh
cat issues/<NNN>-*.md
```

Parse frontmatter for `title` and `status` (map via Label map). `body` = everything after the closing `---`.

## set-status

- **Input**: `number`, `from`, `to`
- **Behaviour**: atomic transition

Read the file's `status:` frontmatter line. If it does not equal the `from` label, halt with `"expected #<number> to be <from>, found <actual>"`.

Otherwise rewrite the `status:` line in place to the `to` label.

## get-dependency-edges

- **Output**: list of `(blocked, blocker)` pairs

Convention: the `blocked-by:` frontmatter field. Values:

- `none` — no edges
- `"NNN"` — one edge
- `"NNN,NNN,..."` — comma-separated for multiple

For each file under `issues/*.md`, parse `blocked-by:`. For every non-`none` entry, emit `(<this file's NNN>, <blocker NNN>)`.

## Informal: creating issues

Used by `/to-prd` and `/to-issues` (not part of the enforced contract):

Write a new file at `issues/<next-NNN>-<slug>.md` with full frontmatter (default `status: ready-for-agent`).
