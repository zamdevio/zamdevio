# Phases — maintainer hub

**Not on docs site** — `maintainer/phases/` is contributor-only.

---

## Start here

| Doc | Role |
|-----|------|
| [`active-phase.md`](./active-phase.md) | **Current sprint focus** |
| [`shipped/`](../shipped/README.md) | Closed slices — when present |

Scratch / spikes: **`maintainer/temp/`** (gitignored when adopted in monorepo).

---

## Active work (planned only)

| Doc | Status |
|-----|--------|
| [`auth-concurrency-hardening.md`](./auth-concurrency-hardening.md) | **Next** — login / OIDC race & identity safety |

**Systems maps:** `maintainer/systems/` — add `auth.md` when wiring is documented.

---

## Lifecycle

1. Scope from the **active plan doc** only — one slice per PR.
2. Close slices in `maintainer/shipped/` when done.
3. Session noise → **`maintainer/temp/`** only (not linked from committed docs).
