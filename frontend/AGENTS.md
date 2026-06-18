<!-- BEGIN:nextjs-agent-rules -->
# This is NOT the Next.js you know

This project runs **Next.js 16 + React 19 (App Router only)** with **Tailwind CSS v4**. APIs, conventions, and file structure may differ from your training data — React 19 (`use`, Actions, ref-as-prop) and Tailwind v4 (CSS-first `@theme`, no `tailwind.config.ts`) in particular. Read the relevant guide in `node_modules/next/dist/docs/` before writing any code, and heed deprecation notices. There are **no Server Actions and no Prisma** in this repo.
<!-- END:nextjs-agent-rules -->

# Ovation Budget Platform — FrontEnd

Internal construction budget management for Ovation. Replaces a fragmented, Excel-based workflow with a centralized, multi-user web app that tracks budgets across four levels (L0–L3) — concept through construction documents — with bid leveling, a server-side formula engine, approval workflows, and a full audit trail.

**This repo is the FrontEnd only.** The Next.js app talks to a separate **.NET 8 API** (in `../Backend/Ovation.Api`) over HTTP. No database or ORM lives here — the API owns persistence (Azure SQL) and **all financial calculation**. The frontend shows *preview* totals; the server returns the authoritative numbers.

## Project context

Read these for the full picture before working:

- `context/projectOverview.md` — product, target users, tech stack, architecture, data model, budget levels, formula engine, foundation invariants
- `context/roadmap.md` — phased build plan (Phase 1 Foundation → Year 2+)
- `context/coding-standards.md` — TypeScript/React/Next.js rules, file org, API conventions, Zod validation, styling/brand tokens
- `context/ai-interaction.md` — per-feature workflow (document → branch → implement → test → commit → merge) and review focus
- `context/local-dev.md` — running the FE while the backend is in progress (typed API client + mock layer, env vars, priority endpoints)
- `context/current-features.md` — the active feature
- `context/features/` — one numbered spec per planned feature (build order in its README)
- `context/UI/` + `context/screenshots/` — UI references and mockups
- `../Ovation-Docs/` — source-of-truth docs (schema, formula engine, tech-stack ADRs, 8-week scope)

## Commands

> **Node is portable on this machine and not on the shell PATH** — prepend it before every `npm` command (see auto-memory: *Node portable PATH*). The app is scaffolded (Next.js 16 + React 19 + Tailwind v4); `dev`/`build`/`start`/`lint` exist now, `test`/`test:run` arrive with the Vitest setup.

- `npm run dev` — start dev server (`http://localhost:3000`)
- `npm run build` — production build (must pass before committing)
- `npm run lint` — run ESLint
- `npm test` — Vitest in watch mode *(once Vitest is installed)*
- `npm run test:run` — Vitest once (CI / pre-commit)

**Test scope:** utilities (`src/lib/utils/`) and API-client / data logic (`src/lib/api/`) only — **not** React components. Priority targets: currency/percent formatting, markup preview math (must match `formula-engine.md`), the budget-level feature gate, and Zod response parsing.

## Guardrails

- Server is the financial source of truth — never compute canonical financials in the FE.
- All server data flows through **TanStack Query** via the typed API client (`lib/api/*`); no raw `useEffect` + `fetch`, no Server Actions.
- **Zod**-validate every API response at the boundary — type drift on financial fields is a real money error.
- Approved budget levels are **locked** (SHA-256 snapshot); disable edit UI and expect `409` on mutation.
- One branch per feature/fix; ask before committing.
