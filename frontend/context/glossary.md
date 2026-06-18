# Domain Glossary

> Shared vocabulary for the Ovation Budget Platform. When a term here conflicts with code,
> the canonical authorities win: [`schema.md`](../Ovation-Docs/docs/database/schema.md),
> [`formula-engine.md`](../Ovation-Docs/docs/backend/formula-engine.md),
> [`ui-patterns.md`](../Ovation-Docs/docs/frontend/ui-patterns.md).

## Core concepts

| Term | Meaning |
|---|---|
| **Budget Level (L0–L3)** | A versioned snapshot of a project's budget at a stage. L0 = pre-schematic concept, L1 = schematic, L2 = design development, L3 = construction documents. Never overwritten. |
| **Sub-level** | A revision within a level (L2.1, L2.2…). Each sub-level is a full copy of the prior one; editors work on the new sub-level and resubmit. Displayed as `Level {Level}.{SubLevel}`. |
| **Promotion** | Moving an approved level to the next major level (e.g. L2.x → L3.1). Copies all line items and **freezes `BaselineAmount`** from the parent's `TakeoffAmount`. |
| **Division** | A CSI MasterFormat division (e.g. `03 — Concrete`). The organizing spine of the budget. Seeded, shared across projects. **Earthwork/grading = Division 31** (never seed a Div 29). |
| **Line item** | A cost row within a division. Master library is seeded; custom rows are added inline (`IsCustom`). |
| **Cost code** | Identifier on a line item (e.g. `03-3101`, `06-1100`). |
| **Category** | Line-item classification: `S` Standard · `A` Allowance · `RR` Repair & Replace · `B` Bond/Insurance · `F` Fee · `R` Risk. |
| **Sub-job** | Physical grouping of a line item: `BUILDING`, `ON-SITE`, `OFFSITE`, `POOL AREA`, `POOL EQUIP`, `PUMPHOUSE`, `CARPORT`, `COMMON AREA`, `DUMPSTERS`, `OTHER SITE`, `WET UTIL`, `EV`, `FFE`. |
| **Source** | Where a number came from: comparable project (Bruner, Durango, …), `Soft Bid`, `Proposal`, `Budget`, `Allowance`, `Historical`, `N/A`. |

## Amounts (BudgetLevelLineItem)

| Term | Meaning |
|---|---|
| **TakeoffAmount** | `TakeoffQuantity × TakeoffUnitPrice × (1 + EscalationPct)`. Stored (not computed) so it can be manually overridden. |
| **BaselineAmount** | Frozen at promotion from the parent's `TakeoffAmount`. Never changes after set. Used **only** for L2-vs-L3 variance — never for calculation. |
| **ProposalAmount** | L3 bid/proposal value (dual-track). `PreferredSource` (`takeoff` \| `proposal`) decides which the L3 budget uses. |
| **Effective amount** | What the formula engine actually sums: `ProposalAmount` if L3 + preferred=proposal, else `TakeoffAmount`. |
| **Escalation** | Per-line inflation factor, stored as a decimal (`0.05` = 5%). |

## Bid leveling (L3)

| Term | Meaning |
|---|---|
| **`group_key` / GroupKey** | **The fundamental unit of bid mapping.** Multiple L2 line items can share one `group_key` (e.g. all grading → `31-MASS-GRADING`); the Master Budget shows one aggregated row per group, and the selected bid + adjustment are tracked at the group level. |
| **Trade package** | An L3 subcontractor scope (framing, concrete-slab, …). Covers one or more `group_key`s. Status `Open` \| `Proposed` \| `Awarded`. |
| **Bidder** | A subcontractor/vendor company (shared reference, not per project). |
| **Proposal** | One bid from one bidder for one trade package. `BaseBid` → `LeveledBid` (adjusted during leveling). |
| **Bid Picker** | The modal opened by clicking any **Bid Cell Button** in the budget tables; selects which bidder/amount a `group_key` uses. Amounts shown are **cost-code-aware** (sub-amount for that group, not the full trade total). |
| **Leveling** | Comparing bidders on a like-for-like scope to pick the awarded bid. |
| **Summary Sheet** | Per-trade leveling sheet: historical benchmark + scope × bidder columns + winner banner. |

## Two-track adjustments (L3)

| Term | Meaning |
|---|---|
| **Proposed adjustment** | Working draft, editable on the **Proposal L3 Budget** tab (`ProposedAdjustmentAmount`). |
| **Committed adjustment** | Approved by the estimator (`CommittedAdjustmentAmount`), shown read-only in the **Master Budget** tab. "Commit Adjustments" moves Proposed → Committed. |
| **Change threshold** | Default **$50,000**. Lines with `|adjustment| ≥ threshold` appear in the **Summary of Significant Changes** tab and require a `ChangeNote`. |

## Markups (formula engine)

| Term | Meaning |
|---|---|
| **Markup** | A percentage- or fixed-amount add-on. 7 kinds: `general_requirements` (01), `bid_risk` (BR), `contingency` (55), `bonds` (50), `insurance` (51), `overhead` (99), `fee` (98). |
| **Hard Cost** | Sum of effective amounts for divisions **not in** `{01, BR, 50, 51, 55, 98, 99}`. |
| **Markup Base** | `Hard Cost + General Requirements`. All remaining markups apply to this single base (per `formula-engine.md` — **not** the per-kind bases shown in the v94 prototype). |
| **Total Project Cost** | `Hard Cost + GR + Bid Risk + Contingency + Bonds + Insurance + Overhead + Fee`. |
| **$/Unit, $/SF** | `TotalProjectCost ÷ TotalUnits` and `÷ GrossSF`. |
| **Proposal coverage %** | How much of the L2 hard cost has been replaced by actual proposals (L3 dashboard metric). |

## Workflow & integrity

| Term | Meaning |
|---|---|
| **Status** | A BudgetLevel is `Draft` → `Submitted` → `Approved` → `Locked`, or `Rejected` (auto-creates the next sub-level). |
| **Locked** | Approved + immutable. UI disables all edits; the API returns `409` on any mutation. |
| **Approval snapshot** | Full budget state JSON captured at approval, hashed with **SHA-256**; the hash is the integrity fingerprint stored in `BudgetApprovals`. |
| **Audit log** | Append-only record of every field change (who/when/old→new). Never updated or deleted. |
| **OrgId hedge** | An `OrgId` column on root tables from day one (single Ovation tenant now) so multi-org is possible later without a rewrite. |

## Roles (⚠ naming not yet reconciled)

The product docs and API docs use **different names** for the same roles — to be confirmed
before wiring auth gating:

| Product (`overview.md` / `user-roles.md`) | API (`api/README.md`) | Likely intent |
|---|---|---|
| Estimator | `editor` | Build/edit line items, markups, bids |
| Manager | `approver` | Editor + approve/lock budgets |
| Admin | `admin` | Everything + user/project management |
| Viewer | `viewer` | Read-only |

> **Decision needed:** pick one canonical set for the frontend (and the NextAuth role
> claim mapping). See [projectOverview.md → Open Questions](./projectOverview.md#open-questions--decisions-to-confirm).
