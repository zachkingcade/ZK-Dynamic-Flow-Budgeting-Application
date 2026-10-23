# Features — Budget 2.0 (V1)

| Field | Value |
|-------|-------|
| **Status** | **Draft / Proposed** — awaiting owner acceptance as the V1 feature baseline |
| **Source issue** | [#3 — Decide on features using user stories](https://github.com/zachkingcade/ZK-Dynamic-Flow-Budgeting-Application/issues/3) |
| **Companion** | Stories: [`USER_STORIES.md`](./USER_STORIES.md). Domain rules: [`BUSINESS_RULES.md`](./BUSINESS_RULES.md) (**Accepted**). UI: [`USER_INTERFACE.md`](./USER_INTERFACE.md) (`implements FXX`). Auth: [`AUTH.md`](./AUTH.md). Architecture: [`ARCHITECTURE.md`](./ARCHITECTURE.md). |
| **Scope** | Product capabilities for V1. Not UI screen layout detail, stack detail, or auth protocol design. |

### Status legend

| Label | Meaning |
|-------|---------|
| **In** | Planned for V1 |
| **Deferred** | Intentionally out of V1; may return later |

### How to use feature IDs

Stable IDs are **F01**, **F02**, … (zero-padded). Later docs (especially `USER_INTERFACE.md`) should cite capabilities as `implements FXX`. Do not renumber after acceptance without an explicit owner decision.

---

## Auth

Auth mechanism (OIDC/SSO flows, password policy, session/token design) is **deferred to `AUTH.md`**. V1 product features below are high-level only.

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F01** | Authentik SSO access | User can access the application using an Authentik account (e.g. from an Authentik dashboard). | 2, P8 | In |
| **F02** | Authentik auto-provision | On first Authentik access, if no account exists for that identity, create one automatically. | 1, P8 | In |
| **F03** | Local account registration | User can create and use an account via the built-in system without Authentik. | 3, P8 | In |

---

## Account and budgets

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F04** | Primary + optional budgets | On account creation, a **primary budget** is created automatically. User may add/switch additional budgets later (power-user); each budget is an isolated pool. Multi-budget is available but not the first-run focus. | 7, P7 | In |
| **F05** | Account timezone | User chooses an IANA timezone at **account setup** (same flow as local registration / first-time setup); defaults to UTC; used for period boundaries and date interpretation for all budgets under the account. | 8 | In |

---

## Nodes (envelopes)

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F06** | Create and name nodes | User can create and name nodes within a budget. | 9 | In |
| **F07** | Pool vs spending nodes | User marks each node as pool (no external transactions) or spending (accepts external transactions). | 10 | In |
| **F08** | Allow deficit | User can turn Allow deficit on or off for both pool and spending nodes. | 11 | In |
| **F09** | Available and pool cash visibility | User can see Available at each node and the budget’s pool cash total. | 12 | In |

---

## Funding graph (configuration)

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F10** | Funding edges | User can create directed funding edges (funder → fundee) used by automatic funding and reclaim. | 13 | In |
| **F11** | Cycle rejection | System rejects any funding edge that would create a cycle (DAG only). | 14 | In |
| **F12** | Multi-parent funders | A node may have multiple parent funders. | 15 | In |
| **F13** | Request strategies | Child node request strategy is equal, proportions, or priority, with proportion renormalize and priority semantics (non-negative integers; lower = higher priority; 0 highest). | 16, 17, 18 | In |

---

## Funding graph (visualization)

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F14** | Visual funding graph | User can see a **visual-only** graph of funding topology and automated funding rules (on the Funding screen with rules below). Not a fully interactable graph editor/engine; configuration of edges/rules is via ordinary forms/lists, not graph-canvas editing. | 5, P1 | In |

---

## Automatic funding

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F15** | Edge baselines | User can set a baseline (scheduled amount) on each funding edge. | 19 | In |
| **F16** | Hard max | User can set a hard max on per-period cumulative **automatic** funding into a node. | 20 | In |
| **F17** | Soft overflow | Automatic funding may exceed baseline up to the hard max when funders can pay. | 21 | In |
| **F18** | Period-lifecycle auto-funding | Automatic funding runs on period open and/or every X periods per rule config; period-open application is idempotent for a given intent. | 22, 24 | In |
| **F19** | Transaction-time auto-funding | Automatic funding runs at transaction time when current Available does not cover the posting. | 23 | In |
| **F20** | Keep money high | System preference: keep unallocated cash as high in the graph as possible; fund lower nodes only as needed. | 25 | In |

---

## Targets

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F21** | Spending-node targets | User can set a target on a spending node for over/under reporting; target alone never accepts or rejects a transaction. | 26, 29 | In |
| **F22** | Target vs hard max constraints | System rejects target > hard max when max is set, and rejects lowering hard max below current target until the target is lowered first. | 27, 28 | In |

---

## Periods and dating

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F23** | Period cadence | User defines a period as every X of unit Y starting on date Z. Cadence may be changed **only when no period is open**; mid-period changes are rejected; closed periods stay immutable. | 30, 50 | In |
| **F24** | User-closed periods | A period stays open until the user closes it. **Closing does not auto-open** the next period; the user must explicitly open a period. While none is open, external posts are rejected. | 31, 51 | In |
| **F25** | Transaction dating rules | Every transaction has a transaction date and a system date; dates in the open period are allowed even if system date is after calendar end; dates outside the open period are rejected. | 32, 33, 34 | In |

---

## Income and expenses

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F26** | Income posting | User can post income that increases Available on the Root income pool by default, or on a chosen override node. | 35, 36 | In |
| **F27** | External expense posting | User can post external expenses only to spending nodes (not pool nodes). | 37 | In |
| **F28** | Cover, deficit, and reject | Uncovered expense with Allow deficit off is rejected with no balance/funding changes; with Allow deficit on, system auto-funds as far as ancestors and hard max allow, then posts with negative Available for any remainder. | 38, 39 | In |
| **F29** | Rejection explanations | User is told why a transaction was rejected. | 40 | In |

---

## CSV import and assignment

**Owner clarification (Issue #3):** V1 includes **bulk import of external transactions** from a user-supplied CSV. The product does **not** track bank accounts, reconcile to banks, or remember where the CSV came from. After import, assignment to nodes follows the manual path.

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F30** | CSV column-mapping import | User imports a CSV into a holding area by mapping columns to **amount**, **transaction description**, and **transaction date**, and chooses whether to keep amount signs or take absolute value. No bank-account or CSV-source tracking. | 4, P2 | In |
| **F31** | Assign transactions to nodes | User can easily assign imported (holding) or manually entered transactions to nodes. | 6 | In |

---

## Manual fund moves

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F32** | Manual any-to-any moves | User can manually move Available from any node to any other node without a funding edge; moves do not change hard max or period funded-in counters; warn if the amount would have exceeded the destination’s automatic hard max. | 41, 42, 43 | In |

---

## Reclaim

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F33** | User-initiated reclaim | User can reclaim Available upward along the funding graph (full reclaim, or down to combined baselines); if nothing is reclaimable, succeed and report $0; reclaim is never automatic after an expense. | 44, 45, 46, 47 | In |

---

## Integrity and corrections

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F34** | Rule and limit audit | Changes to limits and automatic funding rules are audited with old and new values (settings / config history — not the same as funding movement history). | 48 | In |
| **F35** | Explicit corrections | Corrections use explicit reversing or compensating entries (available from Transactions and Funding history) rather than silent history rewrites. | 49 | In |

---

## History and UX aids

| ID | Name | Description | Stories | V1 |
|----|------|-------------|---------|----|
| **F36** | Client-side pre-checks | Client may pre-check before posting for UX; server re-evaluates and may still reject. | P3 | In |
| **F37** | Post outcome visibility | After a post, user can see which funding edges ran, whether deficit was used, and period funded-in versus hard max. | P4 | In |
| **F38** | Funding history (internal) | User can list zero-sum **internal** movements (automatic funding, manual moves, reclaim), distinct from the Transactions list of external income/expense. | 52 | In |

---

## Story / possible-item coverage

| Source | Mapping |
|--------|---------|
| Stories 1–3, P8 | F01–F03 (detail → `AUTH.md`) |
| Stories 4, 6, P2 | F30–F31 (CSV as bulk external import; no bank tracking) |
| Story 5, P1 | F14 (visual-only graph) |
| Stories 7–49, 50–52, P7 | F04–F38 as above |
| P3, P4 | F36, F37 |
| **P5** (hidden journals) | **Not a feature** — background double-entry stays invisible; covered only in `BUSINESS_RULES.md` |
| P6 | Deferred (below) |
| P7 | Promoted into **F04** (primary budget on account creation) |

---

## Non-goals and deferred (V1)

Explicitly **not** V1 product features:

| Item | Notes |
|------|-------|
| Bank account tracking / bank sync / reconciliation | Remains a product non-goal; F30 is bulk CSV import of external transactions only, with no bank identity or source tracking. |
| Journal / double-entry UI | Background ledger stays invisible (P5); no user story/feature for journals or `{Node} Expense` accounts in the UI. |
| Interactable funding graph editor | F14 is **visual only**; not a fully interactable graph engine. Edge/rule changes are ordinary config flows. |
| Starter / preset node catalogs | Deferred (P6; also deferred in `BUSINESS_RULES.md`). |
| Multi-currency, credit-card liability models, tax/legal advice | Out of scope per `BUSINESS_RULES.md`. |
| Multi-user collaboration on one budget | Out of tenancy model (`ARCHITECTURE.md`: ≤1 user per account). |
| Auth protocol / IAM design detail | Product access features are F01–F03; mechanism deferred to `AUTH.md`. |

---

## Acceptance criteria for this document (Issue #3)

Ready for owner acceptance when:

- [ ] V1 In/Deferred decisions match owner intent (CSV import, visual-only graph, primary budget, period close/open, F38)
- [ ] Feature IDs are stable and suitable for `implements FXX` in `USER_INTERFACE.md`
- [ ] Auth listed only at high level; journals/P5 not promoted to features
- [ ] Owner promotes **Status** to **Accepted**

---

## Decision log

| Date | Decision | Status | Notes |
|------|----------|--------|-------|
| 2026-10-21 | Draft FEATURES.md from USER_STORIES + Accepted BUSINESS_RULES | Draft / Proposed | Issue #3 |
| 2026-10-21 | CSV import In for V1: column map amount/description/date; sign vs absolute; assign to nodes; no bank/source tracking | Owner clarification | Bulk import only; not bank sync |
| 2026-10-21 | Timezone at account setup only | Owner clarification | Narrows story 8 |
| 2026-10-21 | Funding graph visual In; visual-only (not interactable engine) | Owner clarification | P1 promoted with constraint |
| 2026-10-21 | Hidden journals (P5) not a feature/user story | Owner clarification | BUSINESS_RULES only |
| 2026-10-21 | Primary budget auto-created (F04/P7); close≠auto-open (F24); cadence edit only when no open period (F23); F38 funding history for internal zero-sum | Owner Issue #3 UI vision | Aligns BUSINESS_RULES amendment |
