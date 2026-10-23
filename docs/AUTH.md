# Authentication & Authorization — Budget 2.0 (V1)

| Field | Value |
|-------|-------|
| **Status** | **Draft / Proposed** — awaiting owner acceptance as the V1 auth baseline |
| **Source issue** | [#3 — Decide on features using user stories](https://github.com/zachkingcade/ZK-Dynamic-Flow-Budgeting-Application/issues/3) |
| **Companion** | Features: [`FEATURES.md`](./FEATURES.md) (`F01`–`F03`). UI: [`USER_INTERFACE.md`](./USER_INTERFACE.md) (`UI-01`, `UI-02`). Architecture: [`ARCHITECTURE.md`](./ARCHITECTURE.md) (public Compose host, modular monolith, tenancy). Stories: [`USER_STORIES.md`](./USER_STORIES.md) (1–3, P8). Domain: [`BUSINESS_RULES.md`](./BUSINESS_RULES.md) (**Accepted**; ≤1 user per account; not an auth protocol). |
| **Scope** | How users prove identity (local + Authentik), how sessions work for the Spring Boot API + Angular SPA, and how ownership authorization applies after login. Not IAM admin UI, roles matrix beyond owner, or multi-user budget sharing. |
| **Security owner** | Security Developer (this document) |

### Status legend

| Label | Meaning |
|-------|---------|
| **Draft / Proposed** | Team security recommendation for Issue #3; not binding until owner accepts |
| **Accepted** | Owner-approved V1 auth baseline; implementations must obey |

---

## 1. Purpose and relationship to other docs

### Purpose

Define a **basic but secure** authentication model for Budget 2.0 so that:

1. Users can **register and sign in with local credentials** (`F03`, story 3, `UI-01` / `UI-02`).
2. Users can **sign in with Authentik** via OIDC (`F01`, story 2), with **auto-provision** of an app account on first successful login (`F02`, story 1).
3. Local and Authentik identities are **separate accounts** — **no account merging or linking** in V1 (owner requirement).
4. After authentication, every financial read/write is authorized by **owner identity + `budget_id`** (`ARCHITECTURE.md` §7.2). Auth proves *who*; use-cases enforce *what they may touch*.

### Non-goals of this document

- Full IAM admin console, user directory sync beyond first-login provision, or Authentik realm administration.
- Account linking / merging when the same person uses both local and Authentik.
- Multi-user collaboration on one budget (product non-goal; ≤1 user per account).
- Temporary pre-auth public-API exposure policy (Architecture O7) — still a deploy gate until this auth ships; this doc assumes auth is the permanent control.
- Password-reset email product design beyond a secure minimum placeholder (see open questions).

### Feature / UI mapping

| Capability | Feature | UI | This doc |
|------------|---------|-----|----------|
| Authentik SSO access | `F01` | `UI-01` | Part B |
| Authentik auto-provision | `F02` | `UI-01` → `UI-02` if setup incomplete | Part B |
| Local registration + login | `F03` | `UI-01`, `UI-02` | Part A |
| Timezone + primary budget on account creation | `F05`, `F04` | `UI-02` | Post-auth account setup (not protocol) |

---

## 2. Threat model and security goals (public host)

Deployment baseline from `ARCHITECTURE.md` is unchanged and **must not be weakened**:

- Only `web` (nginx/Caddy) publishes 80/443; `api` and `db` stay on the internal Compose network.
- TLS on the public path before credentials exist.
- Secrets via env / Compose secrets — never baked into images or committed.
- Angular artifacts receive **public** config only (API base URL, OIDC **public** client id if any). No private keys, client secrets, or DB URLs in the SPA build.
- CORS / origin locked to the SPA origin.
- Do **not** ship an open financial API; auth is the permanent product gate for mutating and financial-read calls.

### Assets

| Asset | Why it matters |
|-------|----------------|
| Account credentials / session | Unauthorized access to someone’s budgets |
| Budget balances, journals, funding rules, CSV-imported transactions | Confidential financial data |
| Ownership links (`owner_user_id`, `budget_id`) | Integrity against IDOR / cross-tenant access |
| OIDC client secret, DB password, session signing material | Compromise ⇒ full app takeover |

### Primary threats (V1)

| ID | Threat | Severity if unmitigated |
|----|--------|-------------------------|
| T1 | Credential stuffing / brute force on local login | High |
| T2 | Password theft (plaintext, weak hash, log leakage) | Critical |
| T3 | Session theft / XSS exfiltration of tokens | Critical / High |
| T4 | CSRF on cookie-authenticated state-changing APIs | High |
| T5 | OIDC redirect / callback abuse (open redirect, code interception) | High |
| T6 | Auto-provision creating wrong or colliding accounts | High |
| T7 | IDOR: caller authenticated but accesses another user’s `budget_id` | Critical |
| T8 | Secrets in SPA or git | Critical |
| T9 | Downgrade: HTTP, insecure cookies, disabled CSRF “for SPA convenience” | High |

### Security goals

1. **Confidentiality** — only the authenticated owner reaches their budgets’ financial data.
2. **Integrity** — login/provision cannot be forged; sessions cannot be easily stolen or fixed.
3. **Availability (auth path)** — basic rate limiting / lockout so public host login is not a free oracle.
4. **Least privilege** — Authentik grants *app login*, not elevated IAM inside Budget 2.0; every user is an ordinary account owner of their own budgets.
5. **Fail closed** — unauthenticated → 401; authenticated but not owner → 403 / not found (no cross-tenant leakage).

---

## 3. Recommended stack pattern (normative direction)

**Fit:** modular monolith (one Spring Boot API) + Angular SPA served and reverse-proxied by nginx on the **same public origin**.

### Recommended: server-side session + same-origin cookies (BFF-style)

| Piece | Choice |
|-------|--------|
| Framework | **Spring Security** on the API |
| Local login | Form / JSON login against app-stored credentials → establish **HTTP session** |
| Authentik | **OAuth2 Login** (OIDC authorization code) via Spring Security OAuth2 Client → on success, establish the **same app session** |
| Browser credential | **Session cookie**: `HttpOnly`, `Secure`, `SameSite=Lax` (or `Strict` if UX allows), `Path=/` appropriate to API |
| CSRF | **Required** for cookie sessions — e.g. Spring `CookieCsrfTokenRepository` + Angular reads CSRF cookie / header on mutating requests |
| SPA storage | **Do not** store access tokens or refresh tokens in `localStorage` / `sessionStorage` |
| nginx | Terminate TLS; proxy `/api/*` to `api:8080`; forward only needed headers; do not expose `api` publicly |

**Rationale:** First-party SPA on one origin is the simplest secure default. Cookies keep secrets out of JS; CSRF is well-supported; both local and Authentik paths converge on one app principal and one session model. Domain modules stay free of Spring Security types; the security filter / identity adapter resolves `AuthenticatedUser` for use-cases.

### Acceptable alternative (owner may choose): OIDC JWT resource server + local JWT

- Authentik: SPA or API performs OIDC; API validates JWTs as **OAuth2 Resource Server**.
- Local: API issues its **own** short-lived JWTs (and optionally refresh) after password verify — **not** “homemade crypto”; use a maintained JWT library and keys from env.
- Prefer **Authorization Code + PKCE** for any browser talk to Authentik; never embed client secrets in Angular.
- Still prefer **not** putting long-lived tokens in `localStorage` (XSS). If cookies are used for JWTs, CSRF rules still apply.

**Default for V1 implementation planning: session-cookie model above** unless the owner selects the JWT alternative in open questions.

### Spring placement (Clean Architecture)

```text
bootstrap/          → SecurityFilterChain, OAuth2 client registration, session/CSRF config
identity/adapter/inbound/web/  → register, login, logout, OIDC callback wiring, /me
identity/application/          → RegisterLocalUser, CompleteAccountSetup, ResolveOrProvisionOidcUser
identity/domain/               → Account, User, IdentityProvider, credential hash VO (no framework types)
identity/adapter/outbound/persistence/ → users, credentials, oidc_identities tables (Flyway)
```

Money modules **never** trust a client-supplied `owner_user_id`. They receive the authenticated principal from the application edge and load budgets by `(owner, budget_id)`.

---

## 4. Part A — Local / custom authentication (`F03`)

Basic registration and password login, secured to a public-host baseline.

### A.1 Registration (`UI-02` local path)

**Minimum fields (product + security):**

| Field | Rule |
|-------|------|
| Identifier | Email (recommended) **or** username — pick one primary login id (see open questions). Normalize email (trim + lowercase) before store/compare. |
| Password | Meets policy below; never logged; never returned in API responses |
| Timezone | IANA; default **UTC** (`F05`) — product field, collected at setup |

**On success (same unit of work as account creation where practical):**

1. Create **Account** + **User** (1:1) with `identity_provider = LOCAL`.
2. Store **password hash only** (see A.3).
3. Create **primary budget** (`F04`).
4. Establish authenticated session (or require explicit login — prefer auto-login after register if CSRF/session setup is correct).
5. Land in budget workspace (`USER_INTERFACE.md` first-run local flow).

**Reject:** duplicate local identifier; weak password; invalid timezone. Generic errors on login; clearer errors on register only where they do not enable user enumeration beyond necessity (see A.5).

### A.2 Login (`UI-01` local)

1. Client posts identifier + password to API over HTTPS only.
2. Server looks up **LOCAL** credential row; verifies hash with constant-time comparison via the password library.
3. On success: rotate/create session; bind session to `user_id` / `account_id`.
4. On failure: generic `"Invalid credentials"` (same message for unknown user and bad password).
5. Apply rate limiting / lockout (A.5).

### A.3 Password storage

| Requirement | Detail |
|-------------|--------|
| Algorithm | **Argon2id** preferred (Spring Security `Argon2PasswordEncoder`); **bcrypt** acceptable if Argon2 not yet available in chosen Spring version |
| Parameters | Use Spring defaults or documented OWASP-aligned params; store params with the hash as the encoder requires |
| At rest | Hash (+ salt/params embedded in encoded string) only — **never** plaintext, reversible encryption, or “encrypted password” as a substitute for hashing |
| Verification | Only via the password encoder; no manual string compare of digests |

### A.4 Password policy (minimum V1)

Enforce **server-side** (client checks are UX only):

| Rule | Minimum |
|------|---------|
| Length | ≥ **12** characters |
| Composition | Reject empty / whitespace-only; recommend (not necessarily require) mix of character classes — length matters more than complexity theater |
| Blocklist | Reject obvious passwords (`password`, `123456789012`, site name, etc.) if a small denylist is easy; optional HaveIBeenPwned k-anonymity later |
| Storage on change | Re-hash with current encoder |

No SMS 2FA / WebAuthn required for V1 baseline (optional hardening later).

### A.5 Rate limiting and lockout (basics)

Public host ⇒ assume credential stuffing.

| Control | V1 expectation |
|---------|----------------|
| Per-IP rate limit | On `/api/.../login` and register (and OIDC start if applicable) — e.g. nginx `limit_req` and/or Spring/Bucket4j filter |
| Per-account lockout / backoff | After N consecutive failures (e.g. 5–10), delay or temporary lock (e.g. 15 minutes); unlock via time expiry in V1 |
| Logging | Security events: failed login, lockout, register — **without** passwords; include request id / IP carefully (PII policy) |
| Enumeration | Login responses stay generic; registration may say “email taken” (acceptable tradeoff) or use generic messaging (owner choice) |

Exact numeric thresholds may be tuned in implementation; the **presence** of rate limit + backoff is required.

### A.6 Session lifecycle and logout

| Topic | Rule |
|-------|------|
| Session idle / absolute timeout | Set finite timeouts (e.g. idle 30m–8h, absolute ≤ 24h — owner may tune); document chosen values in implementation |
| Cookie flags | `HttpOnly` + `Secure` + `SameSite`; no `Secure=false` on public deploy |
| Logout | Server invalidates session; clear cookie; SPA clears client state and routes to `UI-01` |
| Concurrent sessions | V1: allow multiple sessions unless owner wants single-session; not a blocker |

### A.7 HTTPS and transport

- Credentials and session cookies **only** over TLS.
- HSTS on `web` once TLS is stable.
- No mixed-content API calls from the SPA.

### A.8 CSRF (cookie sessions)

If the recommended cookie session model is used:

1. Enable Spring CSRF for state-changing requests (`POST`/`PUT`/`PATCH`/`DELETE`).
2. Expose a CSRF cookie the Angular app can read (non-HttpOnly CSRF cookie pattern) and send as header (e.g. `X-XSRF-TOKEN`).
3. Do **not** disable CSRF because “it’s an API” while using cookie auth.

If the JWT Bearer-only alternative is chosen and **no** cookies authenticate the API, CSRF is not applicable to Bearer header auth — but XSS token theft becomes the dominant risk; mitigate accordingly.

### A.9 What NOT to do (local auth)

| Forbidden | Why |
|-----------|-----|
| Store plaintext or reversible “encrypted” passwords | Credential theft = instant account compromise |
| Homemade hashing / rolling your own crypto | Broken under real attacks |
| Put session tokens or JWTs in `localStorage` by default | XSS ⇒ full account takeover |
| Disable TLS, `Secure` cookies, or CSRF “temporarily” on the public host | Violates architecture baseline |
| Trust client-supplied `userId` / `ownerId` as authorization | IDOR (Part D) |
| Log passwords, Authorization headers, or full session ids | Secret leakage |
| Ship default admin/password in Compose | Trivial compromise |
| “Security through obscurity” open API without auth | Explicitly rejected by `ARCHITECTURE.md` |

### A.10 Password reset (V1 stub)

V1 may ship **without** polished email reset if mail is not ready — but then document the ops path (manual DB/admin intervention is **not** a product feature). Prefer a follow-on issue for signed, single-use, time-limited reset tokens over email. Do not implement “reset by knowing email only” without a secret token.

---

## 5. Part B — Authentik integration (`F01`, `F02`)

Authentik is an **external IdP**. Budget 2.0 trusts Authentik for authentication of Authentik-origin users only; it does **not** merge them with local users.

### B.1 Protocol

| Item | Recommendation |
|------|----------------|
| Protocol | **OIDC** (OpenID Connect) over OAuth 2.0 |
| Flow | **Authorization Code** |
| Spring support | **Spring Security OAuth2 Client** (`oauth2Login`) for the session-cookie model; or Resource Server JWT validation if JWT alternative is chosen |
| Client type | Confidential client on the **API** (client secret in env) for server-side code exchange — preferred with session model. If SPA talks to Authentik directly, use **public client + PKCE** and never ship a client secret in Angular. |

### B.2 Redirect / callback (session model)

Illustrative flow (same-origin via nginx):

```text
Browser                web (nginx)              api (Spring)              Authentik
   |  UI-01 "Sign in with Authentik"                 |                        |
   |------------------------------------------------>|                        |
   |                 302 to Authentik authorize      |                        |
   |---------------------------------------------------------------------->   |
   |                 authenticate at Authentik                                |
   |                 302 redirect to callback                                 |
   |------------------------------------------------>|<-- authorization code--|
   |                                    code → token exchange (server-side)   |
   |                                    validate ID token / userinfo          |
   |                                    provision or load LOCAL app user      |
   |                                    create app session + Set-Cookie       |
   |  302 → /account/setup or /b/{primary}/accounts  |                        |
```

**Rules:**

- Register **exact** redirect URI(s) in Authentik (e.g. `https://<public-host>/api/login/oauth2/code/authentik` or the Spring-configured callback). No wildcard open redirects.
- `state` (and PKCE where applicable) mandatory; reject callbacks with invalid/missing state.
- After success, issue **app session** (recommended model); do not permanently expose Authentik access tokens to the SPA.
- On SSO failure: return user to `UI-01` with a generic SSO error (log detail server-side).

### B.3 Claims mapping

| Claim | Use |
|-------|-----|
| **`iss`** (issuer) | Part of identity key; must match configured Authentik issuer |
| **`sub`** | **Stable subject** — primary external identity key within that issuer |
| **`email`** (if present) | Profile display / contact; **not** a merge key |
| `email_verified` | Prefer requiring verified email if Authentik provides it; if absent, still provision but do not treat email as proof across providers |
| `name` / `preferred_username` | Optional display fields |

**Identity key for Authentik users:** `(identity_provider = AUTHENTIK, issuer, subject)` — unique in the database.

**Do not** look up an existing **LOCAL** user by email and attach the OIDC identity. That would be linking/merging — **out of scope / forbidden in V1**.

### B.4 Auto-provision (`F02`)

On first successful Authentik login when no row exists for `(AUTHENTIK, issuer, sub)`:

1. Create **Account** + **User** with `identity_provider = AUTHENTIK`.
2. Persist `issuer`, `sub`, and optional email/display claims on an `oidc_identity` (or equivalent) record.
3. Create **primary budget** (`F04`).
4. If timezone not yet set → send user through `UI-02` for timezone (and any required profile fields only — no password). If product later defaults timezone to UTC without confirmation, that is a UX decision (`USER_INTERFACE.md` Q1); security only requires that an account + ownership exist before money APIs are used.
5. Subsequent logins: resolve by `(issuer, sub)` only; update mutable profile claims if desired; **never** re-bind to a different local account.

**Concurrency:** unique DB constraint on `(provider, issuer, sub)` so two parallel first-logins cannot create duplicates; handle conflict by loading the winner row.

### B.5 Compose / env secrets (placeholders)

Names illustrative — align with `ARCHITECTURE.md` placeholders; real values only in gitignored `.env` / secrets:

```text
# Public (may appear in SPA env for alternative PKCE model — prefer server-only client)
OIDC_ISSUER_URI=https://authentik.example.com/application/o/budget/
OIDC_CLIENT_ID=budget-app

# Server-only (API container)
OIDC_CLIENT_SECRET=          # Compose secret / env — never in Angular
OIDC_REDIRECT_URI=https://budget.example.com/api/login/oauth2/code/authentik

# Session / cookie (API)
SERVER_SERVLET_SESSION_COOKIE_SECURE=true
# CSRF, session timeout, etc. via Spring config

# Existing baseline
POSTGRES_PASSWORD=
# ... other app secrets
```

Angular may know: public site URL, “SSO enabled” flag, and **public** client id **only if** PKCE SPA model is chosen. Prefer keeping client secret and token exchange on the API.

Authentik side (ops, not app code): create OIDC provider/application, map redirect URI, decide which groups/users may access the app (Authentik access policy). Budget 2.0 does not implement Authentik admin UI.

### B.6 Logout / session notes with IdP

| Layer | Behavior |
|-------|----------|
| **App logout** | Invalidate Spring session; clear cookies; SPA → `UI-01`. Required. |
| **Authentik (RP-initiated) logout** | **Optional V1** — if enabled, redirect to Authentik end-session endpoint after app logout so the IdP SSO cookie is cleared. If skipped, user may one-click SSO back in (acceptable if documented). |
| **IdP session vs app session** | Independent. Expiry of one does not magically authorize the other. |
| **Token revocation** | If using JWT resource-server model, prefer short-lived access tokens; session model relies on server-side invalidation. |

### B.7 What NOT to do (Authentik path)

| Forbidden | Why |
|-----------|-----|
| Client secret in Angular / git | Trivial IdP impersonation |
| Account merge by email with LOCAL users | Violates owner “no linking” rule; account takeover risk |
| Accepting ID tokens without validating issuer, audience, signature, expiry | Forged identity |
| Open redirect after login (`?returnTo=` unchecked) | Phishing / token theft aid |
| Treating Authentik group claims as budget sharing | Multi-user budgets are a non-goal |

---

## 6. Part C — Coexistence (local + Authentik)

Both paths ship in one application.

### C.1 Identity provider on the account

Each account/user records how it authenticates:

| `identity_provider` | Credentials | Login path |
|---------------------|-------------|------------|
| `LOCAL` | Password hash | `UI-01` local form → Part A |
| `AUTHENTIK` | `(issuer, sub)` via OIDC | `UI-01` SSO → Part B |

Optional future providers would add new enum values — not in V1.

### C.2 Separate users (no merge)

| Rule | Detail |
|------|--------|
| Separate rows | A person who registers locally **and** signs in with Authentik gets **two** app accounts (two primary budgets, two data sets) |
| No linking UI | No “connect Authentik to this account”, no automatic email match |
| Login routing | Local form only verifies `LOCAL` users; OIDC only resolves `AUTHENTIK` identities |
| Authorization after login | Identical: authenticated `user_id` may access only budgets they own |

### C.3 Email uniqueness (policy to confirm)

**Security recommendation:** uniqueness of login email **within `LOCAL`**, and uniqueness of `(issuer, sub)` for Authentik. **Do not** enforce a single global email across providers if that would block a second account — or, if global uniqueness is desired for UX, **still do not** merge identities; registration/SSO would fail closed with a clear error instead of linking.

Owner must choose (open question O2). Until decided, implementers must not silently link.

### C.4 Explicit non-goals (coexistence)

- Account merging / linking / “same human” reconciliation.
- Multi-user budget sharing, invited members, or Authentik groups → shared budget ACL.
- Using Authentik as the password store for local users (local users are app-local).
- SCIM full directory sync (V1 = login-time auto-provision only).

---

## 7. Part D — Tenancy and authorization after authentication

Auth answers: **who is the principal?**  
Authorization answers: **may this principal act on this budget?**

From `ARCHITECTURE.md` §7.2 (reinforced here as security requirements):

1. Every budget-scoped table carries `budget_id`.
2. Budget (and tenant roots as needed) carry stable **`owner_user_id`** (or equivalent FK).
3. All reads/writes are scoped by **owner + `budget_id`**.
4. **Never trust** client-supplied ownership alone.
5. Module conventions are **not** a security control — ownership checks belong in **use-cases and persistence**.

### D.1 Request authorization pattern

```text
1. Security filter authenticates → AuthenticatedUser { userId, accountId, provider }
2. Controller extracts budgetId from path/body
3. Use-case loads budget WHERE id = :budgetId AND owner_user_id = :userId
4. If missing → 404 or 403 (prefer not revealing others' existence; be consistent)
5. Proceed with domain rules (BUSINESS_RULES.md)
```

Unauthenticated access to financial and mutating endpoints → **401**.  
Wrong owner → **403/404**. No “global admin” role in V1 product auth.

### D.2 What auth does *not* do

- Auth does **not** replace financial invariants (conservation, atomic reject, etc.).
- Auth does **not** grant cross-budget access within the same user beyond “user owns many budgets” (still check each `budget_id`).
- Events / listeners are not a trust boundary: handlers must not bypass ownership checks (`ARCHITECTURE.md` §6.3).

### D.3 Account model reminder

```text
Account 1 ──1── User     (≤1 user per account; BUSINESS_RULES / ARCHITECTURE)
Account 1 ──*── Budget   (primary created at account creation)
Budget   ──*── nodes, edges, periods, journals, …
```

---

## 8. Data model sketch (identity module)

Illustrative — exact schema in Flyway when implementing:

| Entity / table | Key fields |
|----------------|------------|
| `account` | `id`, `timezone`, `created_at`, … |
| `user` | `id`, `account_id` (unique), `identity_provider`, `display_name`, … |
| `local_credential` | `user_id`, `login_identifier` (unique among LOCAL), `password_hash`, `failed_attempts`, `locked_until`, … |
| `oidc_identity` | `user_id`, `issuer`, `subject`, `email` (nullable), unique `(issuer, subject)` |
| `budget` | `id`, `owner_user_id`, `name`, `is_primary`, … |

No password fields on Authentik users. No OIDC fields required on local users.

---

## 9. API / UI surface (security-relevant)

| Surface | Notes |
|---------|-------|
| `UI-01` Sign-in | Local credentials + “Sign in with Authentik”; errors generic for local auth failures |
| `UI-02` Account setup | Local: identifier + password + timezone. Authentik first run: timezone (no password) |
| Register / login / logout / CSRF / `GET /api/me` | Identity adapter only |
| All `/api/**` money and config routes | Authenticated + ownership |

No IAM admin screens in V1.

---

## 10. Threat mitigations checklist

| Threat | Mitigation |
|--------|------------|
| T1 Stuffing | Rate limit + lockout/backoff; generic login errors |
| T2 Password theft | Argon2id/bcrypt; no plaintext; no password logs |
| T3 Session XSS | HttpOnly cookies; tight CSP later; no tokens in `localStorage` |
| T4 CSRF | Spring CSRF with SPA cookie/header pattern |
| T5 OIDC abuse | Fixed redirect URIs; validate `state`/tokens; server-side code exchange |
| T6 Bad provision | Unique `(issuer,sub)`; no email merge; transactional create + primary budget |
| T7 IDOR | `owner_user_id` + `budget_id` in every use-case |
| T8 Secrets | Env/Compose secrets; public config only in Angular |
| T9 Downgrade | TLS + Secure cookies mandatory on public host |

---

## 11. Testing expectations (for later QA / implementation)

| Area | Examples |
|------|----------|
| Local | Register hashes password; login success/fail; lockout after N failures; logout kills session |
| CSRF | Mutating request without token rejected when using cookies |
| OIDC | First login provisions one account + primary budget; second login reuses same user; parallel first-login no duplicate |
| Separation | Local user email ≠ automatic access via Authentik with same email (or fail closed per O2) — **no merge** |
| Authz | User A token/session cannot read/write User B `budget_id` |
| Transport | Cookies marked Secure; HTTPS assumed in prod profile tests |

---

## 12. Open questions for owner

| # | Question | Why it matters | Security recommendation |
|---|----------|----------------|-------------------------|
| **O1** | Session cookies (recommended) vs JWT resource-server / SPA tokens? | CSRF vs XSS tradeoff; Spring config shape | **Session cookies + CSRF** for same-origin SPA |
| **O2** | Email uniqueness: per-provider vs global? | Same human, two accounts vs register clash | Per-provider uniqueness; **never** auto-link |
| **O3** | Local login identifier: email only, username, or either? | Enumeration UX; Authentik email overlap | Email-only for local |
| **O4** | App-only logout vs Authentik RP-initiated logout in V1? | SSO one-click return | App logout required; IdP logout optional |
| **O5** | Password reset / email verification in V1? | Account recovery on public host | Follow-on if mail not ready; no insecure reset |
| **O6** | After Authentik SSO with timezone default UTC: skip `UI-02` or confirm once? | UX (`USER_INTERFACE.md` Q1); ensure account complete before money APIs | Allow money APIs only when account setup status = complete |
| **O7** | Session timeout values? | Balance security vs friction | Document concrete idle/absolute limits at implement time |
| **O8** | Temporary exposure policy until auth ships (Architecture O7)? | Public host risk | VPN / allowlist / private bind until this lands |

---

## 13. Acceptance criteria for this document (Issue #3)

Ready for owner acceptance when:

- [ ] Part A defines secure local register/login (hashing, HTTPS, CSRF-if-cookies, rate limit/lockout, password policy, logout, explicit anti-patterns)
- [ ] Part B defines Authentik OIDC integration, callback, auto-provision (`F02`), claims (`sub`/`iss`/email), env secret placeholders, logout notes
- [ ] Part C states coexistence with **no merge/link**, provider type on account, shared post-login authorization model
- [ ] Part D reinforces `owner_user_id` + `budget_id` checks; auth ≠ ownership
- [ ] Public-host baseline from `ARCHITECTURE.md` is preserved (not weakened)
- [ ] Open questions needing owner choice are listed
- [ ] Owner promotes **Status** to **Accepted**

---

## 14. Decision log

| Date | Decision | Status | Notes |
|------|----------|--------|-------|
| 2026-10-22 | Draft `AUTH.md` for Issue #3: local + Authentik, no account merge/link | Draft / Proposed | Owner requirements; Security Developer |
| 2026-10-22 | Recommend Spring Security session cookies + CSRF for Angular same-origin SPA; OAuth2 Login for Authentik; Argon2id/bcrypt for local passwords | Draft / Proposed | Fits modular monolith + nginx; JWT alternative documented |
| 2026-10-22 | Authentik identity key = `(issuer, sub)`; email not a merge key; auto-provision creates separate account + primary budget | Draft / Proposed | Aligns `F01`/`F02`/`F04` |
| 2026-10-22 | Authorization remains use-case ownership checks per `ARCHITECTURE.md` §7.2 | Draft / Proposed | Auth proves identity only |

---

## 15. Follow-on issue candidates (draft only — not created)

Require human approval before opening GitHub issues:

1. **Implement identity module + Spring Security** — Flyway tables, local register/login/logout, CSRF, session cookies, rate limit/lockout, wire `UI-01`/`UI-02`.
2. **Implement Authentik OIDC** — OAuth2 Client registration, callback, auto-provision, Compose env, smoke test against Authentik.
3. **Ownership enforcement audit** — ensure every budget-scoped use-case filters by `owner_user_id` + `budget_id`; add IDOR tests.
4. **Password reset + email verification** — signed tokens, mail provider, UX.
5. **Optional Authentik RP-initiated logout** and session timeout tuning.
6. **Pre-auth deploy gate (Architecture O7)** — temporary VPN/allowlist policy until auth is live on the public host.
7. **CSP / browser hardening** pass for the Angular app (XSS defense-in-depth).

---

## 16. Severity legend (for future reviews)

| Severity | Meaning |
|----------|---------|
| **Critical** | Direct account or cross-tenant financial data compromise |
| **High** | Realistic exploit on public host without exotic conditions |
| **Medium** | Abuse or hardening gap with limited blast radius |
| **Low** | Defense-in-depth / polish |

**Release blockers for any auth implementation PR:** plaintext passwords; disabled auth on public financial APIs; missing ownership checks; client secret in SPA/git; auto-linking local ↔ Authentik by email.
