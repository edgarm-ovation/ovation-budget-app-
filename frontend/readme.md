AI Interaction Guidelines
Communication
Be concise and direct
Explain non-obvious decisions briefly
Ask before large refactors or architectural changes
Don't add features not in the project spec
Never delete files without clarification
Workflow
This is the common workflow that we will use for every single feature/fix:

Document - Document the feature in @context/current-feature.md.
Branch - Create new branch for feature, fix, etc
Implement - Implement the feature/fix that I create in @context/current-feature.md
Test - Verify it works in the browser. Write/update Vitest unit tests for any server actions or utilities you added or changed and run them (npm test). Then run npm run build and fix any errors. We do not unit-test React components (see Testing).
Iterate - Iterate and change things if needed
Commit - Only after build passes and everything works
Merge - Merge to main
Sync prod DB (if schema changed) - If the change touched prisma/schema.prisma, the Vercel deploy will not migrate prod. Run prisma db push against the prod Neon branch (set -a; source .env.production; set +a; npx prisma db push) so the live DB matches the deployed code — otherwise prod 500s when a query first hits the new field/enum (see Database).
Delete Branch - Delete branch after merge
Review - Review AI-generated code periodically and on demand.
Mark as completed in @context/current-feature.md and add to history
Do NOT commit without permission and until the build passes. If build fails, fix the issues first. Whenever a commit changes prisma/schema.prisma, flag the prod db push (step 8) before considering the work shipped — never assume the deploy migrates prod.

Branching
We will create a new branch for every feature/fix. Name branch feature/[feature] or fix[fix], etc. Ask to delete the branch once merged.

Commits
Ask before committing (don't auto-commit)
Use conventional commit messages (feat:, fix:, chore:, etc.)
Keep commits focused (one feature/fix per commit)
Never put "Generated With Claude" in the commit messages
When Stuck
If something isn't working after 2-3 attempts, stop and explain the issue
Don't keep trying random fixes
Ask for clarification if requirements are unclear
Testing
We use Vitest for unit tests.

Scope: test server actions (src/actions/) and utilities (src/lib/) only. We do not unit-test React components.
Location: co-locate tests next to the code as *.test.ts (e.g. src/lib/utils.test.ts, src/actions/profile.test.ts).
Commands: npm test (watch) · npm run test:run (single run, use in CI / before commit).
Server actions pull in @/auth and @/lib/prisma at import time — mock both with vi.mock so tests stay hermetic (no real session or DB). See docs/testing.md for the pattern.
Config lives in vitest.config.ts (Node environment, @/* alias mirrored from tsconfig, include scoped to src/{actions,lib}).
Code Changes
Make minimal changes to accomplish the task
Don't refactor unrelated code unless asked
Don't add "nice to have" features
Preserve existing patterns in the codebase
Code Review
Review AI-generated code periodically, especially for:

Security (auth checks, input validation)
Performance (unnecessary re-renders, N+1 queries)
Logic errors (edge cases)
Patterns (matches existing codebase?)