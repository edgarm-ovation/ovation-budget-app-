# Current Feature

## Status

In Progress — `feature/checkout-lemonsqueezy`

## Goals

Wire the real Lemon Squeezy payment flow (Phase 1 ⭐ — revenue):

- **Checkout hand-off** — the `/checkout` "Continue to payment" button creates a
  Lemon Squeezy hosted checkout (server action) and redirects the buyer there.
  Cards + PayPal + tax handled by LS (Merchant of Record).
- **Webhook** — `POST /api/webhooks/lemonsqueezy` verifies the HMAC signature,
  is idempotent (`WebhookEvent`), and on `order_created` creates the `Order` +
  `OrderItem`s + `Entitlement`s (unlocking My Library downloads); on
  `order_refunded` marks the order `REFUNDED` and revokes its entitlements.
- **Buyer return** — LS redirects back to `/account/library?purchase=success`;
  cart clears on the success return.

**Done when:** a test-mode purchase on heavenlywaves.com pays → webhook grants
the entitlement → the pack appears in My Library and downloads.

## Notes

- **Design: single LS product.** LS checkouts are single-variant (no native
  multi-item cart). We use ONE generic "order" product in Lemon Squeezy and
  create each checkout with `custom_price` (the cart total in cents), a
  `product_options.name`/`description` override describing the cart, and
  `checkout_data.custom` carrying `user_id` + a compact items string
  (`productId:qty:lineCents,…`) the webhook decodes. DB products never need
  LS syncing (admin-created packs included).
- Requires sign-in before payment (entitlements need a `userId`); the action
  returns `SIGN_IN_REQUIRED` and the page redirects to
  `/sign-in?callbackUrl=/checkout`.
- Prices are recomputed server-side from the DB (`salePriceCents ?? priceCents`,
  PUBLISHED only) — the client cart is never trusted.
- New env vars (optional in `env.ts` so deploys without them don't crash; the
  action/webhook fail gracefully): `LEMONSQUEEZY_API_KEY`,
  `LEMONSQUEEZY_STORE_ID`, `LEMONSQUEEZY_VARIANT_ID` (the generic product's
  variant), `LEMONSQUEEZY_WEBHOOK_SECRET`.
- Webhook idempotency id: `${event_name}:${data.id}` (LS sends no event id);
  `Order.lemonSqueezyId` (unique) double-guards order creation.
- LS dashboard prereqs (user, test mode): create the generic product → variant
  id; webhook at `https://heavenlywaves.com/api/webhooks/lemonsqueezy` with
  `order_created` + `order_refunded` events.
- Resend receipt email: deferred to a follow-up (LS sends its own receipt).

## History
<!-- Keep this updated oldest to newest -->

- Project scaffolded from claude-starter
- Started `feature/main-page-ui`: navbar + hero header UI for the home page
- Added slide-in sidebar menu with social tooltips; restyled hero plugin list
- Added supported-plugins logo grid section
- Added Edition 02 (image statement) and Edition 03 (scroll-stacking cards with
  wave marks) sections; introduced Inter type + the constants-for-copy rule
- Added the site footer
- Moved Navbar + Footer into the root layout (global); hero stays home-only
- Added the shop catalog page (rows/grid toggle, pagination) on `feature/shop-page`
- Added the About page (`/about`): new `AboutHero` (full-bleed `a3.jpg`, "About"
  wordmark, register "+" marks) over reused `Edition` (Edition 01), `SupportedPlugins`
  (Edition 02), and `EditionStack` (Edition 03); copy in `constants/about.ts`
