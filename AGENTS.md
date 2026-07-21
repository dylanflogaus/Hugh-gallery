# Hugh Gallery

A single-artist art gallery / e-commerce site built as a Cloudflare Worker (`src/worker.js`) that serves static HTML/CSS/JS assets plus a small JSON API. It uses a D1 database (`DB`, gallery inventory), a KV namespace (`MEDIA_KV`, admin image uploads), and Stripe Checkout for payments.

## Cursor Cloud specific instructions

### Services
There is one service: the Worker dev server (`npm run dev`, `wrangler dev` on port 8787). It serves the static pages, the `/api/*` endpoints, and the D1/KV bindings locally via Miniflare. There is no separate frontend/backend split.

### Running locally
- `npm run dev` first runs `scripts/sync-public.cjs` (copies `*.html`, `*.js`, `style.css`, `icons/` into `public/`) then starts `wrangler dev --port 8787`. Always edit the root-level source files (e.g. `index.html`, `admin.js`), NOT the generated copies under `public/` — `public/` is regenerated on every dev/deploy and its non-artwork files are gitignored.
- Before the app works locally you need two things that are NOT handled by the update script:
  - `.dev.vars` (gitignored): `cp .dev.vars.example .dev.vars`. It sets `ADMIN_API_SECRET` (used as the Bearer token for `/admin` login and `/api/gallery` PUT). Add `STRIPE_SECRET_KEY=sk_test_...` here to enable checkout.
  - Local D1 schema + seed data: `npm run d1:migrate:local` (i.e. `wrangler d1 migrations apply hugh-gallery --local`). Without this, `/api/gallery` returns HTTP 500. Local state lives under `.wrangler/state/` (gitignored).
- Static pages redirect `*.html` -> extensionless (e.g. `/gallery.html` 307 -> `/gallery`); this is normal `wrangler` asset `html_handling`, not an error.
- Known gotcha (from `wrangler.toml`): if pages return HTTP 500 but `/api/gallery` works, a stale asset binding is likely — stop the `wrangler dev` process and run `npm run dev` again.

### Endpoints (quick reference)
- `GET /api/gallery` — public catalog JSON (reads D1).
- `PUT /api/gallery`, `POST /api/admin/verify`, `POST /api/admin/upload-image` — require `Authorization: Bearer <ADMIN_API_SECRET>`.
- `POST /api/checkout-session` — Stripe Checkout; returns HTTP 503 unless `STRIPE_SECRET_KEY` is set (expected in local dev without a key).
- `/artwork/video/*` — served through the Worker with HTTP Range support (206 partial content).

### Lint / test / build
- There is no lint config and no automated test suite in this repo.
- "Build" for local dev is just the `sync:public` step baked into `npm run dev`. Production deploy is `npm run deploy` (`wrangler deploy`) and requires real Cloudflare credentials + remote D1/KV — not needed for local development.
