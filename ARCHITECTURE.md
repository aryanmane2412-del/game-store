# Game Discovery: architecture

This describes the code **as it is today**. Planned work is marked as such. The full
endpoint reference is [API.md](API.md), the schema and database tooling are in
[DATABASE.md](DATABASE.md), and hosting is in [DEPLOYMENT.md](DEPLOYMENT.md).

## 1. The big picture

```
┌────────────────────────────┐  HTTPS + JSON  ┌───────────────────────────┐  SQL (pgx)  ┌──────────────┐
│ Flutter app                │ ─────────────▶ │ Go REST API  (cmd/api)    │ ──────────▶ │ PostgreSQL   │
│ Web · Windows · Android ·  │ ◀───────────── │ /api/v1/*  /uploads/*     │ ◀────────── │ 14+ (18 used │
│ iOS · admin dashboard      │                │ one static binary         │             │ in testing)  │
└────────────────────────────┘                └───────────────────────────┘             └──────────────┘
```

| Part | Job |
|---|---|
| **Flutter app** (`frontend/`) | One codebase for web, desktop and mobile. The admin dashboard is a section of the same app, shown only to admins. |
| **Go API** (`backend/`) | Checks sign-ins and roles, validates every input, and is the only component that talks to the database. |
| **PostgreSQL** | Stores games, categories, platforms, per-platform prices, users, sessions, wishlists and reviews. Integrity is enforced in the schema itself, not only in Go. |

### What talks to what today

> **Important:** the Flutter app does **not** load the game catalogue from the API yet.

| Feature | Data source today |
|---|---|
| Sign in / register / forgot & reset password / sign out | **API** (`/auth/*`) |
| Admin dashboard (stats, games, prices, images, categories, platforms, users, reviews) | **API** (`/admin/*`) |
| "Get the game" store buttons on a game page | **API** (`GET /games/{slug}/store-links`), with an offline message if it can't be reached |
| Home, search, filters, categories, store, platforms, game detail | **Bundled sample data** in the app (`SampleGameRepository`, `LocalGameSearchService`) |
| Wishlist, recently viewed, profile settings | **In app memory** for the current session (cleared on sign-out) |

