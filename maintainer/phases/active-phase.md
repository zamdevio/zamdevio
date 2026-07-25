# Active sprint

**Shipped receipts:** [`shipped/`](../shipped/README.md) (when present)

---

## Focus now

| Priority | Vertical | Doc |
|----------|----------|-----|
| **Next** | Auth — login / OIDC concurrency | [`auth-concurrency-hardening.md`](./auth-concurrency-hardening.md) |

Engineering maps: `maintainer/systems/` (add `auth.md` during implementation).

---

## Dependency chain

```txt
Auth concurrency hardening (frontend lock + backend enforcement + OIDC callback fix)
  → auth system doc (`maintainer/systems/auth.md`)
  → regression tests + shipped receipt
```

---

## Guiding rules

- **Backend enforcement is mandatory** — UI-only guards are insufficient (ctrl+click / new tab bypass).
- **One concern per PR** — follow slices in the active plan doc.
- **Never mutate `role` on OIDC link** — create vs link must be separate code paths.
- **Invited MS allowlist only** — OIDC callback must not reach password-only or privileged users by email alone.