- Added the Contact page (`/contact`): `ContactHero` ("©Let's Talk" wordmark +
  availability card, reusing the footer's hero treatment) over `ContactForm`
  (dark panel, "©"-marked Name/Email/Message fields, disclaimer, "Contact Us"
  button) on a dark canvas that blends into the global footer; copy in
  `constants/contact.ts`. UI only — no server action wired yet (lands with the
  Resend email flow)
- Added the **category filter** chip bar to the shop catalog (filter packs by
  plugin) on `feature/shop-filter`
- **Phase 0:** adopted the `src/` layout (`app`/`components`/`constants`/`types`
  → `src/`) and repointed the `@/*` alias to `./src/*` on `feature/src-migration`
- **Foundation data layer** (`feature/foundation-setup`): Prisma 7 + Neon driver
  adapter, Zod-typed `src/lib/env.ts`, singleton `src/lib/prisma.ts`, full
  `schema.prisma` (NextAuth + taxonomy + catalog + songs/albums + commerce),
  migrations, dev branch connected. **Seeded the full catalog** from JSON
  (`prisma/seed.ts` + `prisma/seed-shop.ts`): 130 products (114 song packs +
  8 album bundles + 8 shop patch packs), 357 preset files, 11 plugins, 21 sound
  types, 8 albums, 114 songs. Merged + pushed.
  **Deferred (still in roadmap Phase 0):** Tailwind v4 `@theme` theme, shadcn/ui,
  Auth.js. **Not yet wired:** `/remakes` and `/shop` still render demo constants.
- **Wired `/shop` to the database** (`feature/shop-page-data`, commit `ca5668e`):
  `getShopProducts()` reads the `PACK` products; catalog is server-rendered.
- **Wired `/remakes` to the database** (`feature/shop-page-data`): new
  `getAlbums()` query (`src/lib/queries/songs.ts`) reads albums → tracklists →
  the `SONG_PACK`/`BUNDLE` products; page is now an async server component.
  Removed the demo `ALBUMS` + flat-price helpers from `constants/songs.ts`.
  Verified in the browser: 8 albums, 114 tracks, real song + bundle prices.
- **Built the product detail page** (`feature/shop-page-data`): new dynamic route
  `src/app/shop/[slug]/page.tsx` (server component, `getShopProductBySlug` →
  `ProductDetail`, `notFound()` on miss, `generateMetadata`). Assembled from four
  reusable sections mirroring `context/screenshots/details*` —
  `ProductDetailHero` (details1, full-bleed media + wordmark + register marks),
  `ProductDetailInfo` (details2, title + `Stars` rating + "what's included" pills
  + price/add-to-cart + spec table), and one shared `MediaSection` used twice
  (details3 single feature shot w/ centered mark, details4 side-by-side pair).
  A `Media` primitive renders image **or** `<video>` so any slot accepts either.
  Icons via **react-icons** (new dep) per request — no inline SVG. Row cards
  (`ProductRow`) now link to the detail page (tiles already did). Build passes;
  verified all four sections in the browser. Media is placeholder (see Notes).
- **Added a patch audio-preview list** to the detail page (after the info block):
  `PatchPreviewList` (client) lists each pack's `ProductFile` patches with a
  play/pause button, a synthetic per-patch `Waveform` (also a seek bar), and a
  time readout, driven by one shared `<audio>` element bound to each patch's
  `previewKey`. Reusable + accessible; plays for real once preview clips exist
  (currently 404 — see Notes). Inspired by `context/screenshots/sounds.png` but
  restyled to the dark editorial aesthetic. Build passes; verified in browser.
- **Grouped the patch previews by type** (Pads / Arps / Leads / Keys / Bass …).
  Added a nullable `ProductFile.category` (migration
  `20260607193000_productfile_category`, applied via `migrate deploy` —
  `migrate dev`'s shadow replay fails on the branch's out-of-order history where
  `..._init` sorts after the migrations that depend on it; the real DB is in
  sync). `seed-shop.ts` now fills `category` from each patch's `type`; re-seeded
  the 8 PACK packs (160 preset files). `PatchPreviewList` groups rows by
  category (first-seen order, rows numbered within each group) while one shared
  `<audio>` keeps play state correct across groups.
- **Replaced the stacked groups with a category selector.** `PatchPreviewList`
  now shows a chip per type (label + count) and renders only the selected
  category's patches (defaults to the first), so a pack shows ~4–7 rows at a time
  instead of all 20. State lives in the client component; the shared `<audio>`
  keeps playing across category switches.
- **Completed `feature/shop-page-data`** — `/shop`, `/remakes`, and the
  `/shop/[slug]` detail page are DB-backed; merged to `main`. Carry-forward
  gaps for later features: detail media + patch preview audio are placeholders
  (files not in `public/`/R2 yet — drop in with no code change); no reviews
  seeded so ratings show "New"; shop/remakes still render DRAFT rows (switch
  query filters to `status: "PUBLISHED"` before launch); 15 of 114 `SONG_PACK`
  products are `priceCents = 0` (likely a seed gap); migration history is
  out-of-order on this lineage (`..._init` sorts after dependent migrations) so
  `prisma migrate dev` shadow-replays fail — worth a squash.
- **Built the client-side shopping cart** (`feature/cart`, Phase 1): `CartProvider`
  holds global cart state in React context, persisted to `localStorage` so it
  survives reloads; `CartButton` in the navbar shows an item-count badge (white
  treatment to match the transparent nav). `CartDrawer` slides in from the right
  (same `#151515` panel + power-ease as the menu Sidebar) with line items, qty
  steppers, remove, a subtotal, and a placeholder "Checkout — coming soon" button
  (Lemon Squeezy hosted checkout is the next Phase 1 piece). `AddToCartButton` is
  a behavior-only wrapper (calls `addItem`, which also opens the drawer) wired
  into `ProductTile`, `ProductRow`, and the product detail page. Carried
  `priceCents` through `ShopProduct`/`ProductDetail` + the shop query for cart
  math; added a `formatCents` helper in `lib/format.ts`. The drawer's close
  control reuses the menu burger's `transition-transform duration-300 ease-out`.
  Build passes; merged to `main`. Carry-forward: real checkout + multi-item cart
  polish land with the Lemon Squeezy flow.
- **Built the checkout page** (`feature/checkout`, Phase 1): new `/checkout` route
  (`src/app/checkout/page.tsx`, thin server wrapper + metadata) rendering a client
  `Checkout` view — the order-review step between the cart and Lemon Squeezy's
  hosted payment page. Lists cart line items with qty steppers + remove (mirroring
  the drawer) and a sticky order-summary card (subtotal + "tax at payment" note);
  a `mounted` guard avoids an empty-state flash while the localStorage cart
  hydrates, and an empty-cart state offers a "Browse the catalog" link. The
  "Continue to payment" button is a **disabled placeholder** until the LS hosted
  checkout is wired (deferred until the site has a real domain —
  [[lemon-squeezy-deferred]]). Wired the cart drawer's Checkout button to link to
  `/checkout`. Copy in `constants/checkout.ts` typed against `types/checkout.ts`;
  guest checkout (sign-in lands later). Build passes; verified both states in the
  browser; merged to `main`.
- **Built Cloudflare R2 + signed downloads** (`feature/r2-downloads`, Phase 1):
  `src/lib/r2.ts` is a singleton S3-compatible R2 client (region `auto`, account
  endpoint) plus `getSignedDownloadUrl(r2Key, { filename })` — a 60s presigned GET
  URL with a `Content-Disposition` filename; it strips a leading slash so the
  seed's web-path-style keys (`/downloads/…`) match R2 objects uploaded as
  `downloads/…`. `lib/download.ts` holds pure helpers (`isFreeProduct`,
  `downloadFilename`); `lib/queries/downloads.ts` fetches a file's r2Key + label
  + product price/status. `GET /api/download/[fileId]` looks up the file, mints a
  signed URL, and `302`-redirects — `404` if missing/archived, `403` for paid
  files (free lead-magnets only until Auth.js + the LS webhook grant
  `Entitlement`s — `TODO(auth)`; see [[lemon-squeezy-deferred]]). `env.ts`
  validates the five R2 vars. Deps: `@aws-sdk/client-s3` +
  `@aws-sdk/s3-request-presigner`. **Verified:** R2 creds work end-to-end (upload
  → sign → download → cleanup against the real bucket); route returns 404 (bogus)
  and 403 (paid). Build passes; merged to `main`.
  **Carry-forward / upload conventions:** R2 holds three groups —
  `downloads/patches/<pack>/<name>.zip` (private deliverables, the priority),
  `audio/demos/<pack>/<name>-demo.mp3` (public preview clips), and
  `images/products/<pack>/<name>-pack.webp` (public cover art). Nothing is
  downloadable through the app yet: the only free pack (`the-chorus-not-done-yet`)
  has no `ProductFile`s, and packs that do have files are paid (→ 403 until auth).
  No test runner is installed (Vitest is referenced in CLAUDE.md but absent), so
  helpers are pure-but-untested for now. Bucket is named `sound-desing` (sic).
- **Built Auth.js v5 core auth — Phase 1** (`feature/auth`, Phase 0/1; plan in
  `context/features/auth-plan.md`): NextAuth v5 (`next-auth@beta`) +
  `@auth/prisma-adapter`, **split config** — `src/auth.config.ts` (edge-safe:
  Google provider + jwt/session callbacks surfacing `id` + `role`, no adapter) +
  `src/auth.ts` (PrismaAdapter on the `@/lib/prisma` singleton, `strategy: "jwt"`).
  Handler at `src/app/api/auth/[...nextauth]/route.ts`; **`src/proxy.ts`** (Next 16
  rename of middleware, Node runtime) redirects `(account)/*` to sign-in and gates
  `(admin)/*` on `role === "ADMIN"`, reading role straight off the JWT (no DB hit).
  `src/types/next-auth.d.ts` types `Session.user`/JWT with `id` + `role`;
  `src/lib/auth-guards.ts` adds `requireUser()`/`requireAdmin()`. Placeholder
  `/account` (with a server-action **Sign out** button) + `/admin` pages prove the
  gates. Navbar gained an `AccountButton` (User icon → `/account`; proxy bounces to
  sign-in when logged out). `env.ts` validates `AUTH_SECRET`/`AUTH_GOOGLE_ID`/
  `AUTH_GOOGLE_SECRET`/`AUTH_TRUST_HOST` (also repaired empty AUTH vars in
  `.env.production` that overrode `.env` at build). **No migration** — schema
  already had `User`/`Account`/`Session`/`VerificationToken`/`Role`. **Verified in
  the browser:** Google sign-in completes; `/account` + `/admin` redirect when
  anon; ADMIN reaches `/admin`, a CUSTOMER is bounced to `/`; sign-out works. Build
  passes. **Gotcha:** JWT bakes `role` at login — change it in the DB then sign
  out/in to refresh. **Deferred (later phases):** email/password Credentials +
  register (Phase 2); branded sign-in/up UI + session-aware navbar avatar/dropdown
  (Phase 3); email verification + password reset (needs Resend). No Vitest runner,
  so `auth-guards` is untested for now.
- **Added email/password auth — Phase 2** (`feature/auth-credentials`; plan in
  `context/features/auth-plan.md`): `bcryptjs` for hashing; shared Zod schemas in
  `src/lib/validation/auth.ts` (`signInSchema`, `registerSchema` with a
  password-match refinement). A **Credentials** provider lives in `src/auth.ts`
  (not the edge config) with a `bcrypt.compare` `authorize`, extending
  `authConfig.providers` *after* the spread so it adds to — not replaces — Google;
  keeps bcrypt/Prisma off the proxy's edge-safe config. It rejects OAuth-only
  accounts (null `passwordHash`). `POST /api/auth/register` is Zod-validated,
  normalizes + dedupes the email (409 on conflict), `bcrypt.hash`es (cost 12),
  and creates a CUSTOMER `User`; returns `{ success, error? }`, never leaking the
  hash. The default `/api/auth/signin` page (served by `auth.ts` handlers) now
  shows Google **and** an email/password form. **No migration** — `passwordHash`
  already on `User`. **Verified in the browser:** register 201 / 409 dup / 400
  mismatch / 400 invalid-email; credentials sign-in establishes a JWT session;
  protected `/account` renders the user; Google still works. Build passes; test
  user cleaned from the dev DB. **Deferred (Phase 3):** branded sign-in/up UI +
  session-aware navbar avatar/dropdown. Still no Vitest runner, so the register
  route + schemas are untested for now.
- **Built the branded auth UI + session-aware navbar — Phase 3** (`feature/auth-ui`;
  plan in `context/features/auth-plan.md`): an `(auth)` route group with `/sign-in`
  and `/sign-up`, styled to the shop references (`shop1.png`/`list.png`) — a shared
  `AuthShell` puts a dark intro band (eyebrow + register mark + oversized heading,
  keeping the overlaid white navbar legible) over a cream form. Copy in
  `constants/auth.ts` typed by `types/auth.ts`. `SignInForm`/`SignUpForm` (client)
  reuse the Phase 2 Zod schemas; sign-up posts to `/api/auth/register` then auto
  signs in via Credentials; sign-in uses `signIn("credentials", { redirect:false })`
  + a `GoogleAuthButton`; both honor the proxy's `?callbackUrl`. New `Avatar`
  primitive (`components/ui/Avatar.tsx`) renders the Google image or name initials.
  `UserMenu` (replaces the Phase 1 `AccountButton`) shows the avatar + a dark
  dropdown (name/email, Account, Admin **only** for `role === "ADMIN"`, Sign out via
  client `signOut`); a user icon → `/sign-in` when signed out. Wired a
  `SessionProvider` into the root layout, set `pages.signIn = "/sign-in"`, and
  pointed the proxy + `auth-guards` redirects at `/sign-in`. Removed the now-redundant
  page-level Sign out and deleted the dead `AccountButton` + `SignOutButton`.
  **Verified in the browser:** sign-up → auto sign-in → `/account`; credentials +
  Google; avatar/menu role-gated (no Admin link for a CUSTOMER); sign-out clears the
  session and reverts the navbar; `/account` bounces anon users to `/sign-in` with
  `callbackUrl`. Build passes; dev DB cleaned. **Auth plan complete** except the
  deferred, Resend-dependent pieces: email verification on signup + password reset.
- **Added Tailwind v4 `@theme` palette tokens** (`feature/theme-tokens`, Phase 0):
  consolidated the four repeated brand colors into `@theme` tokens in `globals.css`
  — `--color-ink` `#0a0a0a`, `--color-void` `#070707`, `--color-cream` `#f3f1ec`,
  `--color-panel` `#151515` — then migrated 57 arbitrary classes across 24
  components to the generated utilities (`bg-ink`/`text-ink`/`bg-void`/`bg-cream`/
  `bg-panel`). Exact-value tokens → zero visual change; verified on the production
  build/server (sign-in band + cream form render identically). Scattered one-off
  darks (`#0d0d0d`, `#181818`, `#202020`, `#232323`, `#333`, `#0e0e0e`) left as
  arbitrary classes for now. **Dev gotcha:** Tailwind v4 doesn't always hot-reload
  new `@theme` tokens — restart `npm run dev` if new tokens don't apply.
- **Reverted the `@theme` token migration.** Turbopack's dev pipeline would not
  emit the new `@theme` block's utilities even after a dev-server restart
  (`bg-void`/`bg-cream` resolved transparent → white surfaces), though the
  production build generated them correctly. Rather than fight the dev cache,
  restored all of `src/` to the pre-theme state (`c59aca0`): removed the `@theme`
  tokens from `globals.css` and reverted the 57 class swaps back to the
  self-contained arbitrary hex classes (`bg-[#070707]`, `bg-[#f3f1ec]`, …), which
  render reliably in dev. Verified colors back in the browser. **If we retry
  theme tokens later:** likely needs a clean `.next` (Turbopack cache) on first
  introduction, and/or merging tokens into the existing `@theme inline` block
  instead of a second `@theme` block.

- **Built the admin console shell — Admin Phase 1** (`feature/admin-shell`; plan in
  `context/features/admin/`): a gated `/admin` area with its own chrome and read-only
  views. New `(admin)/layout.tsx` calls `requireAdmin()` (gates the whole group,
  defense-in-depth with the proxy) and renders `AdminSidebar` (client, `usePathname`
  active-route highlight; Dashboard/Products/Orders live, Albums/Reference/Reviews
  shown disabled until their phases). **Suppressed the marketing Navbar/Footer on
  `/admin` via approach (b):** a client `SiteChrome` gate in the root layout reads
  `usePathname` (SSR-aware → no flash) and renders the chrome everywhere *except*
  `/admin`, passing the server Navbar/Footer/CartDrawer through as slots — chosen over
  relocating every public route into a `(public)` group since chrome is wanted on
  `(auth)`/`(account)` too. Read-only **dashboard** (stat cards from Prisma
  counts/aggregates: products + status split, orders + PAID revenue, users, reviews,
  reference-data counts), **products** table (cover monogram, title, kind, plugins,
  sale-aware price, status badge, file count, featured/unlisted flags) with URL-param
  title search + status filter + pagination, and a read-only **orders** table (empty
  state until the LS chain lands). New `src/lib/queries/admin.ts` (admin reads — sees
  DRAFT/ARCHIVED, unlike the public PUBLISHED-only queries), `constants/admin.ts` +
  `types/admin.ts` (nav/copy), `StatCard`/`StatusBadge`/`AdminPagination` (dark
  pager). **No mutations, no migration.** Build + lint pass (one pre-existing img
  warning). **Verified in the browser as ADMIN:** anon `/admin` → `/sign-in?callbackUrl`;
  dashboard shows real counts (130 products, 2 published/128 draft, 11 plugins/3
  genres/21 sound types); products paginate (Page 1 of 7) and filter (`?search=gold&
  status=PUBLISHED` → 1 row); orders empty state; marketing nav/footer absent on
  `/admin` but intact on the public site. Temp admin user created for verification then
  deleted. Still no Vitest runner, so the admin queries are untested for now.

- **Built admin catalog management — Admin Phase 2** (`feature/admin-catalog`; plan in
  `context/features/admin/phase-2-catalog.md`): the project's **first `src/actions/admin/`**.
  Three guarded, Zod-validated Server-Action modules (each re-checks `requireAdmin()`,
  writes via `prisma`, `revalidatePath()`s the admin + public surfaces, returns
  `{ success, data?, error? }`): `products.ts` (`createProduct`/`updateProduct` over a
  shared `productSchema` — text/enums/money/flags/tags/counts/media-keys + many-to-many
  `pluginIds`/`genreIds`/`soundTypeIds` via `connect` on create, `set` on update;
  `setProductStatus`; `deleteProduct`, blocked when the product has `orderItems` →
  archive instead, else cascades files), `product-files.ts`
  (add/update/remove/`reorderProductFile` neighbour-swap), and `taxonomy.ts`
  (`createTerm`/`renameTerm`/`deleteTerm` over plugin/genre/soundType via per-op switches —
  a union of Prisma delegates doesn't typecheck — with delete blocked when products
  reference the term). New `lib/validation/admin.ts` (schemas + concrete hand-written
  input types, not `z.input`, to dodge coercion inference) and `lib/slug.ts` (`slugify`).
  UI: `/admin/products/new` + `/admin/products/[id]` server pages → client `ProductForm`
  (all fields; **checkbox-group** multi-selects for the m2m relations; title→slug
  auto-suggest until the slug is touched; inline validation), `ProductFiles` editor
  (add/edit/remove/reorder; `r2Key` as text — upload UI is Phase 3),
  `DeleteProductButton` (danger zone), and `/admin/reference` → `ReferenceManager`
  (inline add/rename/delete per term with usage counts). Enabled the **Reference data**
  nav item + a **New product** button on the list. Admin query helpers added
  (`getProductForEdit`, `getProductFormOptions`, `getReferenceData`). **No toast lib**, so
  feedback is **inline status banners** (mirrors `SignUpForm`); shared `ActionResult`
  type in `types/action.ts`. **No migration.** Build + lint pass (one pre-existing img
  warning). **Verified in the browser as ADMIN:** added a plugin term → created a PACK
  product (relation persisted as `set`/`connect`) → redirected to edit → added a
  `ProductFile` → set status PUBLISHED + saved → product shows `published`, `1` file,
  `$29`, linked plugin in the list; **delete-guard** on the in-use term surfaced the
  inline error; `deleteProduct` removed the product (confirm dialog → cascade) and
  redirected. Test product/term/admin cleaned from the dev DB. Still no Vitest runner,
  so the actions/schemas are untested for now.

- **Built admin media uploads, albums/songs, orders & reviews — Admin Phase 3**
  (`feature/admin-media-content`; plan in `context/features/admin/phase-3-media-content.md`).
  Completes the console. **R2 uploads:** `getSignedUploadUrl` (presigned PUT) in
  `lib/r2.ts`; guarded API route `POST /api/admin/upload` (`auth()` ADMIN check →
  validates group + content type via `lib/upload.ts` → `{ uploadUrl, r2Key }`); client
  `UploadField` (pick → sign → XHR PUT with progress → fills the key field), wired into
  `ProductForm` (cover/preview/demo/video) + `ProductFiles` (deliverable) + `AlbumForm`
  + the song editor. Groups: `images/products`, `audio/demos`, `videos/products`,
  `downloads/patches`. **Browser PUT needs bucket CORS** → committed
  `scripts/setup-r2-cors.ts` + `npm run setup:r2-cors` (not yet run — needs
  authorization to modify the shared bucket). **Albums & Songs:** `actions/admin/albums.ts`
  (album CRUD + song CRUD, slug unique-per-album, `setSongProducts` for the song↔pack
  m2m), `/admin/albums` (list) + `/admin/albums/new` + `/admin/albums/[id]` (`AlbumForm`
  + `SongsManager` with per-song product-link chips). **Orders + entitlements:**
  `actions/admin/entitlements.ts` `grantEntitlement` (wraps a synthetic PAID order with a
  `manual:<uuid>` `lemonSqueezyId` sentinel — comps before the LS chain) / `revokeEntitlement`;
  `/admin/orders/[id]` detail (line items, buyer, entitlements + revoke) + a `GrantAccessForm`
  on the orders list. **Reviews:** `actions/admin/reviews.ts` `deleteReview` (hard delete);
  `/admin/reviews` moderation list. Enabled the Albums/Reference/Reviews nav. New queries
  (`getAdminAlbums`, `getAlbumForEdit`, `getSongProductOptions`, `getAdminOrderDetail`,
  `getGrantableProducts`, `getAdminReviews`). **No migration.** Build + lint pass (one
  pre-existing img warning). **Verified in the browser as ADMIN:** grant access → manual
  PAID order created + order detail (items/entitlement/buyer/`manual:` ref) → revoke;
  created an album → added a song → linked a SONG_PACK; deleted a seeded review; the
  upload API route returns a real presigned R2 URL + correct key (server half verified —
  the literal browser→R2 PUT is blocked until `setup:r2-cors` runs). Test data cleaned
  from the dev DB. Still no Vitest runner, so actions/schemas remain untested for now.

- **Built My Library + entitlement-gated downloads** (`feature/my-library`, Phase 1
  delivery side): `/account/library` lists a signed-in user's owned packs and
  re-downloads. `getUserLibrary` (`lib/queries/library.ts`) reads the user's
  `Entitlement`s newest-first, then resolves the products by id (Entitlement has no
  `product` relation — only `productId`) with their `ProductFile`s (BigInt `sizeBytes`
  → Number). Page mirrors the **store design** (`context/screenshots/list.png`): a short
  dark intro band over a cream `#f3f1ec` / ink `#0a0a0a` editorial list — `LibraryRow`
  reuses the shop's `ProductRow` three-column layout but swaps add-to-cart for per-file
  **Download** links (label + kind + size → `/api/download/[fileId]`); empty state links
  to `/shop`. Copy in `constants/library.ts` typed by `types/library.ts`. **Upgraded the
  download route**: `GET /api/download/[fileId]` now does a **per-user entitlement check**
  (free `$0` packs still open to anyone; paid files need the signed-in user to own an
  `Entitlement` → else **401** anon / **403** non-owner) and bumps
  `Entitlement.downloadCount` on each paid grant; `getDownloadFile` gained `productId` +
  new `getEntitlement`. Wired a **My Library** link into the navbar `UserMenu` + a CTA on
  `/account`. **No migration.** Build + lint pass. **Verified in the browser:** paid file
  → 403 before owning; admin grant (Phase 3 tool) → pack appears in Library with two
  download links; owner download → 302 signed R2 URL; `downloadCount` 0→1; empty state +
  cream design confirmed by screenshot. Test data cleaned. Still no Vitest runner, so the
  library query + download route stay untested for now. Added `context/features/admin/
  FOLLOW-UPS.md` (the R2 CORS task) to the repo.

- **Added an admin help guide + light/dark theme toggle** (`feature/admin-guide`):
  a personal, ADMIN-only **`/admin/guide`** ("How to run the store") rendered from
  `constants/admin-guide.ts` (typed by `types/admin-guide.ts`) — a how-to for
  adding/removing products, files, albums & songs, **linking packs to a song**
  (spells out the rule that only `SONG_PACK`/`BUNDLE` products appear in the song's
  link dropdown), granting access, reviews/reference data, and a pre-launch
  checklist; a sticky table of contents. Added a **"Guide"** sidebar item. **Plus a
  light/dark toggle for the admin console:** `AdminThemeProvider` (wraps the layout,
  reads the preference from `localStorage` via `useSyncExternalStore`, applies it as
  `data-admin-theme` on a `.admin-shell` wrapper) + a `ThemeToggle` sun/moon button
  at the bottom of the sidebar. **Theming approach:** the admin is built almost
  entirely from Tailwind's `white`/`black` utilities, which compile to
  `var(--color-white)`/`var(--color-black)` in v4 — so dark mode just overrides
  those two variables inside `.admin-shell[data-admin-theme="dark"]` in `globals.css`
  (plus `color-scheme: dark` for native controls), flipping the whole console with
  **no per-component `dark:` classes**. Only 3 hardcoded `#0a0a0a` hexes were
  converted to `black` utilities so they flip too; the public site never sets
  `.admin-shell`, so it's untouched. **No migration.** Build + lint pass (one
  pre-existing img warning). **Verified in the browser as ADMIN** (temp admin user,
  deleted after): dashboard, guide page, and the product form (inputs, native
  selects, textareas, placeholders) all render with good contrast in both modes; the
  toggle persists across navigation.

