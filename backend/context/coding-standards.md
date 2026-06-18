# Coding Standards

## TypeScript

- Strict mode enabled
- No `any` types - use proper typing or `unknown`
- Define interfaces for all props, API responses, and data models
- Use type inference where obvious, explicit types where helpful

## React

- Functional components only (no class components)
- Use hooks for state and side effects
- Keep components focused - one job per component
- Extract reusable logic into custom hooks

## Next.js

- Server components by default
- Only use `'use client'` when needed (interactivity, hooks, browser APIs)
- Use Server Actions for form submissions and simple mutations
- Use API routes when you need:
  - Webhooks (Stripe, GitHub, etc.)
  - File uploads with progress tracking
  - Long-running operations
  - Specific HTTP status codes or headers
  - Endpoints for future mobile/CLI clients
  - Third-party integrations
- Otherwise, fetch data directly in server components
- Dynamic routes for item/collection pages

### Mutations: Server Actions vs API routes (project rule)

> **Default for internal form mutations = a Server Action.** Reach for an API
> route only when one of the carve-outs below actually applies. This keeps client
> code thin (`await doThing(data)` → `{ success, data, error }`) instead of hand-rolling
> `fetch` + JSON + status-code handling, and keeps mutation logic testable (our
> Vitest setup targets `src/actions/`).

- **Use a Server Action** for: sign-up-style forms, reviews, wishlist toggles,
  admin create/update/delete, the contact form — anything a client component of
  ours submits. Put them in `src/actions/[feature].ts` with `"use server"`,
  validate input with **Zod**, and return the `{ success, data, error }` shape.
- **Use an API route** (`src/app/api/.../route.ts`) only for: webhooks (Lemon
  Squeezy), file/stream responses (e.g. `/api/download` — needs 302/403/404),
  Auth.js's own handler, or a genuinely public endpoint for a future mobile/CLI
  client.
- **Adopt going forward; do not retrofit.** Existing working endpoints
  (`/api/auth/register`, `/api/download/[fileId]`, `/api/auth/[...nextauth]`) stay
  as API routes — they hit carve-outs and rewriting them buys nothing. New
  internal mutations use Server Actions.
- **Status as of this rule:** the codebase has no `src/actions/` yet; the next
  feature with an internal mutation creates it and is the first Server Action.

## Tailwind CSS v4

**CRITICAL**: We are using Tailwind CSS v4, which uses CSS-based configuration.

- **DO NOT** create `tailwind.config.ts` or `tailwind.config.js` files (those are for v3)
- All theme configuration must be done in CSS using the `@theme` directive in `src/app/globals.css`
- Use CSS custom properties for colors, spacing, etc.
- No JavaScript-based config allowed

Example v4 configuration:

```css
@import "tailwindcss";

@theme {
  --color-primary: oklch(50% 0.2 250);
}
```

## File Organization

> Application code lives under **`src/`** (separates source from root config).
> Routes are grouped by site section; non-routable code lives in top-level
> `src/` folders. Colocate route-only helpers in `_components` / `_lib`.

- Routes: `src/app/(group)/[route]/page.tsx` — groups: `(marketing)`, `(shop)`, `(account)`, `(admin)`
- API routes: `src/app/api/[name]/route.ts` (webhooks, downloads, auth)
- Components: `src/components/[feature]/ComponentName.tsx` (shared primitives in `src/components/ui/`)
- Server Actions: `src/actions/[feature].ts`
- Types: `src/types/[feature].ts`
- Lib / integrations: `src/lib/[name].ts` — `prisma.ts`, `r2.ts`, `lemonsqueezy.ts`, `email.ts`, `env.ts`, `utils.ts`
- Auth: `auth.ts` + `auth.config.ts` at root; handler at `src/app/api/auth/[...nextauth]/route.ts`
- Database: `prisma/schema.prisma`, `prisma/seed.ts`
- Email templates: `emails/`

## Naming