The consequence: games that an admin creates or edits in PostgreSQL show up in the
dashboard and through the public API, but **not** in the app's browse/search screens.
The public `/games`, `/categories` and `/platforms` endpoints already support everything
those screens need (filters, search, sorting, pagination). Connecting them is the next
step; see [QA_REPORT.md](QA_REPORT.md#open-items).

## 2. Backend

### Request flow

```
request → RequestID → Logger → Recover → CORS → SecurityHeaders → router
        → [auth.Require(role)] → handler → service → repository → PostgreSQL
```

### Packages (`backend/`)

| Package | Job |
|---|---|
| `cmd/api` | Entry point. Loads config and connects to PostgreSQL. With no database configured (development only) it serves embedded sample data and keeps accounts in memory. Graceful shutdown on SIGTERM. |
| `cmd/dbtool` | `setup`, `create-db`, `migrate`, `rollback`, `status`, `seed`, `ping`, `make-admin`. `seed`/`setup` refuse to run in production. |
| `internal/config` | Reads `.env` and the environment. Refuses unsafe production settings (see [DEPLOYMENT.md §2](DEPLOYMENT.md#2-production-configuration)). |
| `internal/routes` | URL → handler table. Every route is served under both `/api` and `/api/v1`. JSON 404/405 responses. |
| `internal/middleware` | Request IDs, structured access logs, panic → JSON 500, CORS, security headers. |
| `internal/auth` | Argon2id passwords, opaque session tokens, password resets, login rate limiting, `Require(role)` middleware. |
| `internal/admin` | Admin dashboard API and image uploads. Enforces rules such as "an admin can't remove their own admin role". |
| `internal/handlers`, `internal/services` | Public catalogue endpoints and health. |
| `internal/repository` | All SQL. PostgreSQL implementations plus an in-memory catalogue for the no-database mode. |
| `internal/database` | Connection pool, migration runner, error translation (e.g. unique violation → 409). |
| `internal/models`, `internal/validation`, `internal/apperror`, `internal/httpx` | Request/response types, per-field validation, error codes, strict JSON decoding with a body-size limit. |
| `internal/seed`, `internal/seeddata` | Demo catalogue (23 games) and the loader. |
| `migrations/` | Versioned SQL (`000001`…`000010`), embedded into the binaries. |

Libraries: `jackc/pgx/v5` (PostgreSQL), `golang.org/x/crypto/argon2`, `golang.org/x/image/webp`.
Routing is Go's standard `net/http` mux. There is no ORM: queries are hand-written and parameterised.

## 3. Database design

```
developers ──< games >── publishers                 (RESTRICT: a studio with games can't be deleted)
games ──< game_categories >── categories            (CASCADE)
games ──< game_platforms  >── platforms             (price per platform; platform RESTRICT)
games ──< screenshots                               (CASCADE)
users ──< wishlists / favourites / recently_viewed / game_reviews >── games   (CASCADE both sides)
users ──< user_sessions, password_resets            (CASCADE)
```

- **UUID keys** for users, games, studios, screenshots and reviews. Small identity keys for the
  lookup tables. Composite primary keys on link tables, so duplicates are impossible.
- **Money is integer cents** plus a currency code. `is_free` is a generated column, and a CHECK
  constraint keeps discounts below the price.
- **CHECK constraints** on slugs, emails, usernames, ratings, lengths and `https://` store URLs.
  `citext` makes email and username uniqueness case-insensitive.
- **Triggers** keep `updated_at` current and keep `games.review_count / review_average` in sync with reviews.
- **Indexes** on every foreign key (verified with a catalog query), on the sort columns, and for
  full-text (`search_vector`) and trigram (`title`) search.

Details: [DATABASE.md](DATABASE.md#4-schema).

## 4. Authentication and authorisation

1. **Register** (`POST /auth/register`) always creates role `user`. Nobody can make themselves admin
   through the API. The first admin is created with `dbtool make-admin <email>`; after that, admins
   promote others from the dashboard.
2. **Passwords** are hashed with **Argon2id** (m=19 MiB, t=2, p=1, 16-byte random salt, PHC format).
   Verification compares in constant time. A login for an unknown account still runs a dummy hash,
   so response timing doesn't reveal which emails are registered. Hashes are never returned by the API.
3. **Sessions** are 256-bit random **opaque tokens** (`gds_…`), not JWTs. Only their SHA-256 hash is
   stored (`user_sessions`), so a database leak contains no usable tokens, and there is no signing key
   that could leak. The token is valid for at most 30 days, or 7 days without use.
4. The app sends `Authorization: Bearer <token>`. `auth.Require()` looks up the session.
   `auth.Require(RoleAdmin)` also checks the role and returns **401** without a valid session and
   **403** for a non-admin. Every `/admin/*` route is wrapped this way in `routes.go`.
5. **Logout** deletes the session row, so the token stops working immediately. **Logout everywhere**,
   **change password** and **reset password** end the user's other sessions.
6. **Rate limiting** (HTTP 429): 5 failed logins per account per 15 minutes, and 30 sign-in
   requests (register, login, forgot/reset password) per client IP per 15 minutes. The counters
   are kept in process memory. Behind a reverse proxy, set `TRUSTED_PROXIES` so the real client
   IP is used.
7. **Forgot password** gives the same reply whether or not the email exists. Reset codes are single-use
   and expire after 30 minutes. *No email service exists yet:* in development the code is printed in
   the API log, and in production it is not delivered at all.
8. **Flutter** stores the session with `flutter_secure_storage` (Keychain on iOS, Keystore-backed on
   Android, Credential Manager on Windows; on web it is browser storage encrypted with WebCrypto,
   which is no stronger than the browser profile itself). On sign-out or session expiry it clears the wishlist, viewing history, profile and
   settings held in memory, so the next person on the device starts clean.

## 5. Images

An admin uploads a cover or screenshot (multipart, max 5 MB). The server:
- judges the type from the **file's bytes**, not its name (PNG, JPEG or WebP only);
- decodes the image header and rejects sides below 16 px or above 8000 px;
- generates the file name itself (`games/<id>/<kind>-<random>.<ext>`), so uploads can't overwrite
  files or escape the folder.

Files are stored under `UPLOAD_DIR` and served at `/uploads/…` with `X-Content-Type-Options: nosniff`,
no directory listings and no dotfiles. Deleting a game deletes its files. Moving to object storage
(S3/R2) is needed before running more than one API instance.

## 6. Flutter app (`frontend/lib`)

```
main.dart, app.dart  entry point, MaterialApp (Material 3, light/dark)
core/        AppConfig (API_BASE_URL), responsive breakpoints, formatters, validators
theme/       colours, typography, theme, category visuals
routing/     AppRoutes + AppRouter (hash URLs on web: /#/search). /settings,
             /profile/edit and /admin need sign-in; /admin needs the admin role
models/      Game, GameCategory, UserProfile, WishlistItem, admin models, …
services/    ApiClient (HTTP + JSON errors), AuthApi, AdminApi, session storage,
             sample catalogue + local search (until the catalogue uses the API)
providers/   ChangeNotifier controllers: auth, wishlist, search, store, profile, notifications
screens/     home, search, categories, store, platforms, game detail, wishlist,
             profile/settings, notifications, auth, admin
widgets/     GameCard, CategoryCard, carousels, grids, filter panel, search input, …
```

**Responsive layout** (`core/responsive.dart`): mobile < 600 px ≤ tablet < 1024 px ≤ desktop, with
content capped at 1400 px. Widget tests render the main screens at phone, tablet and desktop sizes
and with large text, and fail on any overflow.

The admin route check in the app only decides what is shown. The **server** enforces admin
access on every `/admin/*` request.