- **Rebuilt the admin dashboard with analytics charts** (`feature/admin-dashboard-charts`):
  replaced the 6 stat boxes with a KPI row (Revenue, Orders, Avg. order value,
  Products) + a chart grid using **Recharts 3** (`recharts`, first chart dep). Charts:
  **Revenue** (area, 12-month) + **Orders** (bar, 12-month) — both show empty states
  until paid orders exist; **Products by status** + **Products by type** (donuts with
  side legends), **Products by plugin** + **Price distribution** (horizontal bars), and
  a **Top products** units-sold list (empty until sales). New `lib/queries/dashboard.ts`
  (`getDashboardData` — one batched read: paid-order rows for the series, groupBys for
  status/kind, plugin `_count`, non-archived prices, `orderItem` groupBy for top
  sellers) and pure, testable helpers in `lib/dashboard.ts` (`lastMonths`/
  `bucketByMonth` month-series, `priceBands`, `averageOrderValueCents`). Removed the
  now-dead `getDashboardStats`/`DashboardStats` from `queries/admin.ts`. Chart colours
  derive from the admin light/dark theme via a `useChartTheme` hook (Recharts can't read
  CSS vars), so charts flip with the console. **Recharts gotcha:** its
  `ResponsiveContainer` reported width/height of -1 on first paint (charts blank until a
  manual resize) — replaced it with a small `ChartFrame` that measures via
  `ResizeObserver` and hands explicit px dimensions to the chart, fixing first-paint
  render (0 console warnings after). **No migration.** Build + lint pass (one
  pre-existing img warning). **Verified in the browser as ADMIN** (temp admin, deleted
  after): all charts render on load in both light + dark with real catalog data (130
  products, 11 plugins, status/kind/price splits); revenue/orders/top-products show
  empty states (no sales yet). Charts deps add 5 moderate `npm audit` advisories
  (transitive, dev-time) — not addressed.

