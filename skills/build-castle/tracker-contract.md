# Tracker contract

Verb set every tracker adapter must implement. Callers (`/build-castle` and any future tracker-aware skill in this bundle) depend only on this contract.

A concrete tracker adapter lives at `docs/agents/tracker-adapter.md` in the target repo, written by `/lay-foundation` from a template in `skills/lay-foundation/tracker-adapter-<name>.md`.

## Logical statuses

Callers pass logical names. Adapters translate to/from concrete label strings in their own "Label map" section.

| Logical | Default label string |
|---------|----------------------|
| `READY` | `ready-for-agent` |
| `IN_PROGRESS` | `in-progress` |
| `COMPLETED` | `completed` |
| `FAILED` | `failed` |
| `NEEDS_HUMAN` | `needs-human` |

## Issue shape

```
{ number, title, body, status }
```

`number` = string identifier (zero-padded for local, integer-as-string for github). `status` = one logical name from the table above. Body = full markdown body, exactly as stored.

## Verbs

Each adapter exposes one `##` section per verb, in this order, with this exact heading text.

### list-by-status

- **Input**: `statuses[]` — list of logical names
- **Output**: list of `{ number, title, body, status }` for every open issue whose status is in the input set
- Body included; caller does not need a separate `read`

### read

- **Input**: `number`
- **Output**: one `{ number, title, body, status }`

### set-status

- **Input**: `number`, `from` (logical), `to` (logical)
- **Behaviour**: atomic transition. Apply the `to` label, remove the `from` label.
- **Error mode**: if the issue's current status ≠ `from`, halt with `"expected #N to be <from>, found <actual>"`. Caller (build-castle) stops execution. Guards against concurrent edits.

### get-dependency-edges

- **Output**: list of `(blocked, blocker)` pairs, where both are issue `number`s
- Includes edges from all open + recently-closed issues — caller needs the full graph to detect blocked-by-failed
- Convention is adapter-specific; see each adapter's section

## What is not in the contract

- **Creating issues.** `/to-prd` and `/to-issues` (mattpocock-owned) publish via informal prose in the adapter file. They do not enforce the contract.
- **Closing issues.** Status is tracked by label; `COMPLETED` does not close the underlying ticket.
- **Reading comments.** Not needed by `/build-castle`.
