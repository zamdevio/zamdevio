# Auth concurrency hardening — phase plan

**Status:** **next** — ships before further auth/OIDC feature work.  
**Tracker:** [`active-phase.md`](./active-phase.md)  
**Receipts:** `maintainer/shipped/auth/` (when shipped).

**Goal:** Eliminate cross-flow auth races (password login overlapping Microsoft OIDC) that can corrupt user identity — especially `role` and `isOidc` on existing accounts that were never OIDC-eligible.

---

## Problem

### Reported repro

1. User signs in with **password** as `super_admin` (not OIDC-enabled).
2. While `POST /auth/login` is in flight, user **ctrl+clicks** “Sign in with Microsoft” → new tab starts OIDC.
3. OIDC completes for an **invited MS employee** account (legitimate, separate user in allowlist).
4. Both tabs briefly show `super_admin` (session cookie last-write-wins masks DB corruption).
5. Later: `super_admin` account shows wrong `role` / `isOidc` in DB; admin capabilities lost.

### Root causes (confirmed / highly likely)

| Layer | Issue |
|-------|--------|
| **Frontend** | SSO is a plain `<a href>` inside `Button asChild disabled={loading}` — Radix does not block navigation; ctrl+click opens a second auth flow in a new tab. |
| **Browser** | Tabs share `.cepatedge.com` cookies; two auth writers race on session + OIDC state cookies. |
| **Backend** | No `auth_in_progress` mutex between `POST /auth/login` and `GET /auth/oidc/login` / callback. |
| **Backend** | OIDC callback likely reuses provisioning/link path that sets `role: employee` and `isOidc: true` on the wrong row (email match or session bleed), not only on invited JIT users. |

### Impact

- **Privilege downgrade** — `super_admin` → `employee` in DB.
- **Auth mode flip** — password-only admin becomes SSO-flagged (`isOidc`), blocking password/security flows (`SSO_*_FORBIDDEN` errors).
- **Misleading UX** — winning session cookie hides corruption until next login or admin review.
- **Cross-user risk** — invited employee OIDC flow must not touch unrelated password accounts.

---

## Product decisions (locked for this phase)

| Decision | Choice |
|----------|--------|
| OIDC users remain OIDC-only | **Yes** — no automatic password enablement on link. |
| Auto-link by email alone | **No** — resolve via `oidcSub` + invited allowlist only. |
| Auto-link for `super_admin` / `administrator` | **No** — reject or require explicit admin action (out of scope: build rejection first). |
| Fields mutable on OIDC link | `oidcSub`, `isOidc`, profile name only — **never `role`, `permissions`, `departmentId`**. |
| Concurrent login | **Blocked** — frontend + backend. |

---

## Proposed architecture

```txt
Login page
  ├─ acquireAuthLock()  [sessionStorage + BroadcastChannel]
  ├─ POST /auth/login   [sets auth_attempt cookie]
  └─ SSO button         [blocked while lock held; no <a href> when loading]

GET /auth/oidc/login
  └─ 409 AUTH_IN_PROGRESS if auth_attempt cookie present

GET /auth/oidc/callback
  ├─ resolve user: oidcSub → invited allowlist → existing linked user | JIT create
  ├─ linkOidcToUser()   [immutable role]
  └─ clear auth_attempt
```

---

## Slices (one PR each)

### Slice A — OIDC callback identity safety (backend, P0)

**Primary artifacts**

- `apps/worker/src/routes/auth/oidc/callback.ts` (or equivalent)
- `apps/worker/src/lib/services/auth/oidc.ts` — split `linkOidcToUser` vs `createOidcUser`
- Invited MS user lookup service

**Tasks**

- [ ] Resolve OIDC user **only** by `oidcSub`, then invited-allowlist record — not generic `findByEmail` on arbitrary users.
- [ ] `linkOidcToUser`: update `oidcSub`, `isOidc`, optional name — **assert `role` unchanged**.
- [ ] Reject link when target user is privileged (`super_admin`, `administrator`) or not on allowlist.
- [ ] `createOidcUser`: default `role: employee` **only** on insert.
- [ ] Callback must **not** read existing session or in-flight password-login context.
- [ ] Audit log: `oidc.linked`, `oidc.login`, `oidc.rejected` with `userId`, `oidcSub`, reason.