- **Added filtering + sort to the admin products list** (`feature/admin-products-filters`):
  the `/admin/products` table had only title search + status filter (newest-first), which
  doesn't scale to ~200 products. Extended `getAdminProducts` with `kind`, `pluginId`
  (m2m `plugins.some`), and `sort` args (`PRODUCT_SORTS`: newest/oldest/title/price-desc/
  price-asc → `productOrderBy`); the row now carries `createdAt`. `AdminTableControls`
  gained **Type** (kind), **Plugin**, and **Sort** dropdowns plus a **Clear** button
  (shown when any filter is active), all URL-param driven and resetting to page 1; the
  page fetches plugin options via the existing `getProductFormOptions` and added an
  **Added** (date) column. Removed the now-dead `getDashboardStats` earlier; here just
  reused `AdminPagination` (already echoes all params). **No migration.** Build + lint
  pass (one pre-existing img warning). **Verified in the browser as ADMIN** (temp admin,
  deleted after): Type=Bundle → 8 rows; Plugin=Serum → only Serum packs; sort by
  price-desc and title A–Z both correct; Clear resets; filters survive pagination.

- **Added Genre + Sound-type filters to the admin products list**
  (`feature/admin-products-taxonomy-filters`): rounded out the taxonomy filters on the
  same pattern — `getAdminProducts` gained `genreId`/`soundTypeId` args (m2m
  `genres.some`/`soundTypes.some`); `AdminTableControls` now also takes `genres` +
  `soundTypes` (already returned by `getProductFormOptions`) and renders **All genres** +
  **All sound types** dropdowns, both folded into the Clear/`hasFilters` check and the
  `activeParams` echoed onto pagination. **No migration.** Build + lint pass (one
  pre-existing img warning). **Verified in the browser as ADMIN** (temp admin, deleted
  after): both dropdowns list real taxonomy; Sound type=Bass → 2 matching products;
  combines with the other filters; Clear resets.

