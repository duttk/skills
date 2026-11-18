# Pitfalls already hit, and how to catch them

Each entry: what went wrong, the fix, and the check that would have caught it.

## iOS Safari

**Storing Blobs in IndexedDB fails** (`UnknownError: Error preparing Blob/File data to be stored
in object store`), especially Blobs from `fetch()`. Chrome is fine, so tests passed and phones broke.

- Fix: store `ArrayBuffer` + mime type; rebuild the Blob in a Dexie `reading` hook. Read the
  bytes (`await blob.arrayBuffer()`) **outside** Dexie transactions; awaiting non-IndexedDB
  promises inside one commits it early.
- Catch: run every e2e test with the Safari-like fixture (`tests/e2e/fixtures.ts`: patches
  `IDBObjectStore.put/add` to throw on Blob values), and unit-test raw stored records via
  `table.toCollection().raw()`.

**WebP encoding unsupported:** `canvas.toBlob(cb, 'image/webp')` silently returns PNG.

- Fix: check `blob.type`; fall back to JPEG. Catch: e2e test that stubs `toBlob` to ignore WebP.

**Other iOS limits:** no install prompt (explain Share → Add to Home Screen); Web Push only for
installed apps; no `navigator.vibrate`; audio needs a user gesture first (create the
AudioContext on the "Start" tap); site data can be evicted (call `navigator.storage.persist()`
and offer backup/export); safe-area insets with `viewport-fit=cover`.

**Inputs under 16px zoom the page.** Use `font-size: 1rem` on inputs.

## Testing

**Tests that can't fail.** Break the feature once on purpose (e.g. skip an upload) and see the
test fail before trusting it.

**Playwright WebKit limits** (not app bugs): offline emulation with a service worker fails;
no `Touch` constructor. Skip only those tests in WebKit, with the reason, and keep them in
Chromium.

**Races in navigation:** after an async action that navigates, wait for the destination
(e.g. the heading) before the next `goto`. In the app, navigate _before_ refreshing state that
unmounts the current screen.

**Fake timers:** `page.clock.install()` before load; store timer end times as timestamps so they
survive the screen sleeping.

**Grep-masked failures:** `cmd | grep COMPLETED` hides a failing exit code. Check exit codes
(`npm run check >/dev/null && echo OK`).

## Sync

**Echo pushes:** a remote record `put` over an existing one has no `dirty` in its modifications,
so the hook marked it dirty and it was pushed back. Tag sync transactions and skip them in hooks.

**Clock skew:** use server `seq` for the pull cursor, never client time.

## Cloudflare

**wrangler adds config you didn't ask for.** `r2 bucket create` and `d1 create` may offer to edit
`wrangler.jsonc`; accepting added a duplicate binding with `"remote": true`, which made local dev
and CI connect to the real account ("non-interactive environment … CLOUDFLARE_API_TOKEN").
Tell users to answer no; review `wrangler.jsonc` diffs.

**Editor reformatting** (tabs) fails `format:check` in CI and blocks the deploy. Run Prettier
on the config before committing.

**Access "Failed to fetch user/group information"** or **"Cloudflare sign in restricted to
members of account":** the Access app is using an identity provider other than One-time PIN
(often the Cloudflare-account login added by the one-click workers.dev option). Enable
One-time PIN, disable other login methods on the app, keep a single _Emails_ policy.

**Secrets not visible to Actions:** must be _Repository secrets_ on the Actions page, not
Environment, Codespaces or Dependabot secrets; account ID pasted with whitespace fails
("Invalid account ID"). The deploy job's notices say which.

**Free plans may ask for a card** (R2, Zero Trust). Warn the user up front.

## Cloud session environment

- Outbound hosts are allow-listed. If a host is blocked (`000`/`fetch failed`), propose
  adding the exact host with the environment tools; never wildcards.
- Node's built-in `fetch` ignores the proxy: run Node scripts that fetch with
  `NODE_USE_ENV_PROXY=1`.
- Playwright browsers other than the preinstalled Chromium can't be downloaded here; run
  WebKit in CI.
