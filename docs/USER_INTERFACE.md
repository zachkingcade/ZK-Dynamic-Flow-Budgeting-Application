# User Interface — Budget 2.0 (V1)

| Field | Value |
|-------|-------|
| **Status** | **Draft / Proposed** — awaiting owner acceptance as the V1 page inventory |
| **Source issue** | [#3 — Decide on features using user stories](https://github.com/zachkingcade/ZK-Dynamic-Flow-Budgeting-Application/issues/3) |
| **Companion** | Features: [`FEATURES.md`](./FEATURES.md). Stories: [`USER_STORIES.md`](./USER_STORIES.md). Domain rules: [`BUSINESS_RULES.md`](./BUSINESS_RULES.md) (**Accepted**). Auth: [`AUTH.md`](./AUTH.md). Architecture: [`ARCHITECTURE.md`](./ARCHITECTURE.md) (Angular SPA). |
| **Scope** | Browser screens (pages + modals), information architecture, primary flows, and feature mapping (`implements FXX`). Not pixel design, component library, or auth protocol detail. |

### Status legend

| Label | Meaning |
|-------|---------|
| **Draft / Proposed** | Team UI/UX recommendation for Issue #3; not binding until owner accepts |
| **Accepted** | Owner-approved V1 UI baseline |

### How to use screen IDs

Stable IDs are **UI-01**, **UI-02**, … . An ID may be a **page** or a **modal** (not every ID is its own route). Features cite as `implements FXX`. Do not renumber after acceptance without an explicit owner decision.

### UX principles (V1)

1. **Combine related work** — prefer one page with clear sections/modals over many thin routes.
2. **UI “Accounts” = domain nodes** — the Accounts page shows allocation nodes (pool vs spending). Prefer **node** in help/API; the nav label may say Accounts. Do not imply bank accounts.
3. **Money truth is server-owned** — client may pre-check (`F36`); server always re-evaluates; rejects show API reasons (`F29`).
4. **Never expose background double-entry** — no journals, ledger lines, or `{Node} Expense` screens.
5. **Funding graph is visual-only** — top band of the Funding page; configure edges/rules in the form/list area below, not by canvas editing.
6. **Auth is screen-level only** — sign-in + account setup; protocol detail → `AUTH.md`.
7. **Timezone once** — IANA timezone on **account setup** (default UTC); also editable later under Account settings if product allows (V1: set at setup; settings may show read-only).
8. **Primary budget first** — one budget is created automatically at account creation; multi-budget is a power-user path, not the first-run focus.
9. **External vs internal** — **Transactions** = money in/out of the system (income/expense). **Funding history** = zero-sum internal movements (auto-funding, manual moves, reclaim).

---

## 1. Navigation / information architecture

### 1.1 App shells

| Shell | When | Contents |
|-------|------|----------|
| **Unauthenticated** | Signed out | Sign-in (UI-01) |
| **Account setup** | New local register / first-time timezone | Account setup (UI-02) |
| **Budget workspace** | Signed in with active budget (always after first login once primary budget exists) | Chrome + primary nav |

### 1.2 Global chrome (budget workspace)

| Region | Behavior |
|--------|----------|
| **Active budget** | Shows current budget name. Under an account/nav menu: **Switch budget…** → Budget switcher hub (UI-03). Most users never need this. |
| **Pool cash summary** | Read-only budget pool cash (`Σ Available`). |
| **Open period chip** | Current open period dates, or “No open period”; jumps to period controls on Accounts (UI-05). |
| **Primary nav** | Accounts · Funding · Transactions · Assign · Funding history · Settings |

Suggested Angular routes (illustrative):

```text
/sign-in
/account/setup
/budgets                          # switcher hub (power user)
/b/:budgetId/accounts             # balances, nodes, period open/close
/b/:budgetId/funding              # visual + rules + reclaim
/b/:budgetId/transactions         # external in/out
/b/:budgetId/assign               # holding / assignment (+ CSV modal)
/b/:budgetId/funding-history      # internal zero-sum movements
/b/:budgetId/settings             # account/budget settings incl. period cadence
```

Modals (not separate primary routes): Add budget, Add/edit transaction, CSV import, Manual fund move, Post outcome, Reversal confirm.

### 1.3 IA diagram (conceptual)

```text
[Sign-in] → [Account setup: register fields + timezone]
                │  (creates account + primary budget)
                ▼
         ┌── Budget workspace (primary budget active) ──┐
         │ Accounts (balances, nodes, period open/close) │
         │ Funding (visual 35–40% + rules + reclaim)     │
         │ Transactions (external list + Add)            │
         │ Assign (holding; CSV import modal)            │
         │ Funding history (internal zero-sum)           │
         │ Settings (cadence, …)                         │
         └───────────────────────────────────────────────┘
                ▲
   power user: nav → Switch budget → hub → Add budget (modal)
```

### 1.4 Cross-cutting patterns (not primary pages)

| Pattern | Role | Features |
|---------|------|----------|
| **Client pre-check** | Before money/config submit; labeled estimate; server may still reject | `F36` |
| **Reject banner** | Atomic reject; plain-language reason | `F29`, `F11`, `F22`, `F25`, `F28` |
| **Post outcome panel** | After successful post/move/reclaim/period open: edges run, deficit?, funded-in vs hard max | `F37` |
| **Hard-max warning (manual)** | Non-blocking when manual move would exceed destination automatic hard max | `F32` |
| **Reversal** | Action on a row in **Transactions** and **Funding history** — creates explicit reverse/compensate entry; not a standalone page | `F35` |

---

## 2. Screen inventory

> **Note:** “Page inventory” here means the product surface map. Several former thin “pages” are **sections or modals** on a shared screen.

### UI-01 — Sign-in (page)

| | |
|--|--|
| **Purpose** | Access the app via Authentik SSO or local credentials. |
| **Primary job** | Authenticate. |
| **Contents / actions** | Authentik / SSO entry; local sign-in; link to account setup / register (UI-02). |
| **Empty / error / success** | **Error:** invalid credentials / SSO failure. **Success:** if setup incomplete → UI-02; else open workspace on **primary** (or last active) budget. |
| **implements** | `F01`, `F02` |

---

### UI-02 — Account setup (page)

| | |
|--|--|
| **Purpose** | One general account setup page: local registration fields (when not using Authentik) **and** timezone. |
| **Primary job** | Create/finish account so the user can use the app. |
| **Contents / actions** | For local path: identity/password fields (detail → `AUTH.md`) + IANA timezone (default **UTC**). For Authentik first landing with incomplete setup: timezone (and any required profile fields) only. On successful account creation, system creates a **primary budget** automatically and lands the user in the workspace — not an empty budget list. |
| **Empty / error / success** | **Error:** validation, duplicate identity, invalid timezone. **Success:** account ready; primary budget active → Accounts (UI-05). |
| **implements** | `F03`, `F05`, `F04` (primary budget creation) |

---

### UI-03 — Budget switcher hub (page, power user)

| | |
|--|--|
| **Purpose** | List and switch budgets; entry to add another budget. **Not** part of first-run for typical users. |
| **Primary job** | Switch or manage budgets. |
| **Contents / actions** | Reached from account/nav menu → **Switch budget…**. List budgets; open/switch; **Add budget** opens UI-04 modal. Primary budget is marked. |
| **Empty / error / success** | Should rarely be empty (primary always exists). **Error:** load/ownership failure. |
| **implements** | `F04` |

---

### UI-04 — Add budget (modal)

| | |
|--|--|
| **Purpose** | Create an additional budget (power user). |
| **Primary job** | Add one budget. |
| **Contents / actions** | Opened from UI-03 **Add budget**. Name + confirm. No timezone. No starter node presets. |
| **Empty / error / success** | **Success:** new budget available; optional switch to it. |
| **implements** | `F04` |

---

### UI-05 — Accounts (page)

| | |
|--|--|
| **Purpose** | Single page for allocation **nodes** (UI label: Accounts): see balances, create/edit nodes, and period open/close while looking at balances. |
| **Primary job** | Manage nodes and period lifecycle in context of Available. |
| **Layout / contents** | |
| | **Period strip (top):** current or upcoming period date range; status Open / None; actions **Close period** and **Open period**. Closing does **not** auto-open the next period — user reviews balances and may change cadence in Settings before opening again. |
| | **Summary:** pool cash total; list/table of nodes with Available, pool vs spending, Allow deficit, hard max / period funded-in, target (spending, reporting-only). |
| | **Node create/edit:** inline panel or side drawer on this page (name, pool vs spending, Allow deficit, hard max, target, request strategy for multi-parent fundees). |
| | **Manual fund move:** button opens **UI-05a** modal (any → any). |
| **Empty / error / reject / success** | **Empty nodes:** CTA create first node. **No open period:** warn that external posts reject until open. **Period close:** confirm; then remain with no open period. **Period open:** may run period-lifecycle auto-funding; show outcome (`F37`); idempotent retry. **Config reject:** target/max rules (`F22`). |
| **implements** | `F06`, `F07`, `F08`, `F09`, `F13`, `F16`, `F21`, `F22`, `F18`, `F24`, `F25` (messaging), `F32` (via modal), `F37` |

#### UI-05a — Manual fund move (modal on Accounts)

| | |
|--|--|
| **Purpose** | Move Available any node → any node. |
| **Contents** | Source, destination, amount; hard-max warning if applicable; confirm. |
| **implements** | `F32`, `F36`, `F37` |

---

### UI-06 — Funding (page)

| | |
|--|--|
| **Purpose** | Configure and understand the funding graph and rules; reclaim from here. |
| **Primary job** | Funding structure and reclaim. |
| **Layout / contents** | |
| | **Top ~35–40%:** **visual-only** graph of topology and automated funding rules (no drag-to-connect / canvas edit). |
| | **Remainder:** funding rules — edge list create/edit (funder, fundee, baseline); help for soft overflow / multi-parent; links or inline edit for node strategy (or deep-link to Accounts node edit). |
| | **Reclaim:** primary action on this page — full reclaim or to combined baselines; $0 reclaimed is success. |
| **Empty / error / reject / success** | **Cycle edge reject** (`F11`). **Reclaim $0** is not an error. Graph empty state when insufficient nodes/edges; still show rules list. |
| **implements** | `F10`, `F11`, `F12`, `F14`, `F15`, `F17` (help), `F33`, `F37` |

---

### UI-07 — Account / budget settings (page)

| | |
|--|--|
| **Purpose** | Settings that are not day-to-day money ops — including **period cadence**. |
| **Primary job** | Configure account/budget settings. |
| **Contents / actions** | Period cadence: every **X** of unit **Y** starting on **Z**; show account timezone. **Cadence may be edited only when no period is open** (after close or before first open); reject mid-period changes. Optional: show timezone (from setup); rule/limit change log (old → new) if not placed elsewhere (`F34`). |
| **Empty / error / reject / success** | **Reject:** attempt to change cadence while a period is open. |
| **implements** | `F23`, `F05` (display), `F34` |

---

### UI-08 — Transactions (page)

| | |
|--|--|
| **Purpose** | All **posted external** transactions (income and expenses — money in/out of the system). |
| **Primary job** | Review and add external transactions. |
| **Contents / actions** | Filterable list: transaction date, system date, description, amount, node, type (income/expense). **Add** (or Manual transaction) opens a form/modal covering expense **and** income (Root default + override for income; spending nodes only for expenses). **Reverse** on a row (`F35`). View post outcome if stored. No journal lines. |
| **Empty / error / reject / success** | Same domain rejects as before (period date, pool node expense, uncoverable + deficit off, etc.). |
| **implements** | `F25`, `F26`, `F27`, `F28`, `F29`, `F35`, `F36`, `F37` |

---

### UI-09 — Assignment holding (page)

| | |
|--|--|
| **Purpose** | Assign imported or unassigned external rows to nodes before/as they post. |
| **Primary job** | Clear the holding queue. |
| **Contents / actions** | Queue of holding items; assign to nodes; commit via same rules as Transactions. **Import CSV** opens **UI-09a** modal. |
| **implements** | `F31`, `F26`/`F27` (commit path), `F29`, `F36`, `F37` |

#### UI-09a — CSV import (modal on Assignment)

| | |
|--|--|
| **Purpose** | Map CSV columns → holding rows (not bank sync). |
| **Contents** | File upload; map **amount**, **description**, **transaction date**; keep signs vs absolute value; preview; import. No bank/source tracking. |
| **implements** | `F30` |

---

### UI-10 — Funding history (page)

| | |
|--|--|
| **Purpose** | List **zero-sum internal** movements: automatic funding legs, manual fund moves, and reclaim — not external income/expense. |
| **Primary job** | Inspect internal money movement history. |
| **Contents / actions** | Chronological (filterable) list: when, type (auto-fund / manual / reclaim), from node, to node, amount, period, links to outcome detail. **Reverse** option on eligible rows (`F35`). Distinct from Transactions (UI-08) and from rule/limit config audit (`F34` on Settings). |
| **Empty / error / success** | **Empty:** no internal movements yet. |
| **implements** | `F38`, `F32`/`F33`/`F18`/`F19` (as history of those actions), `F35`, `F37` |

---

## 3. Feature coverage matrix (F01–F38)

| Feature | Primary screen(s) |
|---------|-------------------|
| F01 Authentik SSO | UI-01 |
| F02 Authentik auto-provision | UI-01 → UI-02 if needed |
| F03 Local registration | UI-02 |
| F04 Multiple budgets + primary default | UI-02 (create primary), UI-03, UI-04 |
| F05 Account timezone | UI-02; display on UI-07 |
| F06–F09, F13, F16, F21, F22 Nodes / Available / strategy / max / target | UI-05 |
| F10–F12, F14, F15, F17 Edges + visual graph | UI-06 |
| F18 Period-lifecycle auto-fund | UI-05 (on Open period) |
| F19 Transaction-time auto-fund | UI-08 (and UI-09 commit) |
| F20 Keep money high | Behavior; visible via outcomes |
| F23 Period cadence (edit only when no open period) | UI-07 |
| F24 User-closed periods; no auto-open after close | UI-05 |
| F25 Dating rules | UI-05 messaging; UI-08 / UI-09 |
| F26–F29 Income, expense, deficit/reject, reasons | UI-08 |
| F30 CSV import | UI-09a |
| F31 Assign to nodes | UI-09 |
| F32 Manual moves | UI-05a; history on UI-10 |
| F33 Reclaim | UI-06; history on UI-10 |
| F34 Rule/limit audit | UI-07 |
| F35 Explicit corrections / reverse | UI-08, UI-10 |
| F36 Client pre-checks | Cross-cutting |
| F37 Post outcome visibility | Panels/modals from UI-05–UI-10 |
| F38 Funding history (internal zero-sum list) | UI-10 |

---

## 4. Non-screens / exclusions (V1)

| Exclusion | Rationale |
|-----------|-----------|
| Separate register vs timezone pages | Combined in UI-02 |
| First-run empty budget wizard | Primary budget auto-created |
| Standalone create-budget page | Modal UI-04 from switcher |
| Separate home / node list / node edit pages | Combined Accounts UI-05 |
| Separate graph vs edges pages | Combined Funding UI-06 |
| Standalone period lifecycle page | On Accounts UI-05 |
| Separate income vs expense vs activity pages | Combined Transactions UI-08 |
| Standalone CSV page | Modal on Assign UI-09 |
| Standalone manual-move or reclaim pages | Modal / action on Accounts / Funding |
| Standalone reversal page | Row action on Transactions + Funding history |
| Journal / `{Node} Expense` UI | Background only |
| Interactable graph canvas editor | Visual-only band on UI-06 |
| Bank account / sync screens | Out of scope |

---

## 5. Key user flows

1. **First run (local):** UI-01 → UI-02 (register + timezone) → primary budget created → UI-05 Accounts → create nodes → UI-07 set cadence (if needed) → Open period on UI-05 → UI-08 add income/expenses.
2. **First run (Authentik):** UI-01 (F01/F02) → UI-02 timezone if needed → same workspace path.
3. **Close period:** UI-05 review balances → Close → **no** auto-open → optionally UI-07 change cadence → Open period when ready.
4. **Configure funding:** UI-06 (see graph + edit rules) → reclaim as needed.
5. **CSV:** UI-09 → Import modal → assign → posts appear on UI-08; funding legs on UI-10.
6. **Power-user second budget:** nav → Switch budget → UI-03 → Add budget modal UI-04.

---

## 6. Accessibility and mobile (V1)

| Area | Expectation |
|------|-------------|
| **Semantics** | Labels on forms; reject/success not color-only; focus traps in modals. |
| **Graph (UI-06)** | Visual band plus rules list below so topology is never graph-only. |
| **Mobile** | Single-column; graph may scroll or collapse emphasizing the rules list. |
| **Dates** | Show account timezone near period and transaction dates. |
| **Confirms** | Close period, reclaim, reversals, large moves. |

---

## 7. Open UX questions (owner)

| # | Question | Why it matters |
|---|----------|----------------|
| Q1 | After Authentik SSO with timezone already defaulted to UTC, skip UI-02 entirely or always confirm once? | Friction |
| Q2 | Is **Root** income pool system-created with the primary budget, or user-created on Accounts? | Empty state |
| Q3 | Node create/edit on Accounts: side drawer vs expand-in-list? | Density |
| Q4 | Persist rejected external attempts in Transactions, or inline-only? | `F29` history |
| Q5 | Rule/limit audit (`F34`): only under Settings, or also filter on Funding history? | IA |
| Q6 | Manual move hard-max warning: checkbox ack vs warn-and-proceed? | `F32` |
| Q7 | May timezone change later in Settings, or immutable after UI-02? | `F05` |

**Recommendation:** Confirm timezone once for SSO if unset; system-create Root with primary budget; side drawer for node edit; inline rejects only in V1; `F34` under Settings; warn-and-proceed for manual max; timezone immutable in V1 unless you decide otherwise.

---

## 8. Acceptance criteria for this document (Issue #3)

Ready for owner acceptance when:

- [ ] Screen map matches owner combine/modal decisions (Accounts, Funding split layout, Transactions, etc.)
- [ ] Primary budget auto-create and no auto-open-after-close are reflected
- [ ] Every **In** feature F01–F38 maps to a screen or pattern
- [ ] External vs internal history split is clear (UI-08 vs UI-10)
- [ ] Explicit exclusions: no journals, no graph editor, no bank screens
- [ ] Owner promotes **Status** to **Accepted**

---

## 9. Decision log

| Date | Decision | Status | Notes |
|------|----------|--------|-------|
| 2026-10-21 | Initial thin page inventory UI-01–UI-22 | Superseded | Too many standalone pages |
| 2026-10-21 | Owner combine/modal pass | Draft | Account setup merged; primary budget default; Accounts/Funding/Transactions combined; CSV & moves as modals; reclaim on Funding; Funding history = internal zero-sum; reverse on list rows; no auto-open after close; cadence change only when no open period |

---

## 10. Follow-on candidates (not created)

1. Angular routes/modals matching UI-01–UI-10.
2. Reject-reason copy deck ↔ API codes.
3. `AUTH.md` field detail for UI-01/UI-02.
4. Visual design system choice.