- **Sortable headers, a checkmark sort menu, and numbered pagination on the products
  list** (`feature/admin-products-sort-table`): reworked sorting into one `<field>-<dir>`
  vocabulary (`PRODUCT_SORTS`/`DEFAULT_PRODUCT_SORT` = `date-desc`) in a **new
  client-safe `lib/product-sort.ts`** — moved out of `queries/admin.ts` because the
  client `SortMenu` importing a value from the Prisma-bearing query module pulled server
  code into the browser bundle (Turbopack chunk error). `getAdminProducts`'s
  `productOrderBy` now parses field+dir (title/price/type→kind/date, asc/desc). The
  table headers (Title/Type/Price/Added) are clickable `SortableTh` links that toggle
  direction with an arrow indicator; a `SortMenu` (popover, checkmark on the active
  option, closes on outside-click/Escape) replaces the old native sort `<select>` — both
  drive the same `sort` URL param so they stay in sync. `AdminPagination` upgraded from
  prev/next to **numbered pages** (windowed with `…`), shared by orders too. Page size
  stays 20. **No migration.** Build + lint pass (one pre-existing img warning). **Verified
  in the browser as ADMIN** (temp admin, deleted after): Price header → `price-desc`
  (rows ordered by base price) with the menu checkmark in sync; Title A–Z via the menu lit
  the header arrow; numbered pager (1…7) preserved sort across page 2; 20 rows/page.

