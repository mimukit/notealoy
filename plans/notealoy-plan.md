# notealoy — Product Requirements & Design Spec

**Version:** v1 (MVP)
**Last updated:** 2026-06-14
**Status:** Design locked, ready for implementation

---

## 1. Overview

**notealoy** is a disposable, zero-friction personal notepad. Visiting the site
immediately gives you a unique URL you can type into and share. There is no
sign-up, no login, and no user management. Anyone with a note's URL can read and
edit it — all notes are **public by default**.

Reference product: [notepad.pw](https://notepad.pw)

**Design principles**
- Frictionless: land on the site and start typing, no account, no setup.
- Shareable: the URL *is* the note; hand it to someone and they have it.
- Disposable: notes that nobody uses go away on their own.
- Zero-cost: runs entirely within Cloudflare free-tier limits.

**v1 is explicitly single-editor.** "One person at a time" is the intended
usage; the URL is a handoff mechanism, not a live collaboration surface.
Real-time multi-editor sync is deferred (see §13).

---

## 2. Tech Stack

| Layer | Choice |
|---|---|
| Frontend | React (SPA, client-side routing) |
| API / backend | Hono running in a Cloudflare Worker |
| Database | Cloudflare D1 (serverless SQLite) |
| Hosting | Single Cloudflare Worker with Static Assets |
| Scheduled jobs | Cloudflare Cron Triggers |
| Edge protection | Cloudflare WAF rate-limiting rule (free tier: 1 rule) |

> **Storage decision:** D1 was chosen over KV. KV's free tier allows only
> **1,000 writes/day globally** and is eventually consistent — fatal for a
> save-as-you-type, shared notepad. D1 gives ~100K row writes/day and strong
> consistency. Durable Objects (the architecturally ideal primitive for live
> editing) is deferred to a future live-editing phase.

---

## 3. Cloudflare Free-Tier Constraints (the limits that shape this design)

These are the hard ceilings the entire design is built to respect:

| Resource | Free-tier limit | Relevance |
|---|---|---|
| Workers requests | 100,000 / day | **The binding constraint.** Every page load, save, and slug check is one request. |
| D1 rows written | ~100,000 / day | Save budget. Comfortable: ~5,000 editing sessions/day at ~20 saves each. |
| D1 rows read | ~5,000,000 / day | Effectively unlimited for this app. |
| D1 storage | 5 GB | Reclaimed by TTL + empty-note deletion. |
| WAF rate-limiting rules | 1 rule | Spent on writes-per-IP (see §10). |
| Cron Triggers | Included | Daily TTL sweep. |
| Bandwidth (Pages/Workers assets) | Unlimited | Re-sending full note on each save is free. |

**Why not KV:** 1,000 writes/day global + eventual consistency. Ruled out.

---

## 4. Note Identity Model

A note has **two** addresses, and separating them is the core data-model decision:

- **Canonical ID — permanent nanoid.** A 7-character base62 id (e.g. `aB3xK9p`),
  generated server-side, immutable, and used as the primary key. The URL
  `/{nanoid}` works **forever** and is the stable share target.
- **Custom slug — optional, mutable alias.** A human-readable label
  (e.g. `/grocery-list`) stored in a separate unique column pointing at the same
  note. Editing or clearing it never breaks the canonical nanoid URL.

This means "renaming a note's slug" is a cheap `UPDATE` of the alias column, not
a destroy-and-recreate — existing nanoid share links survive renames.

**Consciously accepted tradeoff:** because custom slugs are mutable and
reclaimable, a shared *slug* link (e.g. `/team-standup`) can break if the owner
renames it, and an emptied/reclaimed slug can hand a stranger a previously-shared
URL. This mirrors notepad.pw behavior and is accepted for v1.

---

## 5. URL & Routing Model

Served by a **single Worker with Static Assets**:

- `assets.not_found_handling = "single-page-application"` → any unmatched path
  returns the React `index.html` shell (makes deep links / slug routes work).
- `run_worker_first = ["/api/*"]` → API paths go to Hono; everything else falls
  through to static assets.

**Route resolution**

| Path | Handling |
|---|---|
| `/api/*` | Hono API (note CRUD, slug operations). Worker-first. |
| `/assets/*`, `/favicon.ico`, `/robots.txt`, etc. | Real static files. |
| `/`, `/{nanoid}`, `/{custom-slug}` | SPA shell → React routing → fetch note. |

**Landing behavior (`GET /`):** generate a nanoid **client-side** and use
`history.replaceState` to show `/{nanoid}` in the address bar immediately — no
server round-trip, no redirect (saves a Worker request; bare-root bot hits cost
nothing). The note **row is not written** until first content save.

**Navigating to a custom slug = open-or-create on a shared space.**
Because there is no auth, visiting `/todo` opens and lets you edit whatever
already exists there, or starts a blank note if nothing does. A user cannot tell
in advance which — this indistinguishability is inherent to the feature and
accepted.

**Reserved namespace (required):** custom slugs must not collide with app routes
or asset paths. Blocklist at minimum: `api`, `assets`, `favicon.ico`,
`robots.txt`, `_`, and any other reserved path. Enforced at slug validation.

---

## 6. Lazy Creation

**No database row exists until a note's first content save.**

- Visiting root or an unused slug generates/shows a URL but writes nothing.
- The row is `INSERT`ed only when content is first saved.
- This keeps bots, crawlers, link-unfurlers, and idle visits from spawning junk
  rows and burning the write budget.

**Consequence — unknown slug is NOT a 404.** `GET /api/note/:idOrSlug` for a
slug/id with no row returns **`200` with empty content** (a "new note"). An
unsaved nanoid, a freshly-typed custom slug, and a shared-but-never-filled link
are all the same normal "blank note" state. Returning 404 would break the core
flow.

---

## 7. Data Model

```sql
CREATE TABLE notes (
  id          TEXT    PRIMARY KEY,   -- 7-char base62 nanoid, permanent, canonical URL
  content     TEXT    NOT NULL,      -- plaintext, <= 512 KB (byte length)
  custom_slug TEXT    UNIQUE,        -- mutable alias; case-insensitive; NULL if none; not nanoid-shaped
  updated_at  INTEGER NOT NULL,      -- ms epoch; TTL anchor AND optimistic-concurrency token
  expires_at  INTEGER NOT NULL,      -- updated_at + (30d default | 365d if custom_slug set)
  pin_hash    TEXT,                  -- reserved for future private-note feature (hash only, never the PIN)
  created_at  INTEGER NOT NULL
);

CREATE UNIQUE INDEX idx_notes_custom_slug ON notes (custom_slug);
CREATE INDEX        idx_notes_expires_at  ON notes (expires_at);
```

**Notes on the schema**
- `updated_at` does double duty as the conflict token — no separate `version`
  column needed.
- `idx_notes_expires_at` makes the daily TTL sweep a range scan, not a full-table
  scan.
- `pin_hash` is reserved now (adding a nullable column later is painless, but
  reserving it keeps v1 from being a dead end). v1 never reads or writes it.

---

## 8. Save Loop & Concurrency

**Save triggers**
- **Autosave:** debounced ~1.5–2s after the user stops typing.
- **On blur** of the editor.
- **On unload:** `navigator.sendBeacon` on `beforeunload` / `visibilitychange`
  (a normal `fetch` can be killed when the tab closes; `sendBeacon` survives).

**Conflict handling — last-write-wins, guarded (warn, don't merge):**
- Client sends the `updated_at` value it loaded.
- Server save: `UPDATE notes SET ... WHERE id = ? AND updated_at = ?`.
- If 0 rows affected → another write landed first → return **`409`** → client
  shows *"This note changed in another tab — reload to see the latest."*
- v1 does **not** do real merge/CRDT. It only *detects* the clobber instead of
  doing it silently. (Real merge is the Durable Objects future, §13.)

**Known gap (accepted):** the `sendBeacon` unload write is fire-and-forget and
cannot read the server response, so it **cannot honor the version check** — the
final flush on close is unavoidably blind last-write-wins. It's the same tab's
own latest content ~99% of the time. The version guard protects mid-session
saves, not the last gasp on close.

**Realistic collision target:** not two strangers, but one person with the note
open in two tabs / phone + laptop.

---

## 9. Empty-Note Handling

When a save arrives with empty content:

| Condition | Behavior |
|---|---|
| No custom slug + emptied | **Delete the row.** Reverts to pristine lazy state, reclaims storage. An anonymous empty nanoid note is pure garbage. |
| Has custom slug + emptied | **Keep the row, preserve the slug.** The user deliberately named the space and may be clearing it to reuse it; deleting would surrender their claim. TTL (365d) eventually reaps it if truly abandoned. |

---

## 10. Retention / TTL (what makes it "disposable")

**Sliding TTL from last edit:**

| Note type | TTL since last edit |
|---|---|
| No custom slug | 30 days |
| Has custom slug | 365 days |

- `expires_at` is recomputed on each **save** based on whether `custom_slug` is set.
- **Do not refresh expiry on every read** (that turns reads into writes and
  reintroduces write amplification). Optional read-path refresh is allowed
  **at most once/day per note**: bump `expires_at` on read only if it's more than
  ~1 day stale.
- A daily **Cron Trigger** runs `DELETE FROM notes WHERE expires_at < now()`
  (batched; trivial against the write budget thanks to the `expires_at` index).
- Clearing a custom slug reverts the note to the 30-day bucket on that save.

---

## 11. Abuse Protection & Rate Limiting

The app's write endpoint is unauthenticated and writable by the whole internet.
Two threats share one mitigation strategy:

1. **Content abuse** (phishing, spam link farms, malware redirects under your
   domain) → needs a takedown path + monitoring; not fully preventable cheaply.
2. **Budget / storage exhaustion** (scripted hammering of create/save) → needs
   rate limiting.

**Controls for v1**
- **One free WAF rate-limiting rule, spent on writes-per-IP** (e.g. cap POSTs to
  `/api/note/*` per IP per minute). This runs at the **edge, before the Worker
  is invoked**, so floods it blocks never count against the 100K-requests/day
  Worker budget. This is the single highest-leverage control.
- **In-Worker rate limiting** (Cloudflare Workers rate-limit binding, in Hono)
  for finer per-endpoint caps (slug creation, availability checks).
  Tradeoff: a request blocked *in the Worker* already consumed an invocation, so
  it counts against the 100K/day budget. Edge rule = free to block; in-Worker =
  costs an invocation. Spend the free edge rule where the distinction matters
  most (writes).
- **Hard size cap: 512 KB per note**, enforced server-side on **encoded byte
  length** (not `string.length` — multibyte UTF-8 like emoji/CJK costs 2–4 bytes
  per char). Oversized writes rejected.
- **Takedown path:** an abuse-report link/email + ability to delete a note by id.
  Required for v1; bolting it on under duress is worse.

---

## 12. Content Model

- **v1 is plaintext only.** `<textarea>` in, rendered back in a
  `<textarea>`/`<pre>`. This matches notepad.pw and — critically — eliminates
  stored-XSS exposure for free (no attacker-controlled HTML is ever executed in
  another visitor's browser on your domain). React escapes by default.
- Markdown rendering is deferred (§13); it would require strict HTML
  sanitization (DOMPurify or equivalent) as a permanent, tested commitment.

---

## 13. Copy-Link Behavior

- **"Copy link" copies the custom-slug URL when a custom slug exists**, otherwise
  the nanoid URL.
- Rationale: a human-readable `/team-standup` is the entire reason to set a slug,
  and the 365-day custom-slug TTL keeps it stable through a year of inactivity.
- Accepted residual: if the owner renames the slug, the previously-copied link
  dies; an emptied/reclaimed slug can later point a stranger at the old URL.

---

## 14. SEO / Indexing

Search-engine indexing is **disabled by design** (privacy, anti-abuse, and
budget):

- **`robots.txt: Disallow: /`** — primary control. Stops well-behaved crawlers at
  the door, protecting both privacy and the Worker request budget. For random
  nanoid/slug URLs that aren't publicly linked, this alone covers the vast
  majority.
- **`X-Robots-Tag: noindex, nofollow` HTTP header** on note responses (trivial in
  Hono) — defense-in-depth for any note that gets externally linked and crawled
  anyway.
- The app itself will not appear in search results. Acceptable for a
  word-of-mouth / direct-link tool. (If a marketing landing page is ever wanted,
  carve that one path out of the disallow.)

Reasons: deleted notes could otherwise persist in Google's cache past the TTL;
indexable pages on a clean domain are the primary lure for SEO spammers; and
crawlers reading every note URL waste the 100K-requests/day budget.

---

## 15. Slug Validation Rules

A custom slug is valid only if it:
- is **lowercase-normalized** and trimmed (case-**insensitive**: `/Todo` and
  `/todo` are the same note);
- matches `[a-z0-9-]` only;
- is **3–64 characters** long;
- is **not in the reserved blocklist** (`api`, `assets`, `favicon.ico`,
  `robots.txt`, `_`, …);
- is **not nanoid-shaped** (not exactly 7 base62 chars) — this keeps the nanoid
  and custom-slug namespaces disjoint so `/api/note/:idOrSlug` lookups are
  unambiguous (resolve id-first, slug-second).

Rename to an already-taken slug → **`409`**, keep the old slug, prompt for another.

---

## 16. API Surface (derived)

> Indicative; finalize during implementation.

| Method | Path | Behavior |
|---|---|---|
| `GET` | `/api/note/:idOrSlug` | Return content + `updated_at`. Unknown → `200` empty "new note" (never 404). |
| `PUT` | `/api/note/:idOrSlug` | Save content. Lazy-creates row on first save. Optimistic check on `updated_at` → `409` on conflict. Empty content → delete-or-keep per §9. |
| `PATCH` | `/api/note/:id/slug` | Set/change/clear custom slug. Taken → `409`. |
| `DELETE` | `/api/note/:id` | Delete (takedown / manual). |

All write paths: 512 KB byte cap + rate limiting (§11).

---

## 17. Deferred (post-v1, consciously parked)

- **PIN-based private notes** — `pin_hash` column reserved now; store only a hash
  (bcrypt/scrypt), never the PIN. Intended to reduce guessable public-word
  exposure.
- **Markdown rendering** — opt-in render toggle with strict sanitization.
- **Turnstile** (Cloudflare's free CAPTCHA) on note creation — held in reserve for
  if bot-created junk becomes a problem; adds friction to the "just start typing"
  flow, so not shipped day one.
- **Durable Objects live editing** — per-note Durable Object holding state in
  memory, WebSocket sync to everyone on the URL, periodic flush to storage. The
  architecturally correct primitive for true real-time multi-editor
  collaboration (and real merge/CRDT). Replaces last-write-wins when built.
- **Diff-based saves** — only relevant if 512 KB notes saved every ~2s become
  wasteful; a Durable-Objects-era optimization.

---

## 18. Non-Goals (v1)

- No accounts, auth, or user management.
- No real-time collaboration / live multi-editor sync.
- No rich text / WYSIWYG.
- No private notes (deferred).
- No merge/conflict resolution beyond warn-and-reload.
- No search-engine discoverability.
- No backups (notes are disposable by nature; single-region D1 is acceptable).

---

## 19. Key Risks & Accepted Tradeoffs

| Area | Risk / tradeoff | Status |
|---|---|---|
| Concurrency | Two tabs / two devices can clobber via last-write-wins; only *detected* (409 warn), not merged. | Accepted for v1; DO solves later. |
| Unload save | `sendBeacon` flush can't honor version check — blind last-write. | Accepted. |
| Slug mutability | Renamed/reclaimed slug breaks previously-shared slug links. | Accepted (nanoid stays stable). |
| Public namespace | Custom slugs are a guessable, shared, public space; common words get claimed by strangers. | Accepted; PIN private notes later. |
| Abuse | No-auth public write endpoint attracts phishing/spam. | Mitigated by rate limiting + takedown; monitor. |
| Free-tier ceiling | 100K Worker requests/day is the real cap; heavy growth needs the paid plan. | Monitor; generous for launch. |
| Edge rate limiting | Only 1 free WAF rule; finer limits cost Worker invocations. | Accepted; allocated to writes. |
