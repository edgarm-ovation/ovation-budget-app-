# Ovation Budget Platform — Project Overview

> **Internal construction budget management for Ovation.**
> Replaces a fragmented, Excel-based budget workflow with a centralized, multi-user
> web app that tracks construction budgets across four levels (L0–L3) — from
> pre-schematic concept through final construction documents — with bid leveling,
> a server-side formula engine, approval workflows, and a full audit trail.

> **This repo is the FrontEnd only.** The Next.js app talks to a separate **.NET 8
> API** (in `../Backend/Ovation.Api`) over HTTP. No database or ORM lives here —
> the API owns persistence (Azure SQL) and all financial calculation.

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Target Users](#target-users)
- [Core Features](#core-features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Data Model](#data-model)
- [Budget Levels (L0–L3)](#budget-levels-l0l3)
- [Markup / Formula Engine](#markup--formula-engine)
- [Foundation Invariants](#foundation-invariants)
- [UI / UX Guidelines](#ui--ux-guidelines)
- [Open Questions / Decisions to Confirm](#open-questions--decisions-to-confirm)
- [Development Workflow](#development-workflow)
- [Roadmap](#roadmap)
- [Source of Truth](#source-of-truth)

---

## Problem Statement

Before this platform, Ovation's budget process lived in per-project Excel workbooks:

- Multiple people edited the same file → version conflicts.
- No audit trail — no record of who changed what, when.
- Moving between budget levels meant manually copying data between files.
- Subcontractor files had inconsistent column structures — mapping was manual and error-prone.
- Leadership had no real-time visibility into budget status across projects.
- Bid leveling (comparing subcontractor proposals) was done by hand in Excel.

The platform makes the budget structured, multi-user, versioned, audited, and live —
with the server as the financial source of truth.

## Target Users

| Role | Division | What they do in the platform |
|---|---|---|
| **Estimator** | Development / OCI | Build & maintain budget line items, select bids, submit levels for approval |
| **Manager** | Development / OCI | Review and approve/reject budget levels; oversee project budgets |
| **Admin** | Engineering | Manage users, roles, and project setup |
| **Viewer** | Asset Mgmt / Finance | Read-only access to budget data for reporting |

> Subcontractors do **not** log in — their files are uploaded by Ovation staff (Phase 2+).
> Full permission matrix lives in the backend `auth.md` / `user-roles.md` (still a stub in docs).

## Core Features

- **Budget level management** — L0–L3 per project, each preserved as a versioned snapshot (sub-levels L2.1, L2.2…); levels are never overwritten.
- **Master budget table** — all divisions → line items, inline editing, live totals.
- **Bid leveling** — compare subcontractor proposals per trade; award a bid by `group_key`; track proposal/bid/awarded status.
- **Formula engine** — markups (general requirements, bid risk, contingency, bonds, insurance, overhead, fee) computed **server-side**; overrides per line item; totals update live.
- **Approval workflow** — submit → approve/reject with comments → approved levels **lock** with a SHA-256 snapshot; no further edits without a new sub-level.
- **Variance summary** — L2 vs L3 by division (uses frozen `BaselineAmount`).
- **Benchmark** — per-unit / per-SF cost vs. comparable Ovation projects.
- **Audit trail** — every field change logged (who / when / before-value), append-only.
- **Notifications** — in-app bell for submit / approve / reject (email is Phase 2).
- **File import** *(Phase 2)* — Excel/CSV upload with auto column mapping + manual fallback.
- **Excel export** *(Phase 2)* — ClosedXML; demo uses browser-print of the approval view.

## Tech Stack

> Frontend only. See [coding-standards.md](./coding-standards.md) for the rules and
> [`Ovation-Docs/docs/frontend/tech-stack.md`](../Ovation-Docs/docs/frontend/tech-stack.md)
> for the decision record (ADR-001).

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| Framework | **Next.js** (App Router only) | 16 | Routing, layouts, server components |
| UI runtime | **React** | 19 | Component model (`use`, Actions, ref-as-prop) |
| Language | **TypeScript** (strict) | 5.x | Type safety; financial type mismatch = real money error |
| UI components | **Custom (plain Tailwind)** in `components/ui` | — | Hand-built primitives; no component library (we own a11y) |
| Styling | **Tailwind CSS** | 4.x (CSS-first `@theme`) | Utility-first; Ovation brand tokens in `globals.css` |
| Data tables | **TanStack Table** | v8 | Bid leveling table — the core UI |
| Charts | **Recharts** | 3.x | Variance / benchmark charts |
| Server state | **TanStack Query** (React Query) | v5 | All API data: caching, refetch, optimistic updates |
| Client state | **Zustand** | 5.x | UI state only (sidebar, filters, column visibility) |
| Auth | **NextAuth.js** | 5.x | Azure AD SSO (full wiring is a later feature) |
| Forms | **React Hook Form** | 7.x | Form state |
| Validation | **Zod** | 3.x | Validate every API response at the boundary + every form |
| Icons | **Lucide React** | latest | Icon set |
| Testing | **Vitest** | latest | Utilities + API client logic (not components) |

**Backend (separate repo/folder, for context):** ASP.NET Core .NET 8 (Minimal APIs),
C# 12, EF Core 8 (Azure SQL), SignalR, ClosedXML/CsvHelper, FluentValidation, Serilog.

## Architecture

```
Browser (Next.js 16 App Router)
   │  Server Components fetch + Client Components mutate
   │  via typed API client (lib/api/*) → TanStack Query
   ▼
.NET 8 API (Ovation.Api)  ──►  Azure SQL (EF Core 8)
   • FormulaService  = canonical totals (server is source of truth)
   • Services hold all business logic (never in endpoints or the frontend)
   • SignalR (BudgetHub) for live updates / parse progress
   • Azure AD auth, append-only AuditLog, SHA-256 approval snapshots
```

**Key conventions**

- Frontend **never** computes canonical financials — it shows **preview** totals; the API returns the authoritative numbers on save.
- All server data flows through **TanStack Query** via a typed **API client** (`lib/api/*`) — no raw `useEffect` + `fetch`.
- **Zod** validates every API response at the boundary; mismatches fail loudly in dev.
- The only Next.js **API route** is the NextAuth handler (`/api/auth/[...nextauth]`). There are **no Server Actions and no Prisma** in this project.
- Locked budget levels disable all edit UI; the API also enforces it (409 on mutation).

## Data Model

> The **API owns the schema**. Frontend types in `lib/types/*` **mirror** the backend
> models — keep them in sync. Canonical schema:
> [`Ovation-Docs/docs/database/schema.md`](../Ovation-Docs/docs/database/schema.md)
> (Schema A, per ADR-005 — Schema B is retired).

Core entities the frontend consumes:

| Entity | Purpose |
|---|---|
| `Project` (+ `UnitMix`, `Parking`) | Construction project + unit/building metadata |
| `BudgetLevel` | L0–L3 snapshot per project; sub-levels (L2.1…); `Status` Draft/Submitted/Approved/Locked/Rejected |
| `Division` | CSI division master list (seeded) — the organizing spine |
| `LineItem` | Seeded master line-item library per division |
| `BudgetLevelLineItem` | **The core data row** — actual budget data per line item per level (qty, unit price, takeoff, baseline, proposal, group_key) |
| `Markup` | 7 markup rows per budget level (pct or fixed) |
| `TradePackage` / `Bidder` / `Proposal` / `ProposalLineItem` | L3 bid leveling |
| `Takeoff` | Site-work quantity takeoffs, dual-track L2 vs L3 |
| `BudgetApproval` | Immutable signed approval + SHA-256 fingerprint + snapshot JSON |
| `ComparableProject` / `*Costs` / `TradeBenchmark` | Benchmark comparisons |
| `AuditLog` | Append-only field-change history |
| `Notification` / `UploadedFile` | In-app bell + (Phase 2) file import |

## Budget Levels (L0–L3)

| Level | Stage | Notes |
|---|---|---|
| **L0** | Pre-schematic / concept | Read-only display in Phase 1 |
| **L1** | Schematic | Read-only display in Phase 1 |
| **L2** | Design development | Benchmark, takeoffs, approval enabled |
| **L3** | Construction documents | Adds trades + proposals (full bid leveling) |

- Each level can have multiple **sub-level revisions** (L2.1, L2.2…); a sub-level is always a copy of the prior one.
- Only one sub-level per project+level can be `Draft` or `Submitted` at a time.
- **Promotion** to the next major level copies all line items and freezes `BaselineAmount` from the parent's `TakeoffAmount`.
- **Feature gating** (`lib/utils/budget-level-gate.ts`): `trades`/`proposals` ≥ L3; `takeoffs`/`benchmark`/`approval`/`soft-bids` ≥ L2.

## Markup / Formula Engine

> Computed **server-side** (`FormulaService`). The frontend shows preview math only.
> Full rules: [`Ovation-Docs/docs/backend/formula-engine.md`](../Ovation-Docs/docs/backend/formula-engine.md).

Seven markup rows per budget level (default rates):

| Kind | Label | Code | Mode | Default |
|---|---|---|---|---|
| `general_requirements` | General Requirements | 01 | pct | 6.00% |
| `bid_risk` | Bid Risk | BR | pct | 2.00% |
| `contingency` | Construction Contingency | 55 | pct | 5.00% |
| `bonds` | Sub Bonds | 50 | pct | 1.10% |
| `insurance` | GL Insurance | 51 | fixed | project-specific |
| `overhead` | Overhead | 99 | pct | 2.00% |
| `fee` | Contractor Fee | 98 | pct | 6.00% |

```
Hard Cost  = SUM(line items where Division NOT IN [01,50,51,55,98,99,BR])
General Requirements (01) = Hard Cost × GR rate
Markup Base = Hard Cost + General Requirements
Bid Risk / Contingency / Bonds / Overhead / Fee = Markup Base × rate (or FixedAmount)
GL Insurance = FixedAmount (or Markup Base × rate)
Total = Hard Cost + GR + Bid Risk + Contingency + Bonds + Insurance + Overhead + Fee
```

## Foundation Invariants

These let the demo codebase grow to the full platform without a rewrite:

1. **Server is the financial source of truth** — frontend previews only.
2. **Approved budgets are immutable** — SHA-256 snapshot + lock; changes start a new sub-level.
3. **Audit is append-only.**
4. **`OrgId` on root tables from day one** (single Ovation tenant for now).
5. **Business logic lives in Services**, never in endpoints or the frontend.
6. **Build against Schema A only.**

## UI / UX Guidelines

> **UI/UX source of truth:** [`Ovation-Docs/docs/frontend/ui-patterns.md`](../Ovation-Docs/docs/frontend/ui-patterns.md)
> — extracted from the West Henderson L3 prototype (v94). It defines the layout, the exact
> view list, the **Bid Cell Button → Bid Picker** modal, the Markup Controls panel, the
> Summary Sheet, takeoff layouts, the approval tab, and the local **state shape** (your
> Zustand store contract). Build components against it. Domain terms: [glossary.md](./glossary.md).

- **Ovation brand** (apply the brand tokens; see [coding-standards.md → Styling](./coding-standards.md#styling)):
  Succulent Green `#536857` (primary), Natural White `#F6F5ED` (bg), Designer Black `#1A1A1A`
  (text — never pure black), Red Desert `#D0651E` (highlights only). Multi-series charts:
  Green → Sand → Clay → Red Desert → Lavender.
- **Persistent shell** — sidebar (project nav) + header (notifications, user) + main content; maps to the App Router layout system.
- The **bid leveling table** is the core UI: expandable division → trade → line-item hierarchy, inline editing, virtual scroll at scale (TanStack Table).
- **Sticky budget summary footer** (hard cost, total, $/unit, $/SF) that auto-updates from the React Query cache.
- **Locked state** disables all edit inputs and shows a "Locked" banner.
- Custom components (no library); Tailwind classes only (no inline styles); **we own accessibility** (focus management, ARIA, keyboard nav) + responsive.

## Open Questions / Decisions to Confirm

1. **Tailwind version** — ✅ *Resolved (2026-06-18):* scaffolded on **Tailwind v4** (CSS-first `@theme`, no `tailwind.config.ts`) alongside **Next 16 / React 19** — chosen to avoid a near-term major-version upgrade. (The original v3 pin was tied to shadcn/Next-14; we no longer use shadcn, so it's moot.)
2. **Division 29 vs 31** — use **Division 31 (Earthwork)**; do not seed Div 29 (legacy label). *(Resolved in schema.)*
3. **Zustand store location** — `stores/uiStore.ts` (tech-stack) vs under `lib/` — pick one and standardize.
4. **Auth in the 8-week demo** — confirmed a **role/user switcher**, not real Entra ID SSO (SSO is Phase 2).
5. **Seed data spec** — `seed-data.md` is still a stub and is the Week-1 critical path for the backend.
6. **Role names conflict** — product docs say **Estimator / Manager / Admin / Viewer**; the API says **viewer / editor / approver / admin**. Pick one canonical set for frontend gating + the NextAuth role claim mapping (likely Estimator=editor, Manager=approver). See [glossary.md → Roles](./glossary.md#roles--naming-not-yet-reconciled).
7. **Markup base: prototype vs. engine** — the v94 prototype uses different exclusion sets per markup kind; the canonical `formula-engine.md` uses a single `MarkupBase = HardCost + GR`. Frontend preview math follows the **engine**, not the prototype.

> **Resolved:** Real Azure SQL persistence (not browser-only) · seeded data, no file import in Phase 1 · server-side calculation · export/email deferred to Phase 2 · **no UI component library — primitives are hand-built with plain Tailwind (we own accessibility)**.

## Development Workflow

See [ai-interaction.md](./ai-interaction.md) for the per-feature workflow
(document → branch → implement → test → commit → merge). One branch per feature/fix.
Feature specs live in [`context/features/`](./features/).

## Roadmap

Full phased plan: [roadmap.md](./roadmap.md). Summary:

| Phase | Window | Focus | Outcome |
|---|---|---|---|
| **1 — Foundation** ⭐ | Weeks 1–8 | Real, Azure-SQL-backed thin app for 2–3 seeded projects; replace the Excel workflow | **Owner demo** |
| **2 — Growth** | Months 3–4 | Azure AD SSO, file import, Excel export, email, all active projects, L0/L1 entry | Full team onboarded |
| **3 — Production** | Months 5–6 | Historical archive, reporting, user management UI, hardening | Daily use, all divisions |
| **Year 2+** | — | Subcontractor portal, accounting/schedule/Procore integration, AI-assisted estimating | Platform expansion |

**Phase 1 seed/demo projects:** West Henderson Apartments (L3, primary) · Robindale 215 (L2).

## Source of Truth

When docs disagree, these win (from `Ovation-Docs/README.md`):

| Topic | Authority |
|---|---|
| Data model / schema | [`docs/database/schema.md`](../Ovation-Docs/docs/database/schema.md) (Schema A, ADR-005) |
| 8-week build scope | [`product/roadmap-8-weeks.md`](../Ovation-Docs/product/roadmap-8-weeks.md) |
| Long-term architecture | [`docs/architecture/target-architecture.md`](../Ovation-Docs/docs/architecture/target-architecture.md) |
| Frontend stack | [`docs/frontend/tech-stack.md`](../Ovation-Docs/docs/frontend/tech-stack.md) (ADR-001) |
| Frontend folder structure | [`docs/frontend/folder-structure.md`](../Ovation-Docs/docs/frontend/folder-structure.md) |
| UI / UX patterns | [`docs/frontend/ui-patterns.md`](../Ovation-Docs/docs/frontend/ui-patterns.md) (West Henderson v94 prototype) |
| API contract | [`docs/api/README.md`](../Ovation-Docs/docs/api/README.md) |
| Calculation rules | [`docs/backend/formula-engine.md`](../Ovation-Docs/docs/backend/formula-engine.md) |

---

*Last updated: 2026-06-18*