- **Album list: cover thumbnails, sortable headers, artist filter** (`feature/admin-albums-table`):
  `/admin/albums` gained a **Cover** column, **sortable Title / Artist / Year** headers
  (reusing `SortableTh`), and an **artist filter** — click an artist in a row to see only
  that artist's albums (active-filter chip with a clear ×). `getAdminAlbums` now takes
  `{ artist, sort }` (exact-match `artist` where; `ALBUM_SORTS`/`albumOrderBy` =
  name/artist/year × asc/desc, default still sortOrder→title) and returns `coverImageKey`.
  The page mints **short-lived signed R2 URLs** for covers via `getSignedDownloadUrl`
  (no public R2 URL exists; signed GET without a filename serves inline) and renders them
  through a new client `AlbumCover` that falls back to a letter monogram when there's no
  key or the image fails to load (covers aren't uploaded yet — placeholder data). Plain
  `<img>` (eslint-disabled) since the optimizer can't cache a 60s presigned URL.
  Headers/filter are URL-param driven and compose (artist + sort preserved together).
  **No migration.** Build + lint pass (one pre-existing img warning). **Verified in the
  browser as ADMIN** (temp admin, deleted after): Year header → `year-asc` reorders
  2017→2026; clicking an artist set the chip + `?artist=…&sort=year-asc` (sort preserved);
  covers show monogram fallbacks (no images in R2 yet). Note: all 8 seed albums share one
  artist, so the filter doesn't narrow until albums by other artists exist.

- **Built the Customers admin area** (`feature/admin-customers`): a new gated
  **`/admin/customers`** (sidebar item, Users icon) filling the biggest missing CRUD gap —
  before this you could only see a user *count*. List page: name/email search + role
  filter + sortable Name/Email/Joined headers + numbered pagination, with avatar, role
  badge, order count, **total spent** (sum of PAID orders, one grouped query scoped to the
  page), owned-pack count, and join date. Detail page (`/admin/customers/[id]`): profile
  (avatar, auth method via `passwordHash`/`emailVerified`, role) + their orders (→ order
  detail), owned packs (entitlements, resolving titles since `Entitlement` has only
  `productId`), and reviews. New `getAdminCustomers`/`getAdminCustomerDetail` +
  `CUSTOMER_SORTS` in `queries/admin.ts`, a guarded `setUserRole` server action
  (`actions/admin/users.ts`) that promotes/demotes ADMIN↔CUSTOMER but **refuses to change
  your own role** (no self-lockout) and notes the JWT-baked-role caveat, surfaced by a
  `UserRoleToggle` client button (confirm + inline error). **Fixed a latent crash:** the
  `Avatar`'s `next/image` 500'd on Google profile photos because
  `lh3.googleusercontent.com` wasn't allowlisted — added it to `next.config.ts`
  `images.remotePatterns` (also fixes the navbar avatar for OAuth users). **No migration.**
  Build + lint pass (one pre-existing img warning). **Verified in the browser as ADMIN**
  (temp admin + a disposable temp customer, both deleted after): list shows all 5 users
  with roles/avatars; detail renders with empty order/pack/review states; promote
  CUSTOMER→ADMIN worked (button flipped to "Remove admin", profile showed Admin); the
  self-guard hid the toggle on the logged-in admin's own page.

- **Added a Recent activity feed to the dashboard** (`feature/admin-dashboard-activity`):
  a card under the trend charts merging the latest **orders, customer signups, and
  reviews** into one newest-first feed, each row linking to its admin detail (order →
  `/admin/orders/[id]`, signup → `/admin/customers/[id]`, review → `/admin/reviews`),
  with a type-tinted icon and a relative timestamp. New `getRecentActivity(limit)` in
  `queries/dashboard.ts` (caps each source at `limit`, merges, re-sorts, trims) + an
  `ActivityItem` type; new `timeAgo()` in `lib/format.ts`; presentational
  `RecentActivity` component. Dashboard page fetches it in parallel with
  `getDashboardData`. **No migration.** Build + lint pass (one pre-existing img warning).
  **Verified in the browser as ADMIN** (temp admin, deleted after): the feed listed the
  customer signups with "just now"/"2d ago" timestamps and signup icons; orders/reviews
  absent (none exist yet) — they'll appear once they happen.

- **Added the DAW project-template product kind** (`feature/daw-templates`): a new
  `TEMPLATE` `ProductKind` + `FileKind` (migration `20260611000000_add_template_kinds`,
  applied) for selling DAW project files (Ableton / MainStage / Logic …, delivered as
  a `.zip`) alongside preset packs. Surfaced `TEMPLATE` in the admin product-kind
  select, the products-table kind filter (`AdminTableControls`, "Template" label), the
  `PRODUCT_KINDS`/`FILE_KINDS` Zod enums, and the shop catalog query
  (`kind: { in: ["PACK", "TEMPLATE"] }`). **Then finished the customer-facing side** so
  a template doesn't read as a "pack of patches": a new presentational helper
  `lib/product-display.ts` (`isTemplateKind`, `contentSummary`) drives a `contents`
  string ("12 patches" for a pack, "Project template" for a template) that replaced the
  hardcoded "{n} patches" line on `ProductRow`, `ProductTile`, and the My Library
  `LibraryRow` (so a purchased template no longer shows "0 patches"). On the detail page
  (`getProductBySlug` now returns `kind`): templates get template-specific "what's
  included" pills (DAW project template / install guide / license), a DAW · Format ·
  License spec table (no "Presets" row), and the **patch audio-preview section is hidden
  entirely** (a template has no patches). Catalog `included` pills branch the same way.
  **No new copy in components** — the branch logic lives in the queries + helper.
  Build + lint pass (one pre-existing img warning). **Verified in the browser** (temp
  published Ableton template, deleted after): catalog card shows "ABLETON · PROJECT
  TEMPLATE" + template pills; the detail page shows the DAW/Format/License specs, no
  patches/presets language, and no preview list; a normal pack PDP is unchanged
  (preview section + Platform/Presets specs intact). Still no Vitest runner, so the new
  helper is untested for now.
- **Gated DRAFT products out of the public catalog** (`feature/daw-templates`): the
  shop list, product detail, and remakes queries used `status: { not: "ARCHIVED" }`,
  leaking DRAFT work-in-progress to shoppers. Added a shared `PUBLIC_STATUS_WHERE`
  (`lib/queries/visibility.ts`) keyed on `env.NODE_ENV` — PUBLISHED-only in production,
  non-archived (drafts visible) in development so unpublished work stays previewable
  while building — spread into `getShopProducts`, `getProductBySlug`, and `getAlbums`
  (album/song/product levels). Admin/library/dashboard reads untouched (they see all
  statuses by design). **Verified** by running the real queries under both `NODE_ENV`s:
  prod shows shop 8→5, remakes 8→1 album / 5 songs; dev unchanged. Build + lint pass.
- **Squashed the migration history to one clean baseline** (`feature/daw-templates`):
  the lineage was out-of-order — `20260607020642_init` (creates the base tables) sorted
  *after* four `20260606…` migrations that ALTER those tables, so `prisma migrate dev`'s
  shadow replay failed (ran the ALTERs before the CREATEs), forcing a `migrate diff` +
  `migrate deploy` workaround for every schema change. Replaced the 7 migrations with a
  single `20260606000000_init` generated via `prisma migrate diff --from-empty
  --to-schema` (verified byte-identical to the live DB — a `--from-config-datasource`
  drift check came back empty). Reset the dev `_prisma_migrations` ledger (cleared it,
  then `migrate resolve --applied 20260606000000_init`) — **DB schema/data untouched**,
  only the bookkeeping table. **Verified:** `migrate status` → 1 migration, up to date;
  `prisma migrate dev` now returns "Already in sync" (shadow replay passes). Originals
  backed up at `/tmp/migrations-backup-1781195430`. **Prod note:** the prod Neon branch
  was bootstrapped via `db push`, so on its first real `migrate deploy` mark this
  baseline applied with `prisma migrate resolve --applied 20260606000000_init` (its
  schema already exists) before any future migrations.

- **Storefront search & filter — Phase 2** (`feature/shop-search-filter`): the shop
  catalog had only plugin-chip filtering; added a **search box** (matches title /
  description / plugin), **Genre** + **Sound type** dropdowns, and a **Sort** control
  (Featured / Price ↑ / Price ↓) on top of the existing chips. Kept the page's
  established **client-side** model — `getShopProducts` now also returns each pack's
  `genres` + `soundTypes`, and `ShopCatalog` (client) combines all filters with AND
  logic, derives the genre/sound-type options from the catalog (so new taxonomy
  surfaces automatically), shows a live result count, a Clear button, and an empty
  state. New presentational `ShopFilters` component (cream palette, native selects with
  chevrons); copy in `SHOP_FILTERS` (`constants/shop.ts`). A filter dropdown **renders
  only when the catalog has those terms** — so with the current mock packs (no genres
  assigned) the Genre control is hidden rather than showing a lonely "All genres".
  Filter changes reset to page 1 via wrapped setters (no `setState`-in-effect — the new
  `react-hooks/set-state-in-effect` rule). Build + lint pass (one pre-existing img
  warning). **Verified in the browser:** sound-type Bass → 2 packs; Bass + search
  "pulse" → 1 pack (AND-combine, correct singular); no-match → empty state; Clear resets
  everything (8 packs, chips back to "All packs"); sort price-asc orders $17→$34; genre
  dropdown correctly hidden. URL-param state (shareable filtered links) left as a future
  enhancement — the shop stays client-state, consistent with its existing design.

- **SEO foundation — Phase 3** (`feature/seo-foundation`): the site had only a single
  root title/description and no crawl/share metadata. Added: **`metadataBase` + a title
  template** (`%s — Heavenly Waves`) and structural `openGraph`/`twitter` defaults in the
  root layout (built on the existing `NEXT_PUBLIC_APP_URL`); a **branded dynamic OG
  image** (`app/opengraph-image.tsx` via `next/og` `ImageResponse` — 1200×630 dark
  wordmark card, system font, no remote fetch) that Next applies as `og:image` +
  `twitter:image` site-wide; **`app/sitemap.ts`** (static routes + every PUBLISHED
  product's `/shop/[slug]`, hourly `revalidate`, via a new PUBLISHED-only
  `getSitemapProducts` query) and **`app/robots.ts`** (allow public, disallow
  account/admin/checkout/auth/api, link the sitemap); **per-page `title` + `description`
  + canonical** on shop / remakes / about / contact; and on the product page,
  **canonical + schema.org `Product`/`Offer` JSON-LD** (name, brand=plugin, price,
  USD, InStock, plus `AggregateRating` when the pack has reviews). **Key gotcha:**
  setting a per-page `openGraph` object *replaces* the parent's (dropping the
  file-based image), so pages set only `title`/`description` and let `og:title`/
  `og:description` **fall back** to them — keeping the inherited OG image. Build + lint
  pass (one pre-existing img warning). **Verified with curl:** `/robots.txt` correct;
  `/sitemap.xml` lists 16 URLs (5 static + 11 published products); `/opengraph-image`
  → 200 image/png 1200×630 (rendered card looks on-brand); the PDP emits the title
  template, canonical, `og:*` + `twitter:*` (image inherited), and valid Product JSON-LD
  (price 29.00 USD InStock). **Future:** per-product OG images (a `[slug]/
  opengraph-image.tsx` using the pack art) once real cover art is uploaded.

