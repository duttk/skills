# Default stack and setup

Working example of everything below: the `duttk/recipes` repository (private: only available when
this session has access to it). When in doubt and it is available, copy
from it (`vite.config.ts`, `playwright.config.ts`, `worker/`, `.github/workflows/ci.yml`,
`tests/e2e/fixtures.ts`, `docs/DEPLOY.md`).

## Pieces

| Piece      | Choice                                                | Notes                                                                                                   |
| ---------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Build      | Vite + TypeScript (strict)                            | `npm create vite@latest -- --template svelte-ts`                                                        |
| UI         | Svelte 5 (runes)                                      | Small bundles; Preact is the alternative if they know React                                             |
| PWA        | `vite-plugin-pwa`                                     | `registerType: 'prompt'` + an update toast; `base: './'`; hash routing                                  |
| Icons      | `@vite-pwa/assets-generator` from one SVG             | 64/192/512, maskable, apple-touch 180                                                                   |
| Local data | Dexie                                                 | Store files as **ArrayBuffer**, never Blob (see pitfalls)                                               |
| Fonts      | Fontsource packages                                   | Precache `woff2` in Workbox `globPatterns`                                                              |
| Server     | Cloudflare Worker with static assets                  | `assets.run_worker_first: ["/api/*"]`, SPA fallback                                                     |
| Database   | Cloudflare D1                                         | Migrations in `migrations/`, applied by `npm run deploy`                                                |
| Files      | Cloudflare R2                                         | Worker streams to/from R2 under `/api/…`                                                                |
| Sign-in    | Cloudflare Access (One-time PIN, allow-listed emails) | Worker verifies the Access JWT with `jose`; refuses all API calls until configured                      |
| Unit tests | Vitest + `fake-indexeddb`                             |                                                                                                         |
| E2E tests  | Playwright                                            | Chromium desktop + iPhone viewport; WebKit project in CI; real Worker via `wrangler dev` for sync tests |
| CI/CD      | GitHub Actions                                        | build/check → e2e → deploy (guarded); WebKit job informational                                          |

## Data model rules (sync-ready from day one)

- Client-generated UUID ids; `createdAt`, `updatedAt`, `deletedAt` (soft delete) on every record.
- A `dirty` flag set by Dexie hooks on local writes; sync-engine writes run in a transaction
  tagged as sync so hooks leave them alone.
- Server assigns an increasing `seq`; clients pull `seq > cursor`. Last write wins by `updatedAt`.
- UI talks only to one repository module.

## npm scripts

```json
"dev": "vite",
"build": "vite build",
"check": "svelte-check --tsconfig ./tsconfig.app.json && tsc -p tsconfig.node.json && tsc -p worker",
"test": "vitest run",
"test:e2e": "playwright test",
"format": "prettier --write .",
"format:check": "prettier --check .",
"dev:worker": "npm run build && wrangler d1 migrations apply <db> --local && wrangler dev --var ALLOW_UNAUTHENTICATED:true",
"dev:phone": "git pull --ff-only && npm install && concurrently -k \"vite --host --port 5173 --strictPort\" \"cloudflared tunnel --url http://localhost:5173\"",
"deploy": "npm run build && wrangler d1 migrations apply <db> --remote && wrangler deploy"
```

`dev:phone` needs `server.allowedHosts: ['.trycloudflare.com']` in `vite.config.ts`.

## CI (`.github/workflows/ci.yml`)

Jobs: `build` (format:check, check, test, build) · `e2e` (Chromium + local Worker) ·
`e2e-webkit` (`WEBKIT=1`, informational) · `deploy` (needs build + e2e, push to the working
branch only). The deploy job:

- skips with a notice if the `CLOUDFLARE_API_TOKEN` secret is missing or `database_id` is the
  placeholder,
- **fails** if `ACCESS_AUD` is empty (never publish without sign-in),
- runs `npm run deploy` with `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID`.

## One-time setup the user does

Verify each dashboard path against current docs before giving it (see SKILL.md).

1. Cloudflare account; `npx wrangler login`.
2. `npx wrangler d1 create <db>` → paste `database_id` into `wrangler.jsonc`. **Answer "no"** if
   wrangler offers to edit the config (it adds duplicate `remote: true` bindings).
3. `npx wrangler r2 bucket create <bucket>` (R2 may need enabling in the dashboard first). Again
   answer "no" to config edits.
4. First `npm run deploy`. If asked to register a `workers.dev` subdomain, yes: that's the URL.
   Suggest a name with nothing personal in it.
5. Cloudflare Access on the app's address: self-hosted application, **One-time PIN** login
   method only, policy _Allow → Emails →_ the user's addresses, session 1 month. Copy the
   team domain and AUD tag into `wrangler.jsonc` vars (the `kid=` parameter of the Access login
   URL is the AUD tag; the host before `/cdn-cgi` is the team domain).
6. API token (template _Edit Cloudflare Workers_ + _Account · D1 · Edit_) and the account ID
   (`npx wrangler whoami`) as GitHub **repository** secrets `CLOUDFLARE_API_TOKEN`,
   `CLOUDFLARE_ACCOUNT_ID`.
7. Commit `wrangler.jsonc` (ids and AUD aren't secrets). Push → CI deploys.
8. Phone: open the URL in Safari, sign in, Share → Add to Home Screen.

## Testing on a phone before deploy works

`npm run dev:phone` on the user's computer prints a `https://…trycloudflare.com` URL that
live-reloads. For offline and install testing use `npm run build && npx vite preview` plus
`npx cloudflared tunnel --url http://localhost:4173`.
