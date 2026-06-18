# Ovation Budget Platform — FrontEnd

Internal construction budget management for Ovation: a centralized, multi-user web app that tracks construction budgets across four levels (L0–L3), with bid leveling, a server-side formula engine, approval workflows, and a full audit trail. Replaces the old per-project Excel workbooks.

**FrontEnd only.** This Next.js 16 (App Router) app talks to a separate **.NET 8 API** (`../Backend/Ovation.Api`) over HTTP. No database, ORM, Prisma, or Server Actions live here — the API owns persistence (Azure SQL) and **all financial calculation**. The frontend renders *preview* totals; the server returns the authoritative numbers on save.

## Context Files

Read the following to get the full context of the project:

- @context/projectOverview.md
- @context/roadmap.md
- @context/coding-standards.md
- @context/ai-interaction.md
- @context/local-dev.md
- @context/current-features.md

Feature specs live in @context/features/ (one numbered spec each). Source-of-truth docs (schema, formula engine, tech-stack ADRs) are in `../Ovation-Docs/`.

## Commands

> **Node is portable on this machine and not on the shell PATH** — prepend it before every `npm` command (see auto-memory: *Node portable PATH*). The app is scaffolded (Next.js 16 + React 19 + Tailwind v4, App Router, `src/`, `@/* → ./src/*`). `dev`/`build`/`start`/`lint` exist now; `test`/`test:run` arrive with the Vitest setup.

- `npm run dev` — Start development server (`http://localhost:3000`)
- `npm run build` — Production build (must pass before committing)
- `npm run start` — Start production server
- `npm run lint` — Run ESLint
- `npm test` — Run Vitest unit tests in watch mode *(once Vitest is installed)*
- `npm run test:run` — Run Vitest once (server/data logic + utilities only; not components)

## Key conventions

- **Server is the financial source of truth** — never compute canonical financials in the FE; show previews only.
- All server data flows through **TanStack Query** via the typed API client (`lib/api/*`) — no raw `useEffect` + `fetch`, no Server Actions.
- **Zod**-validate every API response at the boundary; mismatches fail loudly in dev. Type drift on financial fields is a real money error.
- Approved budget levels are **locked** (SHA-256 snapshot) — disable all edit UI and expect `409` on mutation; changes start a new sub-level.
- Apply Ovation brand tokens (Succulent Green `#536857`, Natural White `#F6F5ED` bg, Designer Black `#1A1A1A` text, Red Desert `#D0651E` highlights). See coding-standards → Styling.
- One branch per feature/fix (`feature/[name]` or `fix/[name]`); ask before committing.

> **No hosted DB via MCP here.** The .NET API owns Azure SQL — there is no database MCP server, no Prisma, and no `db push` step in this repo. Schema changes happen in the backend project.