- **Customer review submission — Phase 2** (`feature/reviews`): buyers can now rate +
  review packs on the product page, feeding the existing `Stars` aggregate + the
  `/admin/reviews` moderation list (which already existed). **Migration**
  `20260611120000_review_unique_and_updatedat` adds `Review.updatedAt` + a
  `@@unique([userId, productId])` (one review per user/product → enables `upsert`);
  authored by hand (`migrate diff` → SQL with a `DEFAULT CURRENT_TIMESTAMP` so it's
  safe on non-empty tables) because `migrate dev` is interactive on the constraint
  warning, then applied via `migrate deploy`. New **`submitReview` server action**
  (`src/actions/reviews.ts`, the project's first customer-facing action outside auth/
  checkout): `auth()` + Zod (`lib/validation/reviews.ts`, rating 1–5, body ≤2000) +
  **owner-gate** (must hold an `Entitlement` via the existing `getEntitlement`) →
  `upsert` → `revalidatePath` the PDP + admin. New `getProductReviewData(productId,
  userId)` query (reviews list + the viewer's own review + `canReview`/`isSignedIn`).
  UI: a client `ReviewForm` (interactive star picker + textarea + char counter, inline
  status like the auth forms, `router.refresh()` on success) and a presentational
  `ProductReviews` section (list on the left; form for owners, "own it to review" for
  signed-in non-owners, or a sign-in link for anon on the right) wired into the PDP
  after the previews. Added `id` to `ProductDetail` (the form/query need productId).
  Build + lint pass. **Verified in the browser** (temp customer + synthetic paid order/
  entitlement, deleted after): owner posts a 4★ review → appears in the list with a
  "Your review" badge + author, aggregate flips to "4.0 · 1 review", form switches to
  Edit; editing to 5★ updates in place (still 1 review, "5.0 · 1 review") — `upsert`
  confirmed; signed-in non-owner sees the own-it prompt (no form). **Gotcha:** the
  running dev server cached a stale Prisma client from before the migration — the first
  `upsert` failed until the dev server was restarted to load the regenerated client.
- **Fixed a double site-name suffix in page titles** (`feature/reviews`): the SEO
  feature's new `%s — Heavenly Waves` title template doubled up on the six pages whose
  titles already ended in "— Heavenly Waves" (sign-in/up, account, library, checkout,
  admin) → "Sign in — Heavenly Waves — Heavenly Waves". Dropped the suffix from those
  titles so the template adds it once. Verified the sign-in `<title>` is now single.

<!---->