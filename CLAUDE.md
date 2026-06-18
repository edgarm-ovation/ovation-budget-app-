# Ovation Budget Platform — Monorepo Root

Two apps that ship together to replace Ovation's Excel-based construction budget workflow:

- **`FrontEnd/`** — Next.js 14 (App Router) web app. All rules, context, and commands
  live in `FrontEnd/CLAUDE.md` (it imports the layered `FrontEnd/context/*` docs).
- **`Backend/`** — ASP.NET Core **.NET 8** API on Azure SQL. Context in `Backend/context/`.

The frontend renders **preview** totals only — the **.NET API is the financial source of
truth** and returns the authoritative numbers on save. Never treat a client-computed total
as canonical.

## Where to work (read before coding)

| Task is in… | Read |
|---|---|
| Frontend | `FrontEnd/CLAUDE.md` → pulls in `FrontEnd/context/{projectOverview,coding-standards,ai-interaction,local-dev,current-features}.md` |
| Backend | `Backend/context/{projectOverview,coding-standards,ai-interaction,current-features}.md` |
| Schema / formula engine / ADRs (source of truth) | `FrontEnd/Ovation-Docs/` |

This root file only **routes** you — it does not duplicate the app-level rules. When the
docs disagree, the authority table in `FrontEnd/context/projectOverview.md` → *Source of
Truth* wins.

## Machine note

Node is a **portable install not on the system PATH** (see auto-memory *Node portable
PATH*). The repo-local `.claude/settings.local.json` prepends it for tool shells; if a
command still reports `node`/`npm` not found, prepend it manually:

```powershell
$env:Path = "$env:LOCALAPPDATA\Programs\node-v20.20.2-win-x64;$env:Path"
```

Run `npm` from inside `FrontEnd/`. One branch per feature/fix; **ask before committing.**
