# Roadmap

> Frontend roadmap for the Ovation Budget Platform. Scope authority is
> [`Ovation-Docs/product/roadmap-8-weeks.md`](../Ovation-Docs/product/roadmap-8-weeks.md)
> (Phase 1, confirmed 2026-06-17) and [`product/roadmap.md`](../Ovation-Docs/product/roadmap.md)
> (long-term). This file tracks the **frontend feature order** against those phases.

---

## Timeline Overview

```
NOW          Week 8         Month 3        Month 6        Year 2+
 │              │              │              │              │
 ▼              ▼              ▼              ▼              ▼
Foundation  ──► Demo       ──► Growth     ──► Production ──► Scale
2–3 projects    Owner          All active     Full team      Platform
Seed + calc     presentation   projects       features       expansion
```

**Phase 1 seed/demo projects:** West Henderson Apartments (L3, primary) · Robindale 215 (L2) · optional 3rd.

---

## Phase 1 — Foundation (Weeks 1–8) ⭐

**Goal:** A real, **Azure-SQL-backed** thin app that replaces the Excel budget
workflow for 2–3 seeded projects, presented to ownership with a polished UI that
shows the full vision. Not a production estimating platform yet.

**Success criteria (frontend side):**
- App deployed to Azure (Static Web Apps) and reachable.
- Seeded projects load with real budget data; totals match the source spreadsheet.
- Estimator can edit a line item, select an awarded bid by `group_key`, and see markups recalculate the budget proposal amount.
- Edits persist (via the API) and survive a page reload.
- A budget level can be submitted → approved → locked with a SHA-256 snapshot.
- End-to-end demo runs in < 20 minutes with clear "real vs. roadmap" labeling.

### Feature order (frontend)

> Specs live in [`context/features/`](./features/). Each is one branch + one PR.

| # | Feature | Scope | Phase-1 status |
|---|---|---|---|
| 01 | **Foundation setup** | Scaffold Next.js 16 + toolchain; builds & runs | 🟢 Core — first |
| 02 | **App shell + design system** | Sidebar/header/layout, Ovation brand tokens, hand-built Tailwind primitives | 🟢 Core |
| 03 | **Auth (demo)** | Role/user switcher (NextAuth installed; real Azure AD SSO deferred to Phase 2) | 🔵 Stretch / partial |
| 04 | **API client + types** | `lib/api/*`, `lib/types/*` (mirror backend), React Query setup, Zod validation | 🟢 Core |
| 05 | **Project list + L0–L3 navigation** | Dashboard, project shell, budget-level selector (L0/L1 read-only) | 🟢 Core |
| 06 | **Master budget table** | Divisions → line items, inline edit, live preview totals, sticky summary bar | 🟢 Core |
| 07 | **Markup panel** | 7 markup rows (pct/fixed), edit rates, server recalculates | 🟢 Core |
| 08 | **Bid leveling (L3)** | Trade packages, proposals, bid picker by `group_key`, award trade | 🟢 Core |
| 09 | **Manual overrides + audit surfacing** | Override line items with notes; show change history | 🟡 High-value |
| 10 | **Variance summary (L2 vs L3)** | By division, from `BaselineAmount` | 🟡 High-value |
| 11 | **Approval + lock** | Submit → approve/reject → lock; SHA-256 verify; signature; locked UI | 🟡 High-value |
| 12 | **Notification bell + feed** | In-app only (no email in Phase 1) | 🟡 High-value |
| 13 | **Takeoffs + benchmark views** | Site-work QTO (L2 vs L3); per-unit benchmark vs comparables | 🟡 High-value |
| 14 | **Demo polish** | Loading/empty/error states; browser-print approval view; runbook | 🔵 Stretch |

**Out of scope for Phase 1 (deferred → Phase 2):** file import / spreadsheet parsing,
real Entra ID SSO + RBAC, Excel export (ClosedXML), email notifications, real-time
conflict-resolution UI, cross-project benchmarking, mobile polish.

---

## Phase 2 — Growth (Months 3–4)

- **Real Azure AD / Entra ID auth + role-based access** (replaces the demo role switcher)
- **File import** — Excel/CSV upload with auto field mapping + manual `ColumnMapper` fallback, SignalR parse progress
- **Excel export** (ClosedXML) + **email notifications** (Azure Communication Services)
- All active Ovation projects migrated in; **L0/L1 data entry** flows
- Advanced markup overrides per division; real-time conflict detection (warning UI)
- Cross-project cost benchmarking; bulk import; mobile-responsive polish; perf at scale

## Phase 3 — Production (Months 5–6)

- Historical project archive (searchable); advanced reporting (variance, trend, division benchmarks)
- Cross-project comparison dashboard; PDF budget-snapshot + audit-report exports
- **User management UI** (admin add/remove users, change roles); project templates
- Hardening: backups, DR runbook, security audit, Azure AD group-based roles

## Year 2+ — Platform Expansion (not committed)

Subcontractor portal · accounting integration · schedule integration · Procore/Autodesk
integration · AI-assisted estimating · multi-company / white-label.

---

## Intentional Exclusions

| Not building | Reason |
|---|---|
| General document management | Use SharePoint |
| HR / payroll integration | Out of scope for budgeting |
| Tenant/resident portal | Different product (Property Mgmt) |
| Public-facing website | No external users in scope |
| Native mobile app | Responsive web covers it |

---

*Last updated: 2026-06-18*
