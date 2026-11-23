# Storefront SaaS: modular monolith

One Next.js 15 (App Router) + TypeScript codebase that serves the admin app,
the merchant API (`/api/v1`) and every tenant storefront. A second entrypoint
(`npm run worker`) runs background jobs from the same code.

## Run it

    cp .env.example .env
    docker compose up -d postgres
    npm install
    npx prisma migrate dev --name init   # creates the first migration, commit it
    npm run db:seed
    npm run dev                          # web
    npm run worker                       # jobs (second terminal)

Seed user: `demo@example.com` / `demo-password-123`.
Storefront: http://demo.localhost:3000 (modern browsers resolve `*.localhost`).

Checks: `npm run typecheck && npm run lint && npm test`.
Integration tests need a migrated DB (`DATABASE_URL`); they skip otherwise.

## Layout

    prisma/                 schema + seed
    src/
      server/               kernel: db, tenant-db, http wrapper, errors, security, rate-limit
      config/env.ts         validated environment
      modules/              business logic, one folder per domain (no HTTP in here)
        auth/  tenancy/  catalog/  storefront/  audit/
      app/
        api/v1/**           thin route handlers: parse, authorize, call a module
        sites/[tenant]/**   storefront (reached only through middleware rewrites)
        admin/ login/ verify/
      middleware.ts         host -> tenant routing
      worker/index.ts       job runner

Rules: route handlers never touch Prisma directly; modules never import from
`app/`; a module talks to another module only through its service functions.
When a module needs to become a service, its boundary is already drawn.

## Multi-tenancy (the important part)

1. Every tenant table has `storeId`; composite FKs (`Variant -> Product(id, storeId)`,
   `Membership -> Role(id, storeId)`) make cross-store links impossible in the DB.
2. `requireStoreContext(req, storeId)` resolves user -> active membership -> store.
   The URL's storeId is only honored if the membership exists. Non-members get 404.
3. `ctx.db` is `forStore(storeId)`: a Prisma extension that injects `storeId` into
   every query on tenant models. Register new tenant models in `TENANT_MODELS`.
4. Storefront hosts are resolved server-side (`modules/tenancy/host.ts`); custom
   domains only count when `VERIFIED`.
5. `tests/integration/tenant-isolation.test.ts` proves it. Extend it with every
   new tenant table.

## Themes (multi-theme storefronts)

- **Themes are code** (`src/themes/`): each base theme (Classic, Modern, Bold) is a React
  `Layout` registered in `registry.tsx`. The DB stores only each store's *choices*.
- **Presets and page starters are templates, not limits**: `catalog.ts` holds them as pure
  data. Applying one copies values into the store's own draft; the merchant can then change
  anything. Adding the rest of the 37 presets / 24 starters is data-only (append objects;
  `tests/unit/themes.test.ts` validates every one).
- **Tokens + typed sections, never raw HTML/CSS/JS**: colors are 6-digit hex, fonts and radius
  are allowlists, sections are a zod discriminated union (hero, productGrid, richText), links
  are `/path` or `https://`. Validated on write AND on read.
- **Draft / preview / publish**: `StoreTheme` and `Page` hold draft + published copies.
  `POST .../theme/publish` promotes the theme and all pages atomically and stores a
  `ThemeRevision`; `POST .../revisions/:id/restore` loads a past revision into the draft.
- **Preview**: `POST .../theme/preview` returns a signed 30-minute link; the storefront sets an
  httpOnly cookie that is honored only for the store it was signed for.
- Extending: add a section type = one zod schema + one component in `Sections.tsx`.
  Add a base theme = one entry in `catalog.ts` + one `ThemeImpl` in `registry.tsx`.
- **Visual editor** at `/admin/stores/:storeId/editor` (`src/editor/`): section list with
  drag-and-drop reordering (native HTML5 DnD, no dependency; Up/Down buttons for keyboard),
  drag blocks in from the palette, live canvas, inspector forms, theme panel (colors, fonts,
  radius, base theme, industry presets), page switcher + page starters, history/rollback,
  Save draft (Ctrl/Cmd+S), Preview on the real storefront, Publish.
- **Device frames**: the canvas renders the same theme layout and section components as the
  storefront at 1200 / 768 / 390 px, scaled to fit. Sections reflow with container query units.
- **Google Fonts**: 10 allowlisted families (see `FONTS` in `themes/schema.ts`). Merchants pick
  a key, the URL is built server-side. To add a font, add one entry.
- **Images**: `POST /api/v1/stores/:id/media` (multipart `file`). Type is detected from magic
  bytes (PNG/JPEG/GIF/WebP only; SVG refused), 5 MB cap, server-generated file names.
  Storage is behind `src/server/storage.ts` (local disk now; implement the same interface for S3).
  Sections reference images by asset id, and the server rejects ids from other stores.
  Served publicly at `/api/media/:id` with immutable caching.
- Not built yet: undo/redo, inline text editing on the canvas, image cropping/resizing and CDN,
  media deletion, per-theme custom sections.

## API (so far)

    POST /api/v1/auth/signup | verify | login | logout
    GET  /api/v1/me
    GET|POST /api/v1/stores
    GET  /api/v1/stores/:storeId
    GET|POST /api/v1/stores/:storeId/products
    GET|PATCH /api/v1/stores/:storeId/products/:productId
    POST /api/v1/stores/:storeId/products/:productId/publish
    GET  /api/v1/themes                                   catalog: base themes, presets, starters
    GET|PUT  /api/v1/stores/:storeId/theme                theme state / save draft
    POST /api/v1/stores/:storeId/theme/preset             apply a preset to the draft
    POST /api/v1/stores/:storeId/theme/preview            signed draft-preview link
    POST /api/v1/stores/:storeId/theme/publish            publish theme + pages, snapshot revision
    GET  /api/v1/stores/:storeId/theme/revisions          history
    POST /api/v1/stores/:storeId/theme/revisions/:id/restore
    GET  /api/v1/stores/:storeId/pages                    pages list
    GET|PUT /api/v1/stores/:storeId/pages/:slug           page draft (sections)
    GET|POST /api/v1/stores/:storeId/media                list / upload image
    GET  /api/media/:assetId                              public image bytes
    GET  /api/health

Success `{ data, meta? }`; error `{ error: { code, message, fieldErrors?, requestId } }`.
Money is integer minor units. Sessions are opaque tokens (hash stored), httpOnly cookie.

## Deliberately not here yet

- Redis: rate limiting is in-memory and jobs live in Postgres. Add Redis when you
  run several web instances or need higher job throughput.
- Postgres row-level security as a second isolation layer behind `forStore`.
- MFA, invites/staff management, password reset.

## Next steps (matches the Engineering Pack)

1. Variants/media/collections/inventory (D02-D07): add each to `TENANT_MODELS`.
2. Cart, discounts, checkout session, payment adapter + webhooks (F01-F08).
3. Orders, fulfillment, refunds (G01-G02, F09).
4. Domains + SSL, billing/entitlements, platform admin.
5. AI gateway with store-scoped retrieval (H01-H04): reuse `requireStoreContext`.

## Known caveats

- Written without running `npm install` (sandbox had no network). Expect to fix
  small type or version issues on first `npm run typecheck`.
- No migration is committed; run `prisma migrate dev` once and commit the output.
