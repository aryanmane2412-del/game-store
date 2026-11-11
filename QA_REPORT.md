# QA report

**Date:** 2026-11-01, re-checked 2026-11-02 · **Scope:** Flutter app, Go API, PostgreSQL schema, security review ·
**Deployment status:** **not deployed.** Everything below was run on a Windows 11 development machine.

Toolchain: Go 1.26.5, Flutter 3.44.7, PostgreSQL 18.4.

## Summary

| Area | Result |
|---|---|
| Go build + `go vet` | ✅ clean |
| Go tests **with PostgreSQL** (password authentication, nothing skipped) | ✅ 81 / 81 passed, 0 skipped |
| `flutter analyze` | ✅ no issues |
| Flutter tests (widget + unit) | ✅ 251 / 251 passed |
| Live API smoke tests against PostgreSQL (`backend/scripts/*.sh`) | ✅ 81 / 81 (public 30, auth 19, admin 32) |
| Manual API probes (auth, authorisation, validation, CORS, limits) | ✅ see below |
| Migrations up → down → up, idempotent seed | ✅ |
| Data-integrity queries on the seeded database | ✅ no violations |
| Production web build (`flutter build web --release`) | ✅ builds, API URL compiled in |
| Production Linux binaries (`GOOS=linux go build` for api + dbtool) | ✅ build |
| Production safety guards (unsafe config refused, no demo seed in production) | ✅ |

