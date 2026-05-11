# Build-castle config

Per-repo configuration for the `/build-castle` skill. Copied to `docs/agents/build-castle.md` by `/lay-foundation`.

## review_retry_budget

Integer ≥ 0. How many times `/build-castle` passes review findings back to the TDD agent before escalating an issue to `NEEDS_HUMAN`.

- `0` *(default)* — no retries; first review failure halts the run
- `1` — retry once
- `N` — retry up to N times

Current setting: `0`

---

Status vocabulary and transitions live elsewhere:

- Logical statuses (`READY`, `IN_PROGRESS`, …): `skills/build-castle/tracker-contract.md`
- Concrete label strings per tracker: the Label map in `docs/agents/tracker-adapter.md`
- State machine (events + transitions, including the retry-budget rows): the "State machine" section of `skills/build-castle/SKILL.md`
