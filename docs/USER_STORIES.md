# User Stories — Budget 2.0

| Field | Value |
|-------|-------|
| **Status** | **Draft** — inventory for Issue #3; expand later |
| **Source issue** | [#3 — Decide on features using user stories](https://github.com/zachkingcade/ZK-Dynamic-Flow-Budgeting-Application/issues/3) |
| **Companion** | Domain rules: [`BUSINESS_RULES.md`](./BUSINESS_RULES.md) (**Accepted**). Features: [`FEATURES.md`](./FEATURES.md). UI: [`USER_INTERFACE.md`](./USER_INTERFACE.md). Architecture: [`ARCHITECTURE.md`](./ARCHITECTURE.md). |
| **Format** | One-sentence stories for now; expand later into FEATURES / UI / AUTH docs |

---

### Owner-seeded stories (not all derived from BUSINESS_RULES)

1. As a user accessing from an authentik dashboard If an account does not exist for my authentik account I want one to be created automatically.
2. As a user accessing from an authentik dashboard I want to be able to access the application using my authentik account.
3. As a user accessing from outside authentik I want to be able to create an account without authentik just using the built in system.
4. As a user I want to import transactions from a CSV by mapping which columns are amount, description, and transaction date, and choosing whether to keep amount signs or use absolute values, into a holding area so I can assign them to nodes (no bank-account or CSV-source tracking).
5. As a user I want to be able to see a graph of the automated funding rules.
6. As a user I want to be able to easily assign imported or manual transactions to nodes.

> **Notes:** Stories 1–3 feed later `AUTH.md`. Story 4 is bulk external-transaction import only — not bank sync or reconciliation. Story 5 is a **visual-only** graph for later UI work (not an interactable graph engine). Prefer **node** over bank “account” in domain language.

---

### From BUSINESS_RULES (Accepted) — one sentence each

#### Account and budgets

7. As a user I want a primary budget created automatically when my account is created, and the option to add or switch to more budgets later without needing that for first use.
8. As a user I want to choose a timezone at account setup, defaulting to UTC, so period boundaries and dates use that timezone.

#### Nodes (envelopes)

9. As a user I want to create and name nodes in a budget.
10. As a user I want to mark each node as a pool node or a spending node (whether it accepts external transactions).
11. As a user I want to turn Allow deficit on or off for both pool and spending nodes.
12. As a user I want to see Available at each node and the budget’s pool cash total.

#### Funding graph

13. As a user I want to create directed funding edges from funder to fundee for automatic funding and reclaim.
14. As a user I want the system to reject any funding edge that would create a cycle.
15. As a user I want a node to have multiple parent funders.
16. As a user I want to choose a child node’s request strategy: equal, proportions, or priority.
17. As a user I want proportion parts on parents to renormalize over parents still able to pay when one runs out.
18. As a user I want priority values to be non-negative integers where a lower number is higher priority and 0 is highest.

#### Automatic funding rules

19. As a user I want to set a baseline (scheduled amount) on each funding edge.
20. As a user I want to set a hard max on per-period cumulative automatic funding into a node.
21. As a user I want automatic funding to allow soft overflow above baseline up to the hard max when funders can pay.
22. As a user I want automatic funding to run when I open a period, or every X periods per rule configuration.
23. As a user I want automatic funding to run at transaction time when current Available does not cover the posting.
24. As a user I want period-open automatic funding to apply only once for a given intent (idempotent on retry).
25. As a user I want unallocated cash kept as high in the graph as possible and lower nodes funded only as needed.

#### Targets

26. As a user I want to set a target on a spending node for over/under reporting.
27. As a user I want the system to reject a target greater than the hard max when a hard max is set.
28. As a user I want the system to reject lowering the hard max below the current target until I lower the target first.
29. As a user I want to see over/under target status without that alone accepting or rejecting a transaction.

#### Periods and dating

30. As a user I want to define a period as every X of unit Y starting on date Z.
31. As a user I want a period to stay open until I close it, and I do not want the next period to open automatically when I close one.
32. As a user I want to enter a transaction whose transaction date falls in the open period even if the system date is after the period’s calendar end.
33. As a user I want a transaction whose transaction date falls outside the open period to be rejected.
34. As a user I want every transaction to have a transaction date and a system date.
50. As a user I want to change period cadence only when no period is open, not mid-period.
51. As a user I want to explicitly open a period after closing one (or before the first), so I can adjust cadence in between if needed.

#### Income

35. As a user I want to post income that increases Available on the Root income pool by default.
36. As a user I want to override income so Available lands on a chosen node instead of Root.

#### Expenses and rejects

37. As a user I want to post external expenses only to spending nodes, not pool nodes.
38. As a user I want an uncovered expense with Allow deficit off to be rejected with no balance or funding changes.
39. As a user I want an expense with Allow deficit on to auto-fund as far as ancestors and hard max allow, then post with negative Available for any remainder.
40. As a user I want to be told why a transaction was rejected.

#### Manual fund moves

41. As a user I want to manually move Available from any node to any other node without needing a funding edge.
42. As a user I want manual fund moves not to change hard max or period funded-in counters.
43. As a user I want a warning when a manual move amount would have exceeded the destination’s automatic hard max.

#### Reclaim

44. As a user I want to reclaim Available upward from descendants along the funding graph (full reclaim).
45. As a user I want to reclaim Available down to combined baselines (sum of per-edge baselines).
46. As a user I want a reclaim that finds nothing reclaimable to succeed and report $0 reclaimed.
47. As a user I want reclaim to happen only when I initiate it, not automatically after an expense.

#### Integrity and corrections

48. As a user I want changes to limits and automatic funding rules to be audited with old and new values.
49. As a user I want corrections to use explicit reversing or compensating entries rather than silent history rewrites.

#### History

52. As a user I want a funding history of zero-sum internal movements (automatic funding, manual moves, reclaim) separate from the list of external income and expense transactions.

---

### Possible (inferred or not absolute from BUSINESS_RULES)

These are useful product ideas that are **not** strict Accepted requirements or are only lightly implied. Owner should promote, defer, or drop before locking FEATURES.md.

P1. As a user I want a **visual-only** view of the funding topology and automated funding rules (extends story 5; not a fully interactable graph engine — configuration stays in ordinary flows).
P2. As a user I want CSV column-mapping import into a holding area and assignment to nodes (story 4 clarified: no bank-account or CSV-source tracking).
P3. As a user I want client-side pre-checks before posting, knowing the server will re-evaluate and may still reject.
P4. As a user I want to see which funding edges ran, whether deficit was used, and period funded-in versus hard max after a post.
P6. As a user I want starter/preset node catalogs when creating a budget (explicitly deferred in BUSINESS_RULES).
P8. As a user I want Authentik SSO and/or built-in local accounts (owner stories 1–3; auth design deferred to AUTH.md, not BUSINESS_RULES).

> **Removed:** Former P5 (hidden journals) — not a user story. Former P7 (auto primary budget) — promoted into story 7 / FEATURES F04 / BUSINESS_RULES 8a.