> A plain `go test ./...` **without** a database skips the PostgreSQL tests and still reports
> `ok`. Run it with a database configured; see [DATABASE.md §6](DATABASE.md#6-running-the-database-tests).

## Bugs found and fixed

### 1. One account's data shown to the next account on the same device (frontend, privacy)

**Reproduced** with a failing test: Alice signs in and adds a game to her wishlist, then signs
out. Bob signs in and sees `['g01']`, Alice's wishlist. The same happened with
recently-viewed history, profile settings (e.g. "private profile") and the loaded profile.
These live in app-wide controllers that sign-out never cleared.

**Fix:** `AuthController` now clears the wishlist, history, profile and settings whenever a
signed-in session ends (sign-out, sign-out everywhere, or the server reporting the session
expired) and whenever a *different* account signs in. A guest's wishlist is still carried into
a new sign-in, and restoring the same session at launch keeps its data.
Files: `frontend/lib/providers/auth_controller.dart`, `wishlist_controller.dart`,
`profile_controller.dart`. Test: `frontend/test/account_isolation_test.dart` (5 cases).

### 2. Demo data and a demo admin account could be seeded into production (backend, hardening)

`dbtool seed` and `dbtool setup` had no environment check. Run against a production database,
they would insert 23 demo games and an `admin_demo` account with the **admin** role. That
account has no password, so it couldn't sign in today, but it doesn't belong in a live system.

**Fix:** both commands now refuse when `APP_ENV=production` and explain what to use instead.
Verified: production → refused with a message; development → seeds as before.
File: `backend/cmd/dbtool/main.go`.

### 3. Database test failed on a fresh server (test reliability)

`TestWrongPasswordGivesHelpfulErrorWithoutLeakingIt` connected to the app's main database and
expected "authentication failed". On a server where that database hasn't been created yet
(CI, or a new machine), the error is "database does not exist" and the test failed. It was
previously hidden because the PostgreSQL tests were being skipped.

**Fix:** the test connects to the always-present `postgres` database, so only the password can
cause the failure. File: `backend/internal/database/database_test.go`.

### 4. Module file out of date

`golang.org/x/image` is imported directly (WebP decoding) but `go.mod` marked it `// indirect`.
Fixed with `go mod tidy`.

### 5. Documentation contradicted the code

- `docs/ARCHITECTURE.md` described bcrypt and JWT access/refresh tokens, plus `/me/*` and
  `/admin/media` endpoints that don't exist. The code uses Argon2id and opaque server-side
  sessions. **Rewritten** from the code.
- `docs/DEPLOYMENT.md` said the API couldn't trust `X-Forwarded-For` behind a proxy and linked to
  this report, which didn't exist. `TRUSTED_PROXIES` is implemented, so the doc now explains it.
- `frontend/README.md` was the Flutter template. Replaced.

## Re-check against every request (2026-11-02)

Each request was checked against the code, the test suites and a live run (PostgreSQL → Go API →
Flutter web build in Chrome).

| Request | Status | Evidence |
|---|---|---|
| **1. QA & deployment** — test frontend, backend, database, security; fix errors; docs | ✅ | Numbers above; smoke tests 81/81; docs links all resolve. Not deployed (none claimed). |
| **2. UI redesign** — one design system for Home, Categories, Game Store | ✅ | Exact palette tokens; shared header, navigation (bottom bar < 900 px, header links above), cards, search bar, section headers; the spec's collections, titles, platform cards, price tabs and filters; hover/press animations, skeletons; layouts tested at phone, tablet and desktop sizes. |
| **3. Smart Download / Play Now** | ✅ | Device detection (also the browser's OS on web); per-platform store buttons; store app first, then the official web page; "This game is not available for your current platform." + other platforms; PS5 never installable on phone/PC; Play Now only when an installed app handles the game's link; loading, offline, failure and confirmation states. |
| **4. Backend & PostgreSQL store links** | ✅ | `game_store_links` (platform, OS via `platforms`, store, URL, package id, product id, status, last verified); `GET /games/{id}/store-links`, `/available-platforms`, `/download-options`, `/platforms`; admin add/edit/delete/availability/trusted domains; validation (official domains, listing pages, no redirects, no executables, no duplicates). |
| **5. Official store links** | ✅ | Individual listings first; the five store home pages as clearly labelled "Browse …" fallbacks only where the game is sold but has no listing; `stores` table + `GET /stores`; every button tested for the exact URL it opens. |
| **6. Premium UI upgrade** | ✅ | New palette; banner Download button (opens the official-store sheet); wishlist heart on every card; Story Mode category (frontend + migration 10); Game Store platform tabs, Genre filter, Clear filters, Featured/Popular collections; game page cover poster, Game modes, `GameScreenshotGallery`, trailer section (shown only for a known https link). `test/premium_ui_test.dart` (10 tests). |

### Fixed during the re-check

- **The admin dashboard could not manage store links** (request 4 asked for admin features; they
  existed only as API endpoints). Added **Games → More actions → Store links**: add a link by URL
  or by package / App Store / Steam id, change availability, mark verified, delete, set the
  publisher website. Server errors appear on the field. 5 widget tests; the dashboard's requests
  and the JSON it reads were checked against the live API.
- That dialog **overflowed by 203 px on phones** (the availability dropdown) — fixed.
- Stale test counts in the README and this report — updated.

Earlier during the store-link work, two migration bugs were caught by the round-trip tests and
fixed: migration 8 could not be re-applied on the same connection, and migration 9 assumed a
constraint name that differs on real databases.

## What was tested

### Backend API (live, against PostgreSQL and in no-database mode)

| Check | Result |
|---|---|
| Every public endpoint: list, featured, trending, popular, search (GET/POST), detail by UUID/slug, categories, platforms | 200 with the documented shape |
| Invalid `sort`, `limit=-5`, missing `q`, unknown category/platform | 422 with per-field `details` |
| Malformed JSON / unknown JSON field | 400 / 422 |
| 2 MB JSON body (limit 1 MB) | 413 |
| Unknown route / wrong method | JSON 404 / 405 |
| Register → login (email or username) → `/auth/me` → logout → `/auth/me` | 201 → 200 → 200 → 204 → **401** |
| Duplicate email in different case | 409 (`citext`) |
| Wrong password vs unknown account | identical 401 message |
| 6th failed login within 15 min | 429 |
| Weak new password | 422 |
| Forgot password for unknown email | 202 (same reply as for a real one) |
| `password_hash` in any response | never present |
| `/admin/*` with no token / invalid token / normal user | 401 / 401 / **403** |
| Normal user trying `PATCH /admin/users/{id}` `{"role":"admin"}` | 403 |
| Admin endpoints with no database configured | 503 with an explanation (no crash) |
| Admin flow (smoke test): stats, game CRUD, prices, publish/feature, cover + screenshot upload, categories, platforms, users, reviews, self-demotion blocked | all passed |

### Security review

| Item | Finding |
|---|---|
| **Password storage** | Argon2id (m=19 MiB, t=2, p=1), random 16-byte salt, PHC format, constant-time compare, dummy hash for unknown accounts against timing attacks, parameters bounded when parsing. ✅ |
| **Session tokens** | 256-bit random, only the SHA-256 hash is stored, deleted on logout, ended on password change/reset, 30-day absolute and 7-day idle expiry. No signing secret exists. ✅ |
| **Exposed secrets** | No credentials in source, tests or docs (one *example* token in API.md is a sample response). `.env`, build output and uploads are git-ignored. The Flutter app contains no secrets; its only setting is the public API URL. ✅ |
| **SQL injection** | All values are bind parameters (`$n`). `ORDER BY` comes from a fixed whitelist, table names from constants, LIKE input is escaped. Live probe with `' OR 1=1--` matched nothing. ✅ |
| **Unauthorised admin access** | Every `/admin/*` route is wrapped in `Require(RoleAdmin)` server-side. Registration can't set a role. Admins can't demote themselves. The app's route guard is only cosmetic. ✅ |
| **CORS** | Allow-list of exact origins (`:*` port wildcard for development only). Disallowed preflight → 403, no CORS headers for other origins. Production refuses `*`, `:*` and non-HTTPS origins at startup. ✅ |
| **Other API protection** | Body-size limit, strict JSON decoding, `nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, panic recovery returns no internals, request IDs for tracing. ✅ |
| **Uploads** | Type from magic bytes (PNG/JPEG/WebP), header decoded and size-checked, server-generated names, path traversal (`..%2f`) → 404, no directory listing, no dotfiles. ✅ |
| **Production config guard** | API refuses to start with `DB_SSLMODE=disable`, missing database, or unsafe CORS when `APP_ENV=production`. ✅ |

### Database

| Check | Result |
|---|---|
| Migrations 1–6 apply, roll back and re-apply | ✅ |
| Foreign keys without a supporting index | none (catalog query) |
| Published games with no category / no platform | 0 / 0 |
| Headline price ≠ cheapest platform price | 0 rows |
| `review_count` out of sync with reviews | 0 rows |
| Discount ≥ price, wishlist row for a non-existent user, deleting a developer that has games | all **rejected** by the database (CHECK, FK, RESTRICT) |
| Seed run twice | same counts, no duplicates |

### Frontend

The 185 Flutter tests cover every screen (home, search, categories, store, platforms, game
detail, wishlist, profile, settings, notifications, sign-in/up/forgot, admin). They also cover
navigation and deep links, search and every filter, wishlist add/remove/undo/sort/clear, the
auth flows including server and network errors, admin access gating, and **layouts at phone,
tablet and desktop widths and with large text** (they fail on any overflow).

No manual click-through in a real browser was done in this pass.

## Open items

These are not bugs in the existing code. They are unfinished integration or operational gaps,
listed by priority.

1. **No real game artwork or trailers.** No trailer links exist in the data, so the trailer
   section stays hidden. The home section is called "Recently Released" (not "Recently Added")
   because the data has release dates, not catalogue-added dates.
   **Artwork:** The 23 sample games are fictional and have no cover images or store
   listings, so cards show generated genre artwork and game pages show store home-page fallbacks.
   Real covers (admin upload) and real listings (admin store links) appear automatically once added.
2. **The app's catalogue isn't connected to the API.** Home, search, filters, categories, store and
   game detail use the bundled sample data (`SampleGameRepository`, `LocalGameSearchService`).
   Games managed in the admin dashboard therefore don't appear in the app. The API side is complete;
   the Flutter repositories need API-backed implementations.
3. **Wishlist, recently viewed and profile settings aren't saved to the server.** They last only
   while the app is open. The tables and a tested repository (`PostgresLibraryRepository`) exist,
   but there are no `/me/wishlist`, `/me/recently-viewed` or `/me/settings` endpoints yet. This
   depends on item 1, because sample game IDs (`g01`) aren't database UUIDs.
4. **No email delivery.** Password-reset codes are only printed to the log in development. In
   production a user can't reset a forgotten password.
5. **Only the web build was run here.** Android, iOS, macOS, Linux and native Windows behaviour is
   covered by tests with simulated devices; building them needs the Android SDK / Xcode / Visual
   Studio C++ tools, which this machine doesn't have.
6. **Single API instance only**: uploads go to local disk and rate-limit counters are in memory.
7. Suggested hardening: add `Content-Security-Policy: default-src 'none'; frame-ancestors 'none'`
   to API responses, and `Strict-Transport-Security` at the reverse proxy.

## How to reproduce these results

```bash
# Backend: needs a PostgreSQL role that can create databases (see DATABASE.md §6)
cd backend
go vet ./...
TEST_DATABASE_URL="postgres://postgres:<password>@localhost:5432/game_discovery?sslmode=disable" go test -count=1 ./...

# Live smoke tests (API running on :8080 with the database)
bash scripts/smoke-test.sh && bash scripts/auth-smoke-test.sh && bash scripts/admin-smoke-test.sh

# Frontend
cd ../frontend
flutter analyze
flutter test
```