**Tests**

- [ ] Existing `super_admin` password user + OIDC callback with matching MS email → **reject**, role unchanged.
- [ ] Invited employee first OIDC login → user created/linked, `role: employee`.
- [ ] Returning OIDC user (`oidcSub` match) → login only, no role mutation.

---

### Slice B — Auth-in-progress mutex (backend, P0)

**Primary artifacts**

- `apps/worker/src/routes/auth/login.ts`
- `apps/worker/src/routes/auth/oidc/login.ts`
- Shared helper: `authAttemptCookie` (name TBD)

**Tasks**

- [ ] On `POST /auth/login` start: set short-lived HttpOnly cookie `auth_attempt` (e.g. 30–60s, `Path=/`, `SameSite=Lax`).
- [ ] On `GET /auth/oidc/login`: if `auth_attempt` present → `409` with code `AUTH_IN_PROGRESS`.
- [ ] Clear `auth_attempt` on success/failure of either flow.
- [ ] OIDC callback clears cookie on completion.

**Tests**

- [ ] OIDC login start while password login cookie active → 409.
- [ ] Cookie expires → OIDC allowed again.

---

### Slice C — Login page mutual exclusion (frontend, P0)

**Primary artifacts**

- `apps/web` login page (e.g. `LoginPage` / auth domain)
- Optional: `apps/web/src/lib/auth-lock.ts`

**Tasks**

- [ ] Replace SSO `<a href>` with controlled `onClick` → `window.location.href` only when lock acquired.
- [ ] `acquireAuthLock` / `releaseAuthLock` via `sessionStorage` + `BroadcastChannel('cepatedge-auth')`.
- [ ] Disable form + SSO while `loading` or lock held in another tab.
- [ ] Full-screen overlay (`pointer-events: all`) during password login — blocks ctrl+click on underlying links.
- [ ] Release lock on login error; SSO navigation keeps lock until redirect (or use short TTL).

**Tests**

- [ ] E2E or component: SSO button inert while `loading === true`.
- [ ] BroadcastChannel: tab B disables SSO when tab A starts login.

---

### Slice D — Recovery runbook + admin visibility (docs / small UI, P1)

**Tasks**

- [ ] Document DB recovery steps for corrupted admin (`role`, `isOidc`, `oidcSub`).
- [ ] User detail UI: clearly show auth mode (`password`, `oidc`, `hybrid` if ever added).
- [ ] Optional: admin audit filter for `oidc.*` events.

---

## Acceptance criteria (phase complete)

| # | Criterion |
|---|-----------|
| 1 | Repro from bug report (password login + ctrl+click SSO new tab) does **not** mutate `super_admin` row. |
| 2 | Backend returns `409 AUTH_IN_PROGRESS` when second flow starts during first. |
| 3 | OIDC callback never changes `role` on update/link. |
| 4 | Invited employee OIDC login still works end-to-end. |
| 5 | Frontend blocks SSO during password login (including new tab via BroadcastChannel). |
| 6 | Automated tests cover link rejection for privileged password users. |

---

## Out of scope (later phases)

- Admin-approved OIDC linking UI for privileged accounts.
- Break-glass password for SSO users.
- Full `maintainer/systems/auth.md` (create during Slice A/B, finalize at phase close).
- Session invalidation / “single active login” product policy.

---

## Verification checklist (pre-merge)

```bash
# Worker — unit/integration
pnpm --filter worker test -- auth/oidc
pnpm --filter worker test -- auth/login

# Web — component / e2e if present
pnpm --filter web test -- login

# Manual smoke
# 1. Password login only → super_admin dashboard
# 2. Invited MS user SSO only → employee dashboard
# 3. Password login + attempt SSO (same + new tab) → blocked, DB unchanged
```

---

## References

- Production login chunk: SSO via `` `${API_BASE}/auth/oidc/login` ``, `disabled={loading}` on `Button asChild` + `<a>`.
- API: `POST /auth/login`, `GET /auth/oidc/login` (OpenAPI); callback undocumented — primary fix target.
- Error codes: `SSO_PASSWORD_CHANGE_FORBIDDEN`, `SSO_EMAIL_CHANGE_FORBIDDEN`, etc. — confirm `isOidc` semantics after fix.
