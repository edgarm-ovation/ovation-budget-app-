# AI Interaction Guidelines

> Frontend (Next.js) project. This app talks to the separate **.NET 8 API** — there is
> **no database, ORM, or Server Actions here.** See [coding-standards.md](./coding-standards.md).

## Communication

- Be concise and direct
- Explain non-obvious decisions briefly
- Ask before large refactors or architectural changes
- Don't add features not in the project spec ([projectOverview.md](./projectOverview.md) / [roadmap.md](./roadmap.md))
- Never delete files without clarification

## Workflow

This is the common workflow we use for every feature/fix:

1. **Document** — Document the feature in [`context/features/`](./features/) (one numbered spec per feature) and point [current-features.md](./current-features.md) at it.
2. **Branch** — Create a new branch for the feature/fix.
3. **Implement** — Build the feature per its spec.
4. **Test** — Verify it works in the browser. Write/update **Vitest** unit tests for any **utilities or API-client/data logic** you added or changed and run them (`npm test`). Then run `npm run build` and fix any errors. We do **not** unit-test React components (see [Testing](#testing)).
5. **Iterate** — Adjust as needed.
6. **Commit** — Only after the build passes and everything works.
7. **Merge** — Merge to main.
8. **Verify against the API** — If the change touches the API contract (`lib/api/*` or `lib/types/*`), confirm the shape still matches the .NET API and that Zod schemas validate real responses. The backend owns the DB; there is **no `prisma`/`db push` step in this repo** — schema changes happen in the backend project.
9. **Delete Branch** — Delete the branch after merge.
10. **Review** — Review AI-generated code periodically and on demand.
11. Mark as completed in [current-features.md](./current-features.md) and add to history.

Do NOT commit without permission and until the build passes. If the build fails, fix the issues first.
**Whenever a change alters the API contract (`lib/api/*` / `lib/types/*`), flag it (step 8) and verify against the live .NET API before considering the work shipped — type drift on financial fields is a real money error.**

## Branching

Create a new branch for every feature/fix. Name it **feature/[feature]** or **fix/[fix]**, etc. Ask to delete the branch once merged.

## Commits

- Ask before committing (don't auto-commit)
- Use conventional commit messages (feat:, fix:, chore:, etc.)
- Keep commits focused (one feature/fix per commit)
- Never put "Generated With Claude" in the commit messages

## When Stuck

- If something isn't working after 2–3 attempts, stop and explain the issue
- Don't keep trying random fixes
- Ask for clarification if requirements are unclear

## Testing

We use **Vitest** for unit tests.

- **Scope:** test **utilities** (`src/lib/utils/`) and **API-client / data logic** (`src/lib/api/`, pure logic in hooks) only. We do **not** unit-test React components.
- **Priority targets:** currency/percent formatting, the markup **preview** math (must match `formula-engine.md`), the budget-level feature gate, and Zod response parsing.
- **Location:** co-locate tests next to the code as `*.test.ts` (e.g. `src/lib/utils/format-currency.test.ts`).
- **Commands:** `npm test` (watch) · `npm run test:run` (single run, use in CI / before commit).
- **Mock the API client / `fetch`** — tests stay hermetic and never hit the network or the real .NET API.
- Config lives in `vitest.config.ts` (Node environment, `@/*` alias mirrored from tsconfig, `include` scoped to `src/lib`).

## Code Changes

- Make minimal changes to accomplish the task
- Don't refactor unrelated code unless asked
- Don't add "nice to have" features
- Preserve existing patterns in the codebase (typed API client + React Query; no Server Actions)

## Code Review

Review AI-generated code periodically, especially for:

- Security (auth/role checks, no secrets in `NEXT_PUBLIC_*`, input validation)
- Performance (unnecessary re-renders, large-table virtualization, query-invalidation scope)
- Correctness of financial logic (markup math matches the server, type drift on `lib/types`)
- Patterns (matches the API-client + React Query convention; no stray `fetch`/Server Actions)
