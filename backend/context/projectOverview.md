# Heavenly Waves — Project Overview

> **Premium patches & presets for the plugins producers actually use.**
> An ecommerce store selling sound-design packs (Omnisphere, Serum, Kontakt, TAL U‑NO‑LX, MainStage, and more) with instant secure downloads — and a future AI feature that lets a user submit a song and find the *exact* sound from the catalog.

> ⚠️ **"Heavenly Waves" is a working title** — final brand name is TBD (see [Open Questions](#open-questions--decisions-to-confirm)).

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Target Users](#target-users)
- [Core Features](#core-features)
- [Tech Stack](#tech-stack)
- [Data Model](#data-model)
- [Architecture](#architecture)
- [Monetization](#monetization)
- [Running Costs](#running-costs)
- [UI / UX Guidelines](#ui--ux-guidelines)
- [Open Questions / Decisions to Confirm](#open-questions--decisions-to-confirm)
- [Development Workflow](#development-workflow)
- [Roadmap](#roadmap)

---

## Problem Statement

Producers and performers waste hours hunting for the right preset across scattered
stores, forums, and bloated packs. And once they hear a sound they love in a track,
there's no good way to find *that exact sound*. **Heavenly Waves** is a focused store
for high-quality, plugin-specific patches with instant, secure delivery — plus an AI
"sound finder" that closes the gap between *"I want that sound"* and owning it.

## Target Users

| Persona | Key Needs |
| ------- | --------- |
| **Music producer** (bedroom → pro) | Curated, plugin-specific presets; hear before buying; instant download; lifetime re-download |
| **Live performer** (MainStage) | Reliable, well-organized patches for a live rig |
| **Sound seeker** | "Find me *this* exact sound" from a reference track (AI, Phase 4) |

## Core Features

- Browse & search catalog by **plugin, genre, sound type, price**
- **Click-to-play audio previews** (wavesurfer.js) — instant, no extra clicks
- **Cart + checkout** with cards **and PayPal** via Lemon Squeezy (tax handled)
- **Instant secure downloads** (Cloudflare R2 signed URLs) + lifetime re-download from **My Library**
- **Transactional email** — receipts + download links (Resend)
- Reviews / ratings, wishlist, discount codes *(Phase 2)*
- **Admin dashboard** to upload products & manage orders *(Phase 3)*
- **Songs / Albums section** — curated "find the sounds for this song": browse album → song → the packs that recreate it *(Phase 2)*
- **AI "find this sound"** from an uploaded song *(Phase 4)* — the automated cousin of the Songs section
- **Blog / journal** for SEO & content marketing *(Phase 3)*

## Tech Stack

| Category | Technology | Notes |
| -------- | ---------- | ----- |
| Framework | **Next.js 16** (App Router) + React 19 | Server Components by default |
| Language | **TypeScript** (strict) | — |
| Styling | **Tailwind CSS v4** + shadcn/ui | Dark-mode first; CSS `@theme` config (no JS config) |
| Animation | **Framer Motion** | Page/section transitions |
| Auth | **NextAuth v5** (Auth.js), JWT sessions | + Google OAuth, email verification, password reset |
| Database | **Neon** (PostgreSQL) + **Prisma** ORM | `pgvector` enabled in Phase 4 |
| Validation | **Zod** | Validate every input (forms, actions, webhooks) |
| Payments | **Lemon Squeezy** (Merchant of Record) | Cards + PayPal + global tax/VAT handled; **~5% + $0.50/sale** |
| File storage | **Cloudflare R2** | Product files, artwork, previews; **zero egress**; signed URLs |
| Email | **Resend** (+ React Email) | Receipts, download links, auth emails |
| Audio preview | **wavesurfer.js** | Waveform + click-to-play |
| Rate limiting | **Upstash Redis** | AI quota + download abuse *(Phase 3)* |
| Analytics | **PostHog** or Vercel Analytics | Conversion funnels *(Phase 3)* |
| Error monitoring | **Sentry** | Production *(Phase 3)* |
| AI (Phase 4) | **pgvector** + CLAP embeddings + Replicate/Modal + Inngest/Trigger.dev queue | Audio similarity + stem separation |
| Testing | **Vitest** | Server actions + utilities only (not components) |
| Hosting | **Vercel** | Auto-deploy from Git |
| CI/CD | GitHub + Vercel | `prisma migrate deploy` on release |

## Data Model

> Prisma schema **sketch** — a starting point, will evolve. `pgvector` lines stay
> commented until Phase 4.

```prisma
// ── Auth (NextAuth / Auth.js + Prisma adapter) ──────────────────────────
model User {
  id            String    @id @default(cuid())
  name          String?
  email         String    @unique
  emailVerified DateTime?
  passwordHash  String?   // email/password sign-ups; null for OAuth-only
  image         String?
  role          Role      @default(CUSTOMER)
  createdAt     DateTime  @default(now())

  accounts      Account[]
  orders        Order[]
  entitlements  Entitlement[]
  reviews       Review[]
  posts         Post[]         // Phase 3 — journal
  soundQueries  SoundQuery[]   // Phase 4
}

enum Role { CUSTOMER ADMIN }

model Account { /* NextAuth OAuth accounts: Google, etc. */ }

// ── Taxonomy (admin-managed rows — NOT hardcoded enums) ─────────────────
model Plugin {
  id       String    @id @default(cuid())
  name     String    @unique   // "Omnisphere", "Serum", "Kontakt", "TAL U-NO-LX"…
  slug     String    @unique
  products Product[]
}

model Genre {
  id       String    @id @default(cuid())
  name     String    @unique   // "Trap", "Cinematic", "House", "Lo-fi"…
  slug     String    @unique
  products Product[]           // many-to-many
}

model SoundType {
  id       String    @id @default(cuid())
  name     String    @unique   // "Bass", "Pads", "Leads", "Keys", "FX"…
  slug     String    @unique
  products Product[]           // many-to-many
}

// ── Catalog ─────────────────────────────────────────────────────────────
model Product {
  id            String        @id @default(cuid())
  slug          String        @unique
  title         String
  description   String
  priceCents    Int           // 0 = free (lead magnet)
  currency      String        @default("USD")
  pluginId      String        // exactly one plugin per product
  coverImageKey String?       // R2 key — artwork
  previewKey    String?       // R2 key — short/watermarked demo clip
  status        ProductStatus @default(DRAFT)
  featured      Boolean       @default(false)
  listedInStore Boolean       @default(true)   // false = song-exclusive (only via Songs section)
  createdAt     DateTime      @default(now())
  updatedAt     DateTime      @updatedAt

  plugin        Plugin        @relation(fields: [pluginId], references: [id])
  genres        Genre[]       // many-to-many
  soundTypes    SoundType[]   // many-to-many
  files         ProductFile[] // one or more downloadable files
  songs         Song[]        // curated songs this pack appears in (many-to-many)
  orderItems    OrderItem[]
  reviews       Review[]
  // embedding  Unsupported("vector(512)")?   // Phase 4 — pgvector
}

enum ProductStatus { DRAFT PUBLISHED ARCHIVED }

// Each deliverable in a pack: presets, bonus samples, install guide, …
model ProductFile {
  id        String   @id @default(cuid())
  productId String
  label     String              // "Presets", "Bonus samples", "Install guide"
  r2Key     String              // R2 object key
  sizeBytes BigInt?
  kind      FileKind @default(PRESET)
  sortOrder Int      @default(0)
  product   Product  @relation(fields: [productId], references: [id])
}

enum FileKind { PRESET SAMPLES GUIDE OTHER }

// ── Songs / Albums (curated "find the sounds for this song") ────────────
model Album {
  id            String        @id @default(cuid())
  slug          String        @unique
  title         String
  artist        String
  coverImageKey String?       // R2 key — album art
  year          Int?
  status        ProductStatus @default(DRAFT)
  sortOrder     Int           @default(0)
  createdAt     DateTime      @default(now())
  songs         Song[]
}

model Song {
  id          String        @id @default(cuid())
  slug        String        @unique
  title       String
  albumId     String
  durationSec Int?
  previewKey  String?        // optional reference snippet (R2 key)
  sortOrder   Int           @default(0)
  status      ProductStatus @default(DRAFT)
  createdAt   DateTime      @default(now())
  album       Album         @relation(fields: [albumId], references: [id])
  products    Product[]      // packs needed to recreate this song (many-to-many)
}

// ── Commerce ────────────────────────────────────────────────────────────
model Order {
  id             String       @id @default(cuid())
  userId         String
  lemonSqueezyId String       @unique   // external order id from webhook
  status         OrderStatus  @default(PENDING)
  totalCents     Int
  currency       String
  createdAt      DateTime     @default(now())

  user           User         @relation(fields: [userId], references: [id])
  items          OrderItem[]
  entitlements   Entitlement[]
}

enum OrderStatus { PENDING PAID REFUNDED FAILED }

model OrderItem {
  id        String  @id @default(cuid())
  orderId   String
  productId String
  priceCents Int
  order     Order   @relation(fields: [orderId], references: [id])
  product   Product @relation(fields: [productId], references: [id])
}

// What actually grants download access (lifetime, one per user/product)
model Entitlement {
  id            String   @id @default(cuid())
  userId        String
  productId     String
  orderId       String
  downloadCount Int      @default(0)
  grantedAt     DateTime @default(now())

  user          User     @relation(fields: [userId], references: [id])
  order         Order    @relation(fields: [orderId], references: [id])
  @@unique([userId, productId])
}

// Idempotency guard — duplicate webhook deliveries must not double-grant
model WebhookEvent {
  id          String   @id        // external event id (Lemon Squeezy)
  type        String
  processedAt DateTime @default(now())
}

model Review {
  id        String   @id @default(cuid())
  userId    String
  productId String
  rating    Int      // 1–5
  body      String?
  createdAt DateTime @default(now())
  user      User     @relation(fields: [userId], references: [id])
  product   Product  @relation(fields: [productId], references: [id])
}

// ── Content / Journal (Phase 3, SEO) ────────────────────────────────────
model Post {
  id            String     @id @default(cuid())
  slug          String     @unique
  title         String
  excerpt       String?
  body          String     // Markdown / MDX
  coverImageKey String?    // R2 key
  status        PostStatus @default(DRAFT)
  authorId      String?
  publishedAt   DateTime?
  createdAt     DateTime   @default(now())
  updatedAt     DateTime   @updatedAt
  author        User?      @relation(fields: [authorId], references: [id])
}

enum PostStatus { DRAFT PUBLISHED ARCHIVED }

// ── Phase 4 — AI sound finder ───────────────────────────────────────────
model SoundQuery {
  id        String   @id @default(cuid())
  userId    String
  uploadKey String   // R2 key of the submitted song
  status    String   // queued | processing | done | failed
  resultIds String[] // matched product ids, ranked
  createdAt DateTime @default(now())
  // embedding Unsupported("vector(512)")
  user      User     @relation(fields: [userId], references: [id])
}
```

## Architecture

**Purchase & delivery flow**

1. User browses catalog — **Server Components** read from Neon via Prisma.
2. Adds items to **cart** (client state) → checkout.
3. Checkout creates a **Lemon Squeezy hosted checkout**; user pays by card/PayPal; LS collects & remits tax.
4. LS fires a **webhook** → `POST /api/webhooks/lemonsqueezy` → verify signature → (idempotent via `WebhookEvent`) create `Order` + `Entitlement`(s).
5. **Resend** sends receipt + link to **My Library**.
6. **Download:** user clicks → Server Action checks `Entitlement` → mints a **short-TTL R2 signed URL** → file downloads (links can't be shared).

**AI flow (Phase 4)**

Upload song → R2 → enqueue job (Inngest/Trigger.dev) → Replicate generates a **CLAP
embedding** (and optional Demucs stem separation) → **pgvector** similarity search
against `Product` embeddings → return ranked matches. One submission per user (Upstash quota).

**Key conventions**

- Server Components fetch directly with Prisma; Client Components mutate via **Server Actions**.
- **API routes only** for webhooks (Lemon Squeezy), file/download streaming, and AI job callbacks.
- **Zod** validates every input — forms, actions, and webhook payloads.

## Monetization

- **One-time sales only** of preset packs via **Lemon Squeezy** (Merchant of Record). **No subscriptions / memberships.**
- Fee: **~5% + $0.50 per sale**; in exchange, LS handles cards + PayPal + global tax/VAT and most of the checkout code.
- Digital goods → **no inventory limits** (unlimited stock).
- The `pricing-plan` screen in the references is a **landing-page pricing/feature section** showcasing packs — not a recurring tier.
- **Free packs** (`priceCents = 0`) are supported as lead magnets — granted on sign-in without going through paid checkout.

## Running Costs

> Approximate, 2026. Two buckets: **fixed infrastructure** + **per-sale fees**.

| Service | Free tier covers | Paid when… |
| ------- | ---------------- | ---------- |
| Vercel | All of building/testing | ~$20/mo (Pro) once commercial / higher traffic |
| Neon | 0.5 GB (fine for MVP) | ~$19/mo when outgrown |
| Cloudflare R2 | 10 GB + **zero egress** | ~$0.015/GB after (100 GB ≈ $1.50/mo) |
| Resend | 3,000 emails/mo | ~$20/mo past 3k |
| Domain | — | ~$1/mo (≈$12/yr) |
| NextAuth · Prisma · Zod · wavesurfer | Free (OSS) | $0 |

**Totals:** Building (no sales) **~$0–1/mo** · Just launched **~$20/mo** · Growing store **~$40–60/mo** — plus **~5% per sale** (Lemon Squeezy).
**AI phase** adds only **~$0.01–0.05 per song** processed (pay-per-use), trivial until real scale.

## UI / UX Guidelines

- **Dark-mode first**, light mode optional. Theme built from the references in `context/screenshots/`.
- **Framer Motion** for page/section transitions — smooth, not flashy.
- **Audio previews are one click** and play immediately (no separate "load" step).
- Fully **responsive**; accessible (keyboard + screen-reader friendly players and controls).
- shadcn/ui for components; no inline styles; Tailwind v4 `@theme` tokens in `app/globals.css`.

## Open Questions / Decisions to Confirm

1. **Brand name** — keep "Heavenly Waves" or pick a final name?
2. **Product license / EULA** — what rights do buyers get? Royalty-free? Resale allowed? Needs defined terms.
3. **Refund policy** — Lemon Squeezy requires one; digital-goods refund stance? (e.g. no refunds once downloaded.)
4. **Install/usage guides** — per-product instructions for installing into Omnisphere/Kontakt/etc.? (Cuts support load.)
5. **Free samples / lead magnet** — offer a free pack to build the email list?

> **Resolved:** No subscriptions — one-time digital sales only. · Single-seller to start (marketplace is Phase 5). · Blog/journal in scope for SEO (Phase 3).

## Development Workflow

See `context/ai-interaction.md` for the per-feature workflow (document → branch →
implement → test → commit → merge). One branch per feature/fix.

## Roadmap

Phased plan with detailed, checkable tasks lives in **`context/roadmap.md`**. Summary:

| Phase | Focus | Outcome |
| ----- | ----- | ------- |
| **0** | Foundation (Prisma, Neon, auth, theme) | Plumbing in place |
| **1** ⭐ | MVP store: catalog, preview, checkout, secure download, email | **Revenue** |
| **2** | Shopping experience: cart, search/filter, **Songs/Albums section**, reviews, motion | Higher conversion |
| **3** | Run & grow: admin dashboard, SEO, analytics, rate limiting | Scale ops + traffic |
| **4** 🎯 | AI "find this sound" | The differentiator |
| **5** | Marketplace, multi-currency, affiliates | Grow the model |

---

*Last updated: 2026-06-04*