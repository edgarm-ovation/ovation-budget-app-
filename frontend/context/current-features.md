# Current Feature

## Status

**In progress** — on `feature/foundation-setup`.

> Reality check (2026-06-18): the Next.js app **is now scaffolded** into `FrontEnd/`
> (**Next 16 + React 19 + Tailwind v4**, App Router, `src/` layout, `@/* → ./src/*`).
> `npm install`, `npm run build` (Turbopack), and `npm run dev` (HTTP 200) all pass.
> The earlier Next-14 scaffold lived in the now-deleted lowercase `frontend/` tree (an
> unrelated boilerplate). **Remaining for feature 01:** TanStack Query/Table, Zustand,
> React Hook Form + Zod, NextAuth v5, Recharts, Lucide, Prettier/Vitest, and the Ovation
> brand `@theme` tokens in `globals.css`. **No shadcn/ui** — UI primitives are hand-built
> with plain Tailwind (we own accessibility).

## Goals

Scaffold the Next.js app and install the toolchain so the project builds and runs.
Full spec: [`features/01-foundation-setup.md`](features/01-foundation-setup.md).

- ✅ Initialize npm + scaffold **Next.js 16 (App Router)** + **React 19** + **TypeScript**
  into `FrontEnd/` (`src/` layout, `@/* → ./src/*` alias) — **done**.
- Install the stack: **Tailwind v4 (✅ scaffolded)**, **TanStack Query v5 + Table v8**,
  **Zustand**, **React Hook Form + Zod**, **NextAuth v5 (Azure AD)**, **Recharts**,
  **Lucide**, plus ESLint (✅) /Prettier/Vitest. **No component library** — UI primitives
  are hand-built with plain Tailwind in `components/ui`.
- **No backend/ORM deps here** — data lives in the **.NET 8 API**; this app talks to it
  over HTTP (typed API client + React Query land in a later feature). No Prisma, no
  Server Actions.

**Done when:** `npm install` succeeds, `npm run dev` serves the app, and `npm run build`
passes clean.

## Notes

- **Authority for the stack** is the root `README.md` + [`Ovation-Docs/docs/frontend/`](../Ovation-Docs/docs/frontend/), not generic boilerplate. The previous `context/` docs described an unrelated ecommerce app and have now been replaced to match the Ovation platform.
- **Tailwind v4** (CSS-first) — scaffolded with Next 16 / React 19. No `tailwind.config.ts`; config + tokens live in `@theme` in `src/app/globals.css`.
- **No UI component library** — all primitives (Button, Dialog, DropdownMenu, the bid-picker modal, etc.) are hand-built with plain Tailwind in `components/ui`. **We own accessibility** (focus management, ARIA, keyboard nav).
- Apply **Ovation brand tokens** as `@theme` custom properties in `globals.css` (see [coding-standards.md → Styling](./coding-standards.md#styling)).
- Subsequent features are sequenced in [roadmap.md](./roadmap.md): 02 app shell + design system, 03 demo auth (role switcher), 04 API client + types, then the budget engine (master table → markups → bid leveling → approval).
- **Node** is portable on this machine and not on the shell PATH — prepend it before every `npm` command (see memory: Node portable PATH).
- **Backend DB/API is still being built.** Develop the frontend against the typed API client + a **mock layer** seeded from `Ovation-Docs/jsons/`, swapping in real endpoints one resource at a time. Priority endpoint set + mock strategy: [local-dev.md](./local-dev.md).

## Cross-Functional Flags

- **`features/02-authentication.md` is stale** — it still describes the old ecommerce auth (email/password + Google OAuth + Prisma adapter). For this project, feature 03 is a **demo role/user switcher** (NextAuth installed in 01; real Azure AD SSO is Phase 2). Rewrite that spec before implementing.

## History
<!-- Keep this updated oldest to newest -->

- Repo scaffolded as structure-only placeholders (stub files, no `package.json`/deps yet) — in the now-deleted lowercase `frontend/` tree.
- **Context docs realigned to the real project (2026-06-18):** rewrote `projectOverview.md`, `roadmap.md`, `coding-standards.md`, `ai-interaction.md`, and this file to match the **Ovation Construction Budget Management Platform** (Next.js frontend → .NET 8 API, budget levels L0–L3, bid leveling, server-side formula engine, approval + lock) per `Ovation-Docs/`. Removed all Prisma/Neon/Lemon-Squeezy/Server-Actions assumptions inherited from the prior boilerplate.
- **Next.js scaffolded (2026-06-18):** `create-next-app` into `FrontEnd/` (TS, App Router, `src/`, `@/* → ./src/*`, ESLint). **Stack bumped from the originally-specced Next 14 / React 18 / Tailwind v3 to the latest — Next 16.2.9 / React 19.2.4 / Tailwind v4** — to avoid a near-term major-version upgrade. Tailwind v4 is CSS-first: no `tailwind.config.ts`; `@import "tailwindcss"` + `@theme` in `src/app/globals.css`, `@tailwindcss/postcss` plugin. `npm install` / `npm run build` (Turbopack) / `npm run dev` all verified. ADR-001 + context docs updated to match.
- **Dropped shadcn/ui (2026-06-18):** decided **not** to use a UI component library. All primitives are hand-built with plain Tailwind in `components/ui`; accessibility (focus management, ARIA, keyboard nav) is our responsibility. ADR-001 + context docs updated; feature 02 (App shell + design system) reshaped around custom primitives. Also pinned **Recharts 3.x / Zustand 5.x** (React-19-aligned majors).

<!---->
