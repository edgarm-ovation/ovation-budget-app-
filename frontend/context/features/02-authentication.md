# Authentication (Demo) — Role/User Switcher

> Supersedes the previous ecommerce auth spec (email/password + Google + Prisma), which
> did not apply to this project. Phase-1 auth is a **demo role/user switcher**; real
> **Azure AD / Entra ID SSO is Phase 2**. See [roadmap.md](../roadmap.md).

## Overview

Provide enough auth for the 8-week demo: a lightweight **user/role switcher** so the app
can demonstrate role-gated behavior (Estimator vs Manager vs Viewer) without standing up
real Entra ID SSO. NextAuth is installed and wired with the **Azure AD provider seam** so
swapping in real SSO later is a config change, not a rewrite.

## Requirements

- [ ] **NextAuth.js v5** configured with the Azure AD provider seam (`auth.config.ts` edge-safe + `auth.ts`), JWT session strategy — but a **demo credential/switcher path** active for Phase 1.
- [ ] Route handler at `src/app/api/auth/[...nextauth]/route.ts` (the **only** Next.js API route in the app).
- [ ] A **role/user switcher** (header menu or `(auth)/login`) to pick a seeded demo user + role; selection persists in the session.
- [ ] **Canonical role set** applied to gating — *resolve the naming conflict first* (Estimator/Manager/Admin/Viewer vs editor/approver/admin/viewer; see [glossary.md → Roles](../glossary.md#roles--naming-not-yet-reconciled)).
- [ ] `middleware.ts` guards `/projects/*`; unauthenticated users redirect to `(auth)/login`.
- [ ] Session (user + role) available in Server and Client Components; reflected in the header.
- [ ] A `requireRole(role)` helper for gating views/actions; the API still enforces authorization server-side regardless.
- [ ] Env vars present (`NEXT_PUBLIC_AZURE_AD_CLIENT_ID`, `AZURE_AD_CLIENT_SECRET`, `NEXTAUTH_SECRET`, `NEXTAUTH_URL`) via the typed `lib/env.ts`, even if SSO is dormant in the demo.

## Data & API contract

- **Endpoints:** NextAuth handler only. Demo users may be a static seed list (or `GET /users` if the API exposes it). No password storage — there is **no Prisma/DB in this repo**.
- **Types:** `lib/types/user.ts` — `User`, `Role`. Mirror the backend `Users`/`Roles` model.
- **Later (Phase 2):** real Azure AD tokens carry the role claim; map claim → app role here.

## UI reference

- **Views:** `(auth)/login/page.tsx` (or a header switcher); header user menu.
- **Pattern refs:** sidebar/header layout in [`ui-patterns.md`](../Ovation-Docs/docs/frontend/ui-patterns.md). Apply Ovation brand tokens.

## Acceptance criteria

- Selecting a demo user/role updates the session; the header shows the current user/role.
- A Viewer sees read-only UI (no edit/add controls); an Estimator/Editor can edit; a Manager/Approver sees approve actions.
- Visiting `/projects/*` while signed out redirects to login.
- Swapping `NEXT_PUBLIC_USE_MOCKS` / enabling real Azure AD does not require component changes.

## Out of scope

- Real Entra ID SSO, group-based role assignment (Phase 2/3).
- Email/password accounts, OAuth providers other than Azure AD, email verification, password reset (not part of this product).

## Image
<!-- Auth/switcher screens design in public/assets — replace the filename. -->
![auth screens](/assets/REPLACE_ME.webp)