- Components: PascalCase (`ItemCard.tsx`)
- Files: Match component name or kebab-case
- Functions: camelCase
- Constants: SCREAMING_SNAKE_CASE
- Types/Interfaces: PascalCase (no prefix)

## Styling

- Tailwind CSS for all styling
- Use shadcn/ui components where applicable
- No inline styles
- Dark mode first, light mode as option

## Component Content

- **No hardcoded copy in components.** Static text and asset paths (headings,
  eyebrows, body, labels, image `src`/alt, link lists) live in `constants/[feature].ts`,
  typed against `types/[feature].ts`. The component imports them and uses them as
  prop defaults, staying presentational and overridable.
- Group a section's content into one typed object per instance when the component
  is reused (e.g. `EDITION_02`, `EDITION_03` in `constants/edition.ts`); pass it
  with `<Edition {...EDITION_03} />`. Use loose named constants only for one-off
  values (see `constants/header.ts`).

## Database

- Use Prisma ORM for all database operations
- Instantiate **one** `PrismaClient` in `src/lib/prisma.ts` via `globalThis` (prevents connection storms on hot reload); never `new PrismaClient()` elsewhere
- Use Neon's **pooled** `DATABASE_URL` at runtime and the **direct** `DIRECT_URL` for migrations
- Always use `prisma migrate dev` for schema changes (not `db push`)
- Run `prisma migrate status` before committing to verify migrations are in sync
- **The Vercel build does NOT run migrations** (`prisma generate && next build` only) — so prod schema does **not** update itself on deploy. **Any commit that changes `prisma/schema.prisma` is only half-shipped until you also sync the prod Neon branch**, or the live app talks to an outdated schema and 500s when a query first uses the new field/enum (e.g. the `/shop` `TEMPLATE`-enum outage, 2026-06-12). After such a change reaches `main`, run against prod yourself:
  ```bash
  set -a; source .env.production; set +a
  npx prisma db push     # prod was bootstrapped via db push, so keep syncing it that way
  ```
- **Reference data is rows, not enums** — plugins, genres, and sound types are tables managed in the admin UI; adding one is a row, never a code change or migration
- **Webhook handlers must be idempotent** — record processed event ids (`WebhookEvent`) and no-op on duplicates

## Data Fetching

- Server components fetch directly with Prisma
- Client components use Server Actions
- Validate all inputs with Zod

## Configuration & Secrets

- **No hardcoded secrets, URLs, keys, or magic strings** — all config comes from environment variables
- Validate `process.env` once with Zod in `src/lib/env.ts`, then import the typed `env` object everywhere; never read `process.env.X` directly in feature code
- Keep secrets out of `NEXT_PUBLIC_*` vars (those reach the browser)
- `.env.example` is the committed source of truth for required vars; real keys go in `.env` (local) and the host dashboard (production)

## Auth

- **Auth.js v5**: split config — `auth.config.ts` (edge-safe, providers) + `auth.ts` (Prisma adapter + `session: { strategy: "jwt" }`)
- Route handler at `src/app/api/auth/[...nextauth]/route.ts`
- Gate `(admin)` routes on `role === "ADMIN"`; protect account routes via `auth()` / middleware
- Email/password uses a hashed `passwordHash` (bcrypt/argon2); OAuth uses the adapter `Account`

## Error Handling

- Use try/catch in Server Actions
- Return `{ success, data, error }` pattern from actions
- Display user-friendly error messages via toast

## Code Quality

- No commented-out code unless specified
- No unused imports or variables
- Keep functions under 50 lines when possible

## Testing

- **Vitest** for unit tests; run with `npm test` (watch) or `npm run test:run` (single run)
- **Test server actions and utilities only** — do NOT unit-test React components
- Co-locate tests as `*.test.ts` next to the file under test (`src/actions/`, `src/lib/`)
- Node test environment; the `@/*` path alias is configured in `vitest.config.ts`
- Mock `@/auth` and `@/lib/prisma` with `vi.mock` when testing server actions (keeps tests hermetic — no real session or DB). See `docs/testing.md`
- Prefer testing pure logic and validation/branching paths; don't reach the network or database in tests