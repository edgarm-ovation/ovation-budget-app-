# Local Development & API Strategy

> How to run the frontend while the **backend DB and API are still being built**.
> Strategy: build the frontend against a **typed API client with a mock layer**, then
> swap in real endpoints as they come online — one resource at a time, no UI rewrites.

---

## Current reality (2026-06-18)

- The **.NET 8 API + Azure SQL schema are in progress** — not all endpoints exist yet.
- The frontend does **not** wait for the full backend. It develops against the typed
  API client (`lib/api/*`), which can resolve to **mock data** (shaped from the seed
  JSON in `../Ovation-Docs/jsons/` and `../Ovation-Docs/seed-package/data/`) until the
  real endpoint is live.
- Because every call is **Zod-validated against the same schema** whether mocked or real,
  swapping a resource from mock → live is a config change, not a refactor.

## Prerequisites

- **Node** is portable on this machine and **not on the shell PATH** — prepend it before
  every `npm` command (see memory: *Node portable PATH*).
- Copy `.env.example` → `.env.local` and fill in:
  ```env
  NEXT_PUBLIC_API_URL=http://localhost:5001/api/v1   # real .NET API when running
  NEXT_PUBLIC_USE_MOCKS=true                          # serve mock data instead of hitting the API
  NEXT_PUBLIC_AZURE_AD_CLIENT_ID=
  NEXT_PUBLIC_AZURE_AD_TENANT_ID=
  AZURE_AD_CLIENT_SECRET=
  NEXTAUTH_SECRET=
  NEXTAUTH_URL=http://localhost:3000
  ```
- Run: `npm install` → `npm run dev` (serves on `http://localhost:3000`).

## Mock layer

- Keep one switch (`NEXT_PUBLIC_USE_MOCKS`) read through the typed `lib/env.ts`.
- Each `lib/api/*` resource module exports the same function signature regardless of
  source; when mocks are on, it returns data fixtures (seeded from the JSON files) wrapped
  in the **real response envelope** (`{ data, meta }`) and parsed by the **same Zod schema**.
- Prefer **MSW** (Mock Service Worker) if we want network-level fidelity, or a simple
  `if (env.USE_MOCKS)` branch in each resource module for speed. Pick one and standardize.
- Fixtures live alongside the resource (e.g. `lib/api/__mocks__/projects.ts`) and mirror
  the real shapes — never let a mock diverge from the Zod schema.

## Priority endpoints — build these first

This subset unblocks the **core demo loop** (open project → master budget → edit line item →
markups recalc → live totals → reload). Build the frontend against these now; trades,
approval, takeoffs, and benchmark come after.

| Order | Endpoint | Unblocks |
|---|---|---|
| 1 | `GET /reference/*` (divisions, sub-jobs, categories, sources, comparable-projects) | Static lookups, cache at app startup |
| 2 | `GET /projects` · `GET /projects/{id}` | Project list + project shell |
| 3 | `GET /projects/{id}/budget-levels` · `GET /budget-levels/{id}` | L0–L3 navigation + level shell |
| 4 | `GET /budget-levels/{id}/line-items` · `PUT /line-items/{id}` | Master budget table + inline edit |
| 5 | `GET /budget-levels/{id}/markups` · `PUT /markups/{kind}` | Markup panel |
| 6 | `GET /budget-levels/{id}/summary` | Sticky summary bar + live totals (server-computed) |

**Later (Phase-1 high-value, then stretch):** `/trades/*` + `/proposals` (bid leveling),
`/approval/*` (submit/approve/lock), `/takeoffs`, benchmark, notifications.

## API contract (must match for mock and real alike)

See [coding-standards.md → API Conventions](./coding-standards.md#api-conventions-contract).
Highlights: base `/api/v1`, **Bearer Azure AD JWT**, **amounts are USD `number`**, **dates
are `YYYY-MM-DD` strings**, success `{ data, meta }`, error `{ error: { code, message, field } }`,
list `{ data, pagination }`. Handle `409` (locked) and `422` (business rule) explicitly.

## Notes

- **Totals come from the server** (`/summary`). The frontend shows a *preview* for
  responsiveness but treats the API number as canonical — this holds for mocks too
  (the mock `/summary` should run the same preview math so the swap is seamless).
- Keep `lib/types/*` in sync with the backend models as endpoints firm up — financial
  type drift is a real money error.
