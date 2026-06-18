# [Feature Name]

> Copy this file to `NN-feature-name.md` (next number). Fill every section; delete the
> parenthetical hints. One feature = one branch = one PR. See
> [ai-interaction.md](../ai-interaction.md) for the workflow.

## Overview

(2–4 sentences: what this feature is, who uses it, and the user-visible outcome. Link to
the relevant phase in [roadmap.md](../roadmap.md).)

## Requirements

(Checklist of what must be true when this is done. Be specific and testable.)

- [ ] …
- [ ] …

## Data & API contract

> The .NET API owns the data. While the backend is in progress, build against the typed
> client + mock layer ([local-dev.md](../local-dev.md)).

- **Endpoints used:** (e.g. `GET /budget-levels/{id}/line-items`, `PUT /line-items/{id}`)
- **Types:** (which `lib/types/*` types this reads/writes — keep in sync with backend models)
- **Zod schemas:** (response shapes to validate at the boundary)
- **Mock source:** (which seed JSON in `../Ovation-Docs/jsons/` or `seed-package/data/` to fixture from)
- **Mutations:** (which writes; what queries to invalidate on success)
- **Edge cases:** (locked `409`, business-rule `422`, empty/loading/error states)

## UI reference

> [`ui-patterns.md`](../Ovation-Docs/docs/frontend/ui-patterns.md) is the UI/UX source of truth.

- **Views / routes:** (App Router paths touched, e.g. `projects/[projectId]/[budgetLevelId]/master`)
- **Components:** (new/changed components under `components/…`; reuse `ui/` primitives)
- **Pattern refs:** (link the exact ui-patterns sections — e.g. Bid Cell Button, Markup Controls)
- **Brand:** (apply Ovation tokens — see [coding-standards.md → Styling](../coding-standards.md#styling))

## Acceptance criteria

(How we verify it works — the demo-able behavior. e.g. "Editing a line item's unit price
updates the sticky total within one render and persists across reload.")

## Out of scope

(What this feature explicitly does NOT do — push to a later feature/phase.)

## Image
<!-- Optional design/style reference in public/assets — replace the filename. -->
![reference](/assets/REPLACE_ME.webp)
