# Tracker adapter: GitHub

Implements the tracker contract (`skills/build-castle/tracker-contract.md`) for GitHub Issues. All operations use the `gh` CLI; repo is inferred from `git remote -v`.

## Label map

Logical → concrete label string. Edit the right column to match labels in your GitHub repo.

| Logical | Label |
|---------|-------|
| `READY` | `ready-for-agent` |
| `IN_PROGRESS` | `in-progress` |
| `COMPLETED` | `completed` |
| `FAILED` | `failed` |
| `NEEDS_HUMAN` | `needs-human` |

## list-by-status

- **Input**: `statuses[]` (logical names)
- **Output**: list of `{ number, title, body, status }`

Translate each logical name to its label string via the Label map, then:

```sh
gh issue list \
  --state open \
  --label "<label1>" --label "<label2>" ... \
  --json number,title,body,labels
```

`gh issue list` ANDs labels by default. For OR semantics (the contract requires OR across the input set), call once per label and merge by `number`, or use `--search "label:\"<l1>\" label:\"<l2>\""` with quoted label terms.

Map each result's `.labels[].name` back through the Label map (first match wins) to populate `status`. Drop issues whose labels contain none of the input statuses.

## read

- **Input**: `number`
- **Output**: `{ number, title, body, status }`

```sh
gh issue view <number> --json number,title,body,labels
```

Same Label-map translation for `status`.

## set-status

- **Input**: `number`, `from`, `to`
- **Behaviour**: atomic transition

First verify current status. Fetch labels:

```sh
gh issue view <number> --json labels
```

If `from`'s label is not present, halt with `"expected #<number> to be <from>, found <actual>"`. Otherwise:

```sh
gh issue edit <number> --add-label "<to-label>" --remove-label "<from-label>"
```

## get-dependency-edges

- **Output**: list of `(blocked, blocker)` pairs

Convention: in the issue body, a dependency is declared by a line matching the regex:

```
^(?:Blocked by|Depends on) #(\d+)\s*$
```

Multiple blockers = multiple lines. Case-sensitive on the leading word; numbers are GitHub issue numbers.

Fetch all open + recently-closed issues:

```sh
gh issue list --state all --limit 500 --json number,body
```

For each issue, scan body lines against the regex; emit `(issue.number, capture)` for each match.

## Informal: creating issues

Used by `/to-prd` and `/to-issues` (not part of the enforced contract):

```sh
gh issue create --title "..." --label "ready-for-agent" --body "$(cat <<'EOF'
...
EOF
)"
```
