# Ovation Budget Platform

Application scaffold for the Ovation construction budget platform. Folder structure
is derived directly from `ovation-budget-docs/ovation-platform-docs/docs`.

- **Frontend authority:** `docs/frontend/folder-structure.md`
- **Backend authority:** `docs/backend/tech-stack.md`

All files in this scaffold are **placeholders** — folders and stub files only, no
business logic or dependencies yet.

---

## Stack

| Side | Technology |
|---|---|
| Frontend | Next.js 14 (App Router) · TypeScript · Tailwind + shadcn/ui · TanStack Query/Table · Zustand · React Hook Form + Zod · NextAuth (Azure AD) |
| Backend | ASP.NET Core .NET 8 (Minimal APIs) · C# 12 · EF Core 8 (Azure SQL) · SignalR · ClosedXML/CsvHelper · FluentValidation · Serilog |

---

## Layout

```
ovation-budget-app-/
├── frontend/                         # Next.js 14 app
│   ├── public/                       # Static assets (logos, favicon)
│   └── src/
│       ├── app/                      # App Router routes
│       │   ├── (auth)/login/         # Azure AD sign-in
│       │   └── projects/[projectId]/[budgetLevelId]/
│       │       ├── master/ proposals/ trades/[tradeId]/
│       │       └── takeoffs/ benchmark/ approval/
│       ├── components/               # budget, trades, takeoffs, benchmark,
│       │                             # approval, markups, ui, layout, charts, files
│       ├── lib/
│       │   ├── api/                  # Typed API client per resource
│       │   ├── hooks/                # React Query hooks
│       │   ├── types/                # Domain types (mirror backend models)
│       │   └── utils/                # currency/pct format, budget-level gate, sha256
│       ├── styles/                   # Tailwind globals
│       └── middleware.ts             # Azure AD auth guard
│
└── backend/
    └── Ovation.Api/                  # .NET 8 Minimal API
        ├── Program.cs                # bootstrap, middleware, DI
        ├── Endpoints/                # Minimal API endpoint definitions
        ├── Services/                 # Business logic (parsing, formula, approval, audit)
        ├── BackgroundJobs/           # IHostedService file processing
        ├── Hubs/                     # SignalR (BudgetHub)
        ├── Models/                   # EF Core entities
        ├── DTOs/{Requests,Responses}/
        ├── Validators/               # FluentValidation rules
        ├── Data/                     # DbContext, Migrations, SeedData
        └── Middleware/               # Auth, Role, ErrorHandling
```

---

## Next steps

This is structure only. To make it buildable:

1. **Frontend** — `npx create-next-app@latest` config into `frontend/`, then add
   `package.json`, `tsconfig.json`, `tailwind.config.ts`, `.env.example`, and
   `npx shadcn@latest init` to populate `components/ui`.
2. **Backend** — `dotnet new web -n Ovation.Api`, add the NuGet packages from
   `docs/backend/tech-stack.md`, and wire up `Program.cs`.
3. Keep the frontend `lib/types` in sync with backend `Models` — financial type
   mismatches are real money errors (see `docs/frontend/tech-stack.md`).
