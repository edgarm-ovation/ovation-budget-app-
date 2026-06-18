# Coding Standards

> Frontend (Next.js) standards. This app talks to the **.NET 8 API** over HTTP — there
> is **no database, ORM, or Server Actions here**. Authority: the root `README.md` and
> [`Ovation-Docs/docs/frontend/`](../Ovation-Docs/docs/frontend/).

## TypeScript

- Strict mode enabled. **All files are `.ts` / `.tsx`** — no JavaScript in the frontend.
- **No `any`** — use proper typing or `unknown`. Financial type mismatches are real money errors.
- Define interfaces/types for all props, API responses, and domain models in `lib/types/*`.
- Use type inference where obvious, explicit types where they aid clarity.

## React

- Functional components only (no class components).
- Hooks for state and side effects; one job per component.
- Extract reusable data logic into custom hooks under `lib/hooks/`.
- No raw `useEffect` + `fetch` for server data — use React Query hooks (see [Data Fetching](#data-fetching)).

## Next.js (App Router)

- **App Router exclusively** — the Pages Router is not used.
- **Server Components by default.** Add `'use client'` only when you need interactivity, hooks, or browser APIs (the bid table, inline editors, dialogs, the signature canvas).
- Server Components may fetch read data on the server; interactive views hydrate and use React Query.
- **Dynamic routes** for project / budget-level / trade pages (see [File Organization](#file-organization)).
- **The only API route is the NextAuth handler** (`src/app/api/auth/[...nextauth]/route.ts`). Do **not** add other Next.js API routes or Server Actions — all data goes through the .NET API via the typed client.

## Mutations & Data Fetching

> **All server data flows through the typed API client (`lib/api/*`) + TanStack Query.**
> This replaces the Server-Actions/Prisma pattern from generic Next.js projects — we
> have neither here.

- **Reads:** React Query `useQuery` hooks in `lib/hooks/*` wrapping `lib/api/*` functions.
- **Writes:** React Query `useMutation`; invalidate the affected queries on success (e.g. invalidate `/summary` after a line-item edit so the sticky footer updates).
- **Optimistic updates** for inline edits (line items): update the cache immediately, fire `PUT /line-items/{id}`, roll back on error.
- **The API is the financial source of truth.** The frontend may show *preview* totals for responsiveness, but the authoritative numbers come back from the API (`FormulaService`). Never treat a client-computed total as canonical.
- **Validate every API response with Zod** at the boundary — fail loudly in dev, log a warning in prod. This prevents silent data corruption from contract drift.
- **Locked budget levels:** disable all edit inputs and hide add/delete in the UI; the API also returns `409` on any mutation to a `Locked` level — handle it gracefully.

## API Client

- One Axios (or fetch wrapper) instance in `lib/api/client.ts` with an auth interceptor (attaches the NextAuth session token).
- One module per resource: `projects.ts`, `budget-levels.ts`, `line-items.ts`, `markups.ts`, `trades.ts`, `budget.ts`, `approval.ts`.
- Base URL comes from `NEXT_PUBLIC_API_URL` (via the typed env object — never read `process.env` directly in feature code).
- Each function returns parsed, Zod-validated, typed data — not raw responses.

## API Conventions (contract)

> From [`Ovation-Docs/docs/api/README.md`](../Ovation-Docs/docs/api/README.md). The mock
> layer ([local-dev.md](./local-dev.md)) must honor the same contract so live ↔ mock swaps
> are seamless.

- **Base URL:** `/api/v1`. **Auth:** Bearer Azure AD JWT on every request.
- **Money is USD as a plain `number`; dates are `YYYY-MM-DD` strings.** Type and Zod-validate accordingly; never assume cents/ISO-datetime for budget amounts/dates.
- **Response envelopes:**
  - Success: `{ "data": …, "meta": { "requestId", "timestamp" } }`
  - List: `{ "data": [...], "pagination": { "page", "pageSize", "total" } }`
  - Error: `{ "error": { "code", "message", "field?", "requestId" } }`
- **Status codes to handle explicitly:** `400` validation · `401` no/invalid token · `403` insufficient role · `404` not found · `409` conflict (e.g. **budget locked**) · `422` business-rule violation · `5xx` server. Surface `409`/`422` as user-facing messages, not generic failures.
- **List query params:** `page` (default 1), `pageSize` (default 50, max 200), `sort`, `dir` (`asc`|`desc`).
- The API client unwraps `data`, validates it with Zod, and throws a typed error from the `error` envelope.

## Tailwind CSS (v4)

> **We use Tailwind v4** (CSS-first), scaffolded with Next 16 / React 19. There is **no
> `tailwind.config.ts`** — configuration and design tokens live in `@theme` inside
> `src/app/globals.css`, which starts with `@import "tailwindcss";`. PostCSS uses the
> `@tailwindcss/postcss` plugin (`postcss.config.mjs`). Do **not** reintroduce a JS
> `tailwind.config.ts` unless that decision is explicitly made.

- Tailwind classes for all styling — no external CSS except global resets in `src/app/globals.css`.
- Ovation brand tokens are defined as `@theme` custom properties in `globals.css` (e.g. `--color-brand: #536857;`), which generates the matching utilities (`bg-brand`, `text-brand`, …). See [Styling](#styling).
- Define the design-system colors as `@theme` tokens and build components against the generated utilities (`bg-brand`, `text-brand`, …) — no third-party theme layer.

## File Organization

> Application code lives under **`src/`**. Authority: `docs/frontend/folder-structure.md`.

```
src/
├── app/                              # App Router routes
│   ├── layout.tsx                    # Root layout (auth wrapper, nav)
│   ├── globals.css                   # @import "tailwindcss" + @theme brand tokens (v4)
│   ├── page.tsx                      # Redirect to /projects
│   ├── (auth)/login/page.tsx         # Azure AD sign-in
│   ├── api/auth/[...nextauth]/route.ts
│   └── projects/
│       ├── page.tsx                  # Project list / dashboard
│       ├── new/page.tsx
│       └── [projectId]/
│           ├── layout.tsx            # Project shell (sidebar, breadcrumb)
│           ├── page.tsx              # Project dashboard
│           ├── budget-levels/{page,new}
│           └── [budgetLevelId]/
│               ├── layout.tsx        # Budget-level shell (tab nav, feature gate)
│               ├── master/           # Master budget view
│               ├── proposals/        # L3 dual-track budget
│               ├── trades/[tradeId]/ # Bid leveling
│               ├── takeoffs/         # Site-work QTO (L2 vs L3)
│               ├── benchmark/        # Per-unit benchmark
│               └── approval/         # Approval + sign-off
├── components/
│   ├── budget/ trades/ takeoffs/ benchmark/ approval/ markups/   # feature components
│   ├── layout/                       # Sidebar, Header, PageLayout
│   ├── charts/                       # VarianceChart, DivisionBreakdown
│   ├── files/                        # (Phase 2) FileUploadZone, ColumnMapper
│   └── ui/                           # hand-built primitives (Button, Dialog, …), Tailwind-only
├── lib/
│   ├── api/                          # client.ts + one module per resource
│   ├── hooks/                        # React Query hooks (useProject, useLineItems, …)
│   ├── types/                        # domain types mirroring backend models
│   └── utils/                        # format-currency, format-pct, budget-level-gate, sha256
├── stores/                           # Zustand UI store(s)
└── middleware.ts                     # Azure AD auth guard for /projects routes
```

## Naming

| Thing | Convention | Example |
|---|---|---|
| Components | PascalCase | `BidLevelingTable.tsx` |
| Hooks | camelCase, `use` prefix | `useBudgetLevel.ts` |
| Utilities | camelCase | `formatCurrency.ts` |
| Types / interfaces | PascalCase, no prefix | `BudgetLevelLineItem` |
| Constants | SCREAMING_SNAKE_CASE | `MAX_FILE_SIZE_MB` |
| API route paths | kebab-case | `/api/budget-levels` |

## Styling

- **Tailwind for all styling; custom hand-built components (no component library); no inline styles.**
- **We own accessibility.** With no primitive library, interactive components (dialogs, dropdowns, the bid picker) must implement focus management/trapping, ARIA roles, and keyboard navigation ourselves.
- **Ovation brand** (apply automatically to charts/dashboards/reports):

  | Token | Hex | Use |
  |---|---|---|
  | Succulent Green | `#536857` | Primary / brand, single-series default |
  | Natural White | `#F6F5ED` | Background |
  | Designer Black | `#1A1A1A` | Text (**never** pure `#000`) |
  | Red Desert | `#D0651E` | Highlights only |
  | Clay | `#9F4B4A` | Accent |
  | Sand | `#E4B162` | Accent |
  | Clay Grey | `#5B5B5B` | Accent |
  | Lavender Purple | `#798ABF` | Accent |

  - **Multi-series charts:** Green → Sand → Clay → Red Desert → Lavender.
  - **Single-series default:** Green.
  - **Type:** TT Hoves Pro Demi Bold (headlines), TT Hoves Pro Regular (body), Calluna (serif accent). Fallback: Inter, Helvetica Neue, Arial.
- Define these as `@theme` tokens in `src/app/globals.css` (Tailwind v4 — **no `tailwind.config.ts`**); Tailwind generates the brand utilities from them.

## Domain Conventions (frontend)

- **Currency / percent formatting** lives in `lib/utils/format-currency.ts` and `format-pct.ts` (`0.05 → "5.00%"`). Never format money ad-hoc.
- **Feature gating by budget level** via `lib/utils/budget-level-gate.ts` (`canAccess(level, feature)`): `trades`/`proposals` ≥ L3; `takeoffs`/`benchmark`/`approval`/`soft-bids` ≥ L2. Layouts use it to show/hide tabs.
- **Sub-levels** display as `Level {Level}.{SubLevel}` (e.g. "Level 2.3").
- **Markup math** mirrors the server formula (preview only) — keep it identical to [`formula-engine.md`](../Ovation-Docs/docs/backend/formula-engine.md); exclude divisions `01,50,51,55,98,99,BR` from Hard Cost, then `MarkupBase = HardCost + GR` for **all** remaining markups. ⚠️ The v94 prototype (`ui-patterns.md`) shows *different per-kind exclusion sets* — that is prototype-only; do **not** copy it. The server is the source of truth.
- Keep `lib/types/*` in sync with the backend `Models` and the canonical schema — type drift on financial fields is a real money error.

## Configuration & Secrets

- **No hardcoded secrets, URLs, keys, or magic strings** — all config via environment variables.
- Validate `process.env` once with Zod in `lib/env.ts`; import the typed `env` object everywhere.
- Only browser-needed vars use `NEXT_PUBLIC_*`; secrets (e.g. `AZURE_AD_CLIENT_SECRET`, `NEXTAUTH_SECRET`) never do.
- `.env.example` is the committed source of truth. Required vars:
  ```env
  NEXT_PUBLIC_API_URL=
  NEXT_PUBLIC_AZURE_AD_CLIENT_ID=
  NEXT_PUBLIC_AZURE_AD_TENANT_ID=
  AZURE_AD_CLIENT_SECRET=        # server-side only
  NEXTAUTH_SECRET=
  NEXTAUTH_URL=
  ```

## Auth

- **NextAuth.js v5** with the **Azure AD** provider; JWT sessions; session available in Server and Client Components.
- `middleware.ts` guards `/projects/*` routes.
- **Phase 1 demo** uses a simple **role/user switcher**, not real Entra ID SSO — keep the seam so swapping in real SSO (Phase 2) is a config change, not a rewrite.
- Gate views by role (Estimator / Manager / Admin / Viewer); the API enforces authorization server-side regardless.

## Component Content

- Keep components presentational. Static labels, column definitions, and option lists live in `constants/*` (typed) rather than hardcoded in JSX, so they're reusable and overridable.
- Enum-ish reference data (divisions, sub-jobs, categories, sources) comes from the API `/reference/*` endpoints and is cached at app startup — do not hardcode it.

## Error Handling

- Wrap API calls in try/catch inside hooks; surface user-friendly messages via toast (sonner).
- Distinguish expected states (`409 Locked`, validation errors) from unexpected failures.
- React Query error + loading states render explicit UI — never a blank screen.

## Code Quality

- No commented-out code unless explicitly kept for a reason.
- No unused imports or variables.
- Keep functions/components focused; under ~50 lines where reasonable.

## Testing

- **Vitest** — `npm test` (watch) / `npm run test:run` (single run, CI / pre-commit).
- **Scope:** test **utilities** (`src/lib/utils/`) and **API-client / data logic** (`src/lib/api/`, hooks' pure logic) — especially currency/pct formatting, the markup preview math, the budget-level gate, and Zod schema parsing. **Do not unit-test React components.**
- Co-locate tests as `*.test.ts` next to the file under test.
- Node test environment; `@/*` alias configured in `vitest.config.ts`.
- Mock the API client / `fetch` — tests never hit the network or the real .NET API.
- Prefer pure logic and validation/branching paths.

## Code Review

Review periodically and on demand, especially for:

- **Security** — auth/role checks, no secrets in `NEXT_PUBLIC_*`, input validation.
- **Correctness of financial logic** — markup math matches the server, formatting, type drift.
- **Performance** — unnecessary re-renders, large-table virtualization, query invalidation scope.
- **Patterns** — matches the API-client + React Query convention; no stray Server Actions / `fetch`.
