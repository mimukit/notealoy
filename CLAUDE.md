# CLAUDE.md

## What notealoy is

A disposable, zero-friction public notepad (reference: notepad.pw). Land on the site → you're handed a unique URL and start typing. The entire design is shaped to fit **Cloudflare free-tier limits** — the binding constraint is **100,000 Worker requests/day**, so avoid designs that turn reads or idle visits into Worker requests or D1 writes.

## Tooling

- **Package manager: pnpm** (`devEngines` pins `pnpm ^11.6.0`). Do not use npm/yarn.

## Planned stack

- **Frontend:** React SPA, client-side routing.
- **API:** Hono in a single Cloudflare Worker.
- **DB:** Cloudflare D1 (serverless SQLite). KV was rejected (1K writes/day global + eventual consistency, fatal for save-as-you-type). Durable Objects deferred.
- **Hosting:** one Worker with Static Assets — `assets.not_found_handling = "single-page-application"` (unmatched paths return the React shell so deep links / slug routes work); `run_worker_first = ["/api/*"]` (API → Hono, everything else → static assets).
- **Scheduled:** Cloudflare Cron Trigger for the daily TTL sweep.

## Testing conventions (to establish)

Test external behavior at seams, not internals. Per the plan: assert on HTTP status + body, on D1 row state after an operation, and on what the editor lets the user do — never on private functions or DB call counts. Primary seam = the Hono `fetch` handler against a local/in-memory D1 (Miniflare / wrangler test bindings). Prefer integration-style tests at the API and route seams; drop to unit tests only for pure tricky logic (slug validation, multibyte byte-length).
