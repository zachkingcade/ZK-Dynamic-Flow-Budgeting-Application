# Business Rules — Accounting & Funding (Budget 2.0)

| Field | Value |
|-------|-------|
| **Status** | **Accepted** — owner-approved baseline for Budget 2.0 business rules |
| **Source issue** | [#1 — Decide and document the basic accounting business rules](https://github.com/zachkingcade/ZK-Dynamic-Flow-Budgeting-Application/issues/1) |
| **Last updated** | 2026-10-21 |
| **Scope** | Domain / allocation accounting rules for v1 (not UI layout, stack, or bank sync) |

### Status legend

| Label | Meaning |
|-------|---------|
| **ACCEPTED** | Owner-decided or required for financial soundness; implementations must obey once this draft is human-accepted |
| **PROPOSED** | Team recommendation; not binding until the owner promotes it |
| **OPEN** | Still needs an explicit human decision |

---

## 1. Purpose and non-goals

### Purpose

This product allocates **one budget cash pool** across a **directed funding graph** of **holding nodes** (envelopes). The operating preference is to **keep as much money as high in the graph as possible** and fund lower nodes only as needed, within per-node **automatic funding rules**.

### Non-goals (v1)

- Tracking or reconciling **real-world bank accounts** (post-MVP enhancement only)
- UI layout, branding, or interaction design
- Technology stack, persistence engine, or API shape
- Bank linking / Plaid-style sync
- Tax, legal, or licensed-accounting advice
- Multi-currency and credit-card liability models
- Multi-user capability / IAM design
- Preset account catalogs (users name and type their own nodes; any starter presets are a later development decision)

---

## 2. Glossary

| Term | Definition |
|------|------------|
| **Budget pool cash** | Total cash managed by this budget. In v1 this is the allocation control total (`Σ Available`), not a linked bank ledger. |
| **Node** | An allocation place inside the pool (envelope). Not a bank account. User-named; user chooses pool vs spending. |
| **Pool node** | Node that holds Available and may participate in automatic funding; **cannot** accept external expense transactions. |
| **Spending node** | Node the user has marked as able to accept external transactions (expenses, and optionally income override). |
| **Accepts transactions** | User-set flag. `false` → pool node; `true` → spending node. |
| **Available(node)** | Portion of the pool currently held at that node. May be negative only if **Allow deficit** is enabled and gates are met. |
| **`{NodeName} Expense`** | Background double-entry **expense classification** account paired with a spending node (e.g. `Water Expense`). Does **not** hold Available. |
| **External transaction** | Income or expense that changes budget pool cash. Has at least a **transaction date** (when it happened in the real world) and a **system date** (when it was entered). |
| **Funding** | Internal, **zero-sum** move of Available. Automatic funding moves along funding edges; manual fund moves may be any node → any node. Never creates money. |
| **Funding edge** | Directed authorization funder → fundee used by **automatic** funding (and reclaim along the graph). |
| **Automatic funding rules** | Rules that schedule or request funding automatically (period open, every X periods, or at transaction time). Subject to hard max. Follow funding edges. |
| **Manual fund move** | User-initiated internal transfer between **any** two nodes. Does **not** require a funding edge. Does **not** affect hard max or period funded-in. Audited, double-entry, zero-sum. Warns if the amount would have exceeded the destination’s automatic hard max. |
| **Baseline (scheduled amount)** | Intended floor / due amount on a **funding edge** for automatic funding. |
| **Combined baselines** | **Sum of per-edge baselines** for a node (used by reclaim-to-baselines). |
| **Soft overflow** | Automatic funding **above** baseline, still subject to hard max and funder Available. |
| **Hard max** | Cap on **per-period cumulative automatic funding into** the node only. Manual moves never consume or change this counter. |
| **Target** | Expected funding/spend level for **reporting** (over/under). Never alone accepts/rejects a transaction. When a hard max is set, **target must be ≤ hard max**. |
| **Request strategy** | Child-owned rule for splitting a funding ask across multiple parents: **equal**, **proportions**, or **priority**. |
| **Proportions (parts)** | Relative weights on parents (e.g. 2 and 5 → 2/7 and 5/7). Need not be entered as percents that sum to 100; the system **renormalizes** over parents still able to pay. |
| **Priority** | Non-negative integer per parent edge. **Lower number = higher priority.** **0** is the highest possible priority; users may start at 0 or any higher non-negative integer. Negatives are not allowed. |
| **Allow deficit** | Per-node option (pool or spending): after automatic funding has gone as far as funders/max allow, if still short → post and let Available go negative; if off → **reject the entire transaction** (nothing moves). |
| **Reclaim** | User-initiated upward pull of Available from descendants along the graph. Modes: **full reclaim** or **reclaim to combined baselines**. If nothing is reclaimable, the operation succeeds and reports **$0 reclaimed**. |
| **Period** | User-defined cadence (**every X of unit Y starting on date Z**) that stays **open until the user closes it**. Closing does **not** open the next period automatically. While open, only transactions whose **transaction date** falls in that period may be entered. Period boundaries are interpreted in the account **timezone**. |
| **Timezone** | IANA timezone chosen at **account setup**; **defaults to UTC** if unchanged. Used for period boundaries and interpreting transaction/system datetimes for all budgets under the account. |
| **Transaction date** | Real-world date the money movement happened (in the budget timezone unless later design says otherwise). |
| **System date** | Date/time the transaction was entered into the system (may be after the period’s calendar end, while the period is still open). |
| **Reject** | Atomic failure: no balance/ledger change for the transaction and its funding legs. |

**Do not conflate:** Available ≠ baseline ≠ hard max ≠ target ≠ `{Name} Expense`.

---

## 3. Accepted requirements

### ACCEPTED — Product intent

1. **Graph funding with multi-parent nodes.** Nodes may have multiple funders. Example shape still includes Bills → mortgage / electricity / water, but edges form a directed graph.
2. **Keep money high.** Prefer retaining unallocated cash at higher nodes; fund descendants as needed.
3. **Automatic funding modes** include at least:
   - **Fixed / scheduled:** amount X on the node’s period cadence (stable obligations such as water).
   - **Dynamic / on-demand:** retain at funders; move down when a transaction needs cover (fluctuating obligations such as electricity).
4. **Soft overflow within hard max.** Example: water baseline $50/period may automatically pull to $60 if max is $100; attempts that cannot fully cover without Allow deficit → reject.
5. **Target for spending nodes.** Reporting over/under an expected target while pulls/spend obey max and deficit rules.
6. **Target vs hard max.** If a hard max is set, **target cannot be greater than max** (`target ≤ max`). Setting a new max **below** the current target is **rejected**; tell the user to **lower the target first**.
7. **Allocation-only v1.** No real-world bank account tracking in the first version. Bulk CSV import of external transactions (column-mapped, no bank/source identity) is allowed as a product feature; it is not bank sync or reconciliation.
8. **User chooses names and transaction acceptance.** Users name nodes and choose pool vs spending. Starter node presets are deferred.
8a. **Primary budget on account creation.** Creating an account creates one **primary budget** automatically. The user may add more budgets later; each budget is its own economic pool (no cross-budget cash sharing in v1).
9. **Manual moves:** any node → any other node; audited double-entry; zero-sum; **no edge required**; **do not affect** hard max / period funded-in; warn if amount would exceed what automatic max would have allowed.
10. **Document before building.** Rules are decided and written here before implementation on ambiguous terms.

### ACCEPTED — Topology and pulls

11. **Topology is a funding graph** (not levels). Multi-parent allowed. Directed **funding cycles are forbidden** (DAG): config reject + runtime fail-closed.
12. **Multi-parent request strategy (on the child):**
    - **Equal** — split the ask as evenly as possible to the nearest cent among parents still able to pay; when one parent runs out, **continue the remaining ask** among remaining parents (re-split), until funded or all parents are exhausted.
    - **Proportions** — relative **parts** per parent (e.g. 2 and 5 ⇒ 2/7 and 5/7). **Renormalize** over parents still able to pay when one runs out; continue until funded or all exhausted.
    - **Priority** — non-negative integers; **lower = higher priority**; **0** is highest; exhaust higher priority before the next. Ties use a deterministic order (**PROPOSED:** stable edge/node id order).
13. **Atomic funding preview.** Multi-parent / recursive automatic funding is computed **before** the triggering transaction is accepted. If the node cannot be fully covered and **Allow deficit is false**, **reject the whole transaction** — **no money is left parked** on the child from a partial attempt; nothing moves.
14. **Recursive shortfall.** If a parent cannot pay, try that parent’s parents (and so on), respecting each node’s automatic funding rules and request strategies. If still short → Allow deficit or reject (atomic).

### ACCEPTED — Income, periods, auto-funding, reclaim

15. **Income:** Root-by-default with optional override. Double-entry posts income into the system; Available lands on the **income pool at Root** by default, or moves to the override node when set.
16. **Hard max:** **per-period cumulative automatic funded-in** only. Manual moves never change this counter.
17. **Allow deficit** may be enabled by the user on **both pool and spending nodes**.
18. **If Allow deficit is off:** reject the **entire** transaction when automatic funding cannot fully cover it; nothing moves.
19. **If Allow deficit is on:** automatically fund until ancestors cannot pay or hard max is reached; then post and allow Available to go negative for the remainder.
20. **Automatic funding triggers:**
    1. At **period open**, or every **X periods** per rule configuration.
    2. At **transaction time**, if current Available does not cover the transaction about to post.
21. **Upward reclaim:** user-initiated only; recursive over children; **full reclaim** or **reclaim to combined baselines** (sum of per-edge baselines). If nothing to reclaim, succeed with **$0 reclaimed**.
22. **Period definition:** user sets **every X of unit Y starting on Z** (example: every two weeks starting 2026-10-16).
22a. **Cadence changes:** period size/cadence may be changed **only when no period is open** (including after close and before the first open). Mid-period cadence changes are **rejected**. Closed periods remain immutable; only future opens use the new X/Y/Z.
23. **Period lifecycle:** a period remains **open until the user closes it**. **Closing does not automatically open the next period** — the user must explicitly open a period. After close and until the next open, **no** external transactions may be posted. After the calendar end of an still-open period, the user may still enter transactions for that period (system date may be later). They may **not** enter a transaction whose **transaction date** falls outside the open period. Example: period ends 10/15; on 10/16 the user may still add a transaction dated 10/15; they cannot add one dated 10/17 until that period is closed and a period that includes 10/17 is current/open.
24. **Transaction dates:** every transaction has at least **transaction date** (real world) and **system date** (entry time). More date fields may appear in later data-model design.
25. **Timezone:** at **account setup** the user is asked for their timezone; it **defaults to UTC**. Period open/close boundaries and date interpretation use this timezone for all budgets under the account.
25a. **External vs internal visibility:** external income/expense (pool cash in/out) and internal zero-sum funding (automatic funding, manual moves, reclaim) must remain distinguishable in product history/audit. (“Every X periods” auto-fund counts **opened** periods, not calendar gaps while closed.)

### ACCEPTED — Financial integrity

26. **Single economic pool.** Nodes are allocation envelopes over one budget pool.
27. **Conservation.** After every successful post: `sum(Available(n)) = BudgetPoolCash` (integer minor units), including negative Available.
28. **Internal moves are zero-sum** (automatic funding, manual moves, reclaim). Σ Available unchanged; pool cash unchanged.
29. **Automatic funding rules ≠ balances ≠ targets.** The ledger commits.
30. **Target is variance-only** for posting decisions, but **config-constrained** by hard max (`target ≤ max`).
31. **Hard max binds automatic funding only.**
32. **Rejects leave zero change**; successful expenses change pool cash and the spending node’s Available together.
33. **Corrections are explicit** (reversing/compensating entries).
34. **Domain enforcement** of automatic edges, max, deficit gates, period/transaction-date rules, and rejects — not UI-only.
35. **Precision:** integer minor units; deterministic remainder rules for equal/proportion splits.

### ACCEPTED — Double-entry (background)

36. Journals may be hidden in the UI but are the system of record.
37. Each **spending node** has **`{NodeName} Expense`** (e.g. `Water Expense`).
38. Expense posting (conceptually): debit `{NodeName} Expense`, credit the node’s Available (after funding legs); BudgetPoolCash decreases with Σ Available.
39. Manual fund moves are proper double-entry internal transfers (debit destination Available, credit source Available) even when no funding edge exists.
40. `{NodeName} Expense` never holds Available.

### ACCEPTED — Security / integrity (business level)

41. Clients may pre-check for UX; domain re-evaluates from persisted config and balances.
42. Reject is atomic.
43. **Automatic** funding and graph reclaim move only along allowed funding edges. **Manual** fund moves may connect any two nodes and must still be audited.
44. Concurrent automatic pulls must not oversubscribe a funder’s Available or a child’s remaining max.
45. Limit/rule changes are audited (old → new) with effective-time application (no silent rewrite of history).

---

## 4. Conceptual model

### 4.1 Allocation-only for v1 — ACCEPTED

v1 models **allocation truth only**. Real-world bank account tracking is **post-MVP**.

### 4.2 Node types — ACCEPTED

| Kind | Holds Available | Accepts external transactions | Typical use |
|------|-----------------|-------------------------------|-------------|
| **Pool node** | Yes | No | Root income pool, Bills, other holding groups |
| **Spending node** | Yes | Yes | Water, electricity, groceries, mortgage |

Users name nodes and choose the type. Allow deficit is available on **both**.

### 4.3 Double-entry and `{Name} Expense` — ACCEPTED

Background double-entry for all money events. Spending nodes have `{AccountName} Expense` for classification. Funding and manual moves move Available only; expenses post through the expense account.

### 4.4 Separated concerns — ACCEPTED

1. **Ledger balances** — what is true after posts.
2. **Automatic funding rules** — scheduled/transaction-time pulls along edges (max, baselines, request strategy).
3. **Targets / plans** — expected amounts (variance); config must keep `target ≤ max` when max is set.
4. **Manual fund moves** — any-to-any, audited, zero-sum, ignore max counters.

### 4.5 Allow deficit — ACCEPTED

**Sequence when Available is insufficient for a posting that uses automatic funding:**

1. Compute automatic funding up the graph (request strategy → parents → recursive shortfall), respecting remaining hard max — **as a tentative plan**.
2. Stop the plan when no further fundable Available exists up-chain **or** hard max is reached.
3. If still short and **Allow deficit = false** → **reject entirely**; commit nothing.
4. If cover is complete, or **Allow deficit = true** for any remainder → commit funding legs (if any) and the transaction together.

### 4.6 Precision — ACCEPTED

Integer minor units; deterministic cent remainder for equal/proportion splits.

### Terminology cleanup — ACCEPTED

| Informal (early notes) | Precise term |
|------------------------|--------------|
| Minimum funding | **Baseline / scheduled amount** (per edge) |
| Maximum funding | **Hard max** (per-period automatic funded-in) |
| Target funding | **Target** (variance; `≤` hard max when max set) |
| Funding policies | **Automatic funding rules** |
| Group that can’t take expenses | **Pool node** |
| Negative allowed | **Allow deficit** |

---

## 5. Financial invariants (checklist)

After every **successful** posting:

1. `sum(Available) = BudgetPoolCash` (negatives included).
2. Automatic funding, manual moves, and reclaim change distribution only (Σ Available unchanged).
3. Each pool dollar / deficit unit sits in exactly one node’s Available; `{Name} Expense` never holds Available.
4. **Automatic** period funded-in never exceeds hard max.
5. Manual moves do **not** change period funded-in or hard max accounting; they remain zero-sum and audited.
6. Rejected transactions (including failed multi-parent cover) leave **zero** funding and expense effect.
7. Nodes with Allow deficit false never successfully post an uncovered amount.
8. Nodes with Allow deficit true may go negative only after automatic funding stops at empty up-chain capacity or hard max.
9. Multi-parent automatic pulls never double-commit the same parent dollar; exhausted parents cause renormalize/continue among remaining.
10. While a period is open, accepted external transactions must have a **transaction date** inside that period. While **no** period is open, external transactions are rejected.
11. Audit trail can answer: why rejected, who changed max/target, which edges funded a pull, manual any-to-any moves, whether deficit was used, $0 reclaim results; external in/out vs internal zero-sum movements remain distinguishable.
12. Cadence is not changed while a period is open; close does not auto-open the next period.

---

## 6. Income entry — ACCEPTED

**Root-by-default with optional override.**

1. Double-entry brings income into the system.
2. By default, Available increases on the **income pool at Root**.
3. If an override node is set, Available is moved to that node (zero-sum; auditable).

Income is an external inflow. Automatic/manual internal moves are not income.

---

## 7. Topology — ACCEPTED (Graphs)

| Topic | Decision |
|-------|----------|
| Shape | **Directed funding graph** with multi-parent nodes |
| Loops | **Forbidden** (DAG). Config rejects cycle-creating edges; recursive pull fail-closed on cycle detection |
| Who splits demand | **Child node** via equal / proportions / priority |
| Manual moves | **Not** limited to edges (any → any) |

---

## 8. Funding modes, caps, periods, reclaim — ACCEPTED detail

### Soft overflow example

- Water baseline **$50** / period, hard max **$100** (per-period automatic funded-in)
- Period needs **$60** → soft overflow **$10** allowed
- If full cover is impossible within max/ancestors and Allow deficit is false → **reject entire transaction** (nothing moves)
- If Allow deficit is true → fund as far as max/ancestors allow, then post with negative Available for the rest

### Automatic funding times

1. **Period open** (or every **X periods** as configured).
2. **Transaction time**, when Available does not cover the posting.

### Multi-parent shortfall (equal / proportions)

Follow the split among current fundable parents. When one parent runs out, **continue** with remaining parents (equal re-split or proportion **renormalize**) until the need is met or all parents (and their recursive up-chains per rules) are exhausted. This is evaluated **before accept**; failure without deficit ⇒ full reject, no partial parking.

### Priority

Non-negative integers only. Lower number = higher priority. 0 is highest. User may start at 0 or any larger non-negative value. Exhaust higher priority before lower.

### Baselines

Stored **per funding edge**. **Combined baselines** = sum of those per-edge baselines.

### Reclaim

User-only; recursive; full or to combined baselines. Below baseline / negative / nothing to take ⇒ operation returns **$0 reclaimed**.

### Period and transaction dating

- Cadence: every **X** of unit **Y** starting on **Z**, evaluated in the account **timezone**.
- **Timezone** is collected at **account setup** and **defaults to UTC**.
- Remains **open until user closes** it; **close does not auto-open** the next period.
- Cadence may change only when **no period is open**; mid-period changes rejected; closed periods immutable.
- While no period is open, external posts are rejected.
- Entry allowed after calendar end only for transactions whose **transaction date** is still inside the open period.
- Cannot enter a transaction date outside the open period until a covering period is open.
- Each transaction has at least **transaction date** and **system date**.

### Target vs max (config)

- When hard max is set: **target ≤ max**.
- New max below current target → **reject** the max change; instruct user to **lower the target first**.

---

## 9. Worked scenarios — OWNER APPROVED

Assumptions unless a case overrides them:

- Graph may be multi-parent; cycles forbidden.
- Hard max = per-period cumulative **automatic** funded-in.
- Automatic funding may run at period open and at transaction time.
- Failed cover with Allow deficit off is atomic (nothing moves).
- Dollars shown for readability; implementation uses integer cents.

### Scenario I — Income

| Case | Action | Then |
|------|--------|------|
| I1 | Income **$1000**, no override | Pool cash 1000; **Root income pool** Available 1000; Σ 1000 |
| I2 | Income **$1000**, override = Bills | System income post, then move to Bills; Bills Available 1000; Σ 1000 |
| I3 | Manual fund Root → Bills **$300** | Root −300; Bills +300; Σ unchanged; **no edge required**; does **not** change period funded-in; **warn** if amount would exceed Bills’s automatic hard max |
| I4 | Manual fund Water → Groceries **$20** with no funding edge between them | Allowed; audited double-entry; zero-sum |

### Scenario W — Water (fixed, soft overflow, max, multi-parent, deficit)

Assumptions: baseline **$50**/period (combined from per-edge baselines); hard max **$100**; parents **Bills** and **Side**; request strategy **Equal**; Allow deficit **off** unless noted.

| Case | Given | When | Then |
|------|-------|------|------|
| W1 | Water 0; both parents can pay | Period open auto-fund baseline | Pull **$50** total (**$25** + **$25**); Water 50; period funded-in 50 |
| W2 | Period funded-in 50; Water Available 0 after prior spend; max 100 | Expense **$10** | Transaction-time soft overflow pull **$10** (equal **$5**/**$5**); period funded-in 60; expense posts; Water ends 0 |
| W3 | Period funded-in 60; max 100; Allow deficit **off** | Expense that needs **$50** more automatic funding (would make funded-in 110) | Cannot fully cover within max → **reject entire transaction**; balances and period funded-in unchanged; nothing partially moved |
| W4 | Period funded-in 60; max 100; Allow deficit **on**; Available 0 | Expense **$50** | Auto-fund **$40** (to max); post expense **$50**; Water Available **−10**; period funded-in **100** |
| W5 | Period funded-in 60 | Automatic pull exactly **$40** | Allow; period funded-in 100; at max |
| W6 | Duplicate period auto-fund intent | Retry | No second baseline application (idempotent) |
| W7 | Bills empty; Side has cash; strategy **Priority** with Side priority **0**, Bills priority **1** | Need **$40** under max | Take from Side first (then Side’s parents if needed); Bills unused |
| W8 | Both parents partially funded under **Equal**; Bills runs out mid-need | Need continues | Re-split remaining need across remaining fundable parents until done or exhausted |
| W9 | All ancestors empty; Allow deficit **on** | Expense **$20** | No funding legs; Water Available **−20**; pool cash **−20**; Σ conserved |
| W10 | All ancestors empty; Allow deficit **off** | Expense **$20** | **Reject**; no journal/balance change |

### Scenario E — Electricity (dynamic / keep high / recurse / reclaim)

| Case | Given | When | Then |
|------|-------|------|------|
| E1 | Elec Available 0; expense **$75**; Bills ≥ 75; under max | Transaction posts | Auto-pull **$75** from Bills (recurse if needed); then expense; do not pre-fund idle estimates |
| E2 | Need unknown / idle | — | Keep cash at Bills/Root; no speculative fill |
| E3 | Expense **$75**; Bills free **$40**; Root can pay; under max | Transaction | Pull **$40** from Bills, recurse for **$35** from Root (respecting Root rules); then expense |
| E4 | After expense | — | **No** automatic upward reclaim |
| E5 | User **full reclaim** on Bills | Reclaim action | Recurse children; move reclaimable Available up |
| E6 | Water Available **$80**; combined baselines **$50** | User **reclaim to combined baselines** | Reclaim **$30**; leave **$50** at Water |
| E7 | Water Available already **≤** combined baselines (or negative) | Reclaim to baselines / full | Succeeds; reports **$0 reclaimed** |

### Scenario G — Groceries (target + max + deficit + manual)

| Case | Given | When | Then |
|------|-------|------|------|
| G1 | Target 400; max 500; Available 0; funders can pay | Expense/funding under max | Auto-pull as needed ≤ max; report **under target** until activity vs target reaches 400 |
| G2 | Activity reaches 420; max 500 | Status | Allowed; report **over target by 20**; not a reject by itself |
| G3 | Remaining max room insufficient; Allow deficit **off** | Expense needing more than Available + remaining automatic cover | **Reject entire expense**; nothing moved |
| G4 | Remaining max room **$10**; Available 0; expense **$30**; Allow deficit **on** | Expense | Auto-fund **$10**; post **$30**; Available **−20**; period funded-in +10 only |
| G5 | Period automatic funded-in already at max 500 | Manual move **+$200** into Groceries | **Allow with warning**; period funded-in stays 500; conservation holds |
| G6 | Target 400; attempt new max **300** | Config change | **Reject** new max; user must **lower target** first |
| G7 | Target 400; max 500; attempt target **600** | Config change | **Reject** target change (`target ≤ max`) |

### Scenario P — Period dating

| Case | Given | When | Then |
|------|-------|------|------|
| P1 | Period open through **10/15**; today system date **10/16**; period not closed | Enter expense with transaction date **10/15** | Allowed |
| P2 | Same open period | Enter expense with transaction date **10/17** | **Rejected** until a period covering 10/17 is open |
| P3 | User closes period; new period includes 10/17 is **opened** by user | Enter expense dated **10/17** | Allowed only after explicit open; system date is entry time |
| P4 | User closes period | — | Next period is **not** auto-opened; external posts reject until user opens a period |
| P5 | Period open; user attempts cadence change | Change X/Y/Z | **Rejected** (mid-period); must close first |
| P6 | No open period; user changes cadence then opens | Open period | New boundaries use updated cadence; prior closed periods unchanged |

### Scenario X — Integrity

| Case | Expected |
|------|----------|
| X1 | Funding/reclaim/manual moves never change Σ Available |
| X2 | Same dollar not Available on two nodes |
| X3 | Reject ⇒ zero change to balances and period funded-in |
| X4 | Equal/proportion/priority + posted legs ⇒ no double cover; exhausted parents continue among remaining |
| X5 | Concurrent automatic pulls ⇒ no oversubscription of a funder’s Available |
| X6 | `{Name} Expense` increase does not increase Σ Available |
| X7 | Conservation holds with a negative node and positive ancestors |
| X8 | Cycle-creating edge rejected at config; cyclic pull path rejects intent |
| X9 | Manual any-to-any move is audited double-entry and does not require an edge |

---

## 10. Remaining non-blocking notes

Business rules for issue #1 are **Accepted**. Items below are implementation-detail deferrals only (not open product rules):

1. **PROPOSED:** deterministic tie-break when two parents share the same priority (stable edge/node id order).
2. **Deferred:** starter/preadded account presets (explicitly out of scope for now).
3. **Deferred:** additional transaction date fields beyond transaction date + system date (data-model phase).

---

## 11. Acceptance criteria for this document (Issue #1)

Ready for owner acceptance when:

- [x] Glossary separates max, baseline, target, Available, expense account, Allow deficit
- [x] Accepted vs Proposed vs Open updated from owner feedback
- [x] Conservation and zero-sum funding explicit (including negatives)
- [x] Water / electricity / groceries / income / reclaim / multi-parent scenarios updated
- [x] Topology = graphs; income = Root default + override
- [x] Owner approved §9 worked scenarios
- [x] Former narrow OPEN list closed by owner (2026-10-15)
- [x] Timezone at setup, default UTC (2026-10-15)
- [x] Owner accepted this file as the baseline project memory for later implementation

---

## 12. Decision log

| Date | Decision | Status | Notes |
|------|----------|--------|-------|
| 2026-10-12 | Issue #1 opened | — | Issue body |
| 2026-10-14 | Dynamic hierarchical funding; soft overflow; targets; keep money high | ACCEPTED | Owner comment |
| 2026-10-14 | First team draft of BUSINESS_RULES.md | Draft | Team consultation |
| 2026-10-14 | Allocation-only v1; pool vs spending; `{Name} Expense`; automatic funding rules; Allow deficit; multi-parent graph; Root income default + override; per-period max; period X/Y/Z; recurse shortfall; user reclaim; forbid cycles | ACCEPTED | Owner feedback + team terminology |
| 2026-10-15 | Manual any-to-any moves (no edge); priority lower=higher (0 highest, no negatives); proportions as parts with renormalize; remove levels; max auto-only; per-edge baselines; atomic multi-parent reject; deficit on pool+spending; user-closed periods + dual transaction dates; $0 reclaim; target ≤ max config rule; §9 approved | ACCEPTED | Owner correction round |
| 2026-10-15 | Timezone asked at account setup; default **UTC**; document marked Accepted | ACCEPTED | Owner final approval (issue close by owner) |
| 2026-10-21 | Primary budget auto-created on account creation; close does not auto-open next period; cadence change only when no period open; timezone account-setup only (all budgets); external vs internal history distinguishable; CSV bulk import allowed without bank tracking | ACCEPTED amendment | Owner Issue #3 UI/product clarifications (Accountant: PASS) |

---

## 13. Follow-on candidates (not created)

Draft ideas only — require human approval before opening GitHub issues:

1. ADR: funding DAG + child request strategy (equal / proportions / priority)
2. Formal state machine for automatic funding + Allow deficit gate
3. Period open/close engine and transaction-date enforcement
4. Audit event schema for pulls, rejects, manual any-to-any moves, deficit posts, $0 reclaim
5. Post-MVP: real-world account tracking / reconciliation
