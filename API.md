# Game Discovery REST API

Base URL in development: `http://localhost:8080`. Every route is served under both
`/api/...` and `/api/v1/...` (identical responses; apps should prefer `/api/v1`).

All responses are JSON. Catalogue endpoints are public; account endpoints are
described in [Authentication](#authentication) below.

**Contents:** [Endpoints](#endpoints) · [Query parameters](#query-parameters-all-game-list-endpoints) ·
[Status codes](#status-codes) · [Response shapes](#response-shapes) · [Samples](#sample-requests-and-responses) ·
[Testing](#testing-the-api) · [Authentication](#authentication) · [Admin](#admin-dashboard-api) ·
[Health & uploads](#health-and-uploads) · [CORS & headers](#cors-and-security-headers)

## Endpoints

| Method | Path | What it returns |
|---|---|---|
| GET | `/api/games` | Games, filtered / searched / sorted / paginated by query params |
| GET | `/api/games/{id}` | One game — `{id}` is the game's UUID **or** slug |
| GET | `/api/games/featured` | Featured games (`featured=true` preset) |
| GET | `/api/games/trending` | Always sorted by `trending_score` (what's hot now) |
| GET | `/api/games/popular` | Always sorted by `rating_count` (most played overall) |
| GET | `/api/games/search?q=` | Text search — `q` is required |
| POST | `/api/games/search` | Same as above with a JSON body (for the app's search screen) |
| GET | `/api/categories` | All categories with `id` and `game_count` |
| GET | `/api/categories/{id}/games` | Games in a category — `{id}` is the numeric id **or** slug |
| GET | `/api/platforms` | All platforms with `id`, `family` and `game_count` |
| GET | `/api/platforms/{id}/games` | Games on a platform — `{id}` is the numeric id **or** slug |
| GET | `/api/health` | Service + database status |

## Query parameters (all game list endpoints)

Every list endpoint — including `/featured`, `/trending`, `/popular`, `/search`,
`/categories/{id}/games` and `/platforms/{id}/games` — accepts all of these, and
they **combine**: different parameters are AND-ed, comma-separated values inside one
parameter are OR-ed.

| Param | Values | Default |
|---|---|---|
| `q` | search text (max 100 chars); every word must match | — |
| `field` | where `q` searches: `all`, `title`, `category`, `platform`, `developer`, `publisher` | `all` |
| `category` | category slugs, e.g. `racing,shooting` (unknown slugs → 422) | — |
| `platform` | a family — `mobile`, `pc`, `console` — or any platform slug from `GET /api/platforms` (e.g. `windows`, `ps5`). Unknown values → 422 | — |
| `price` | `free`, `paid`, `discounted` (`paid` includes discounted) | — |
| `offline` / `multiplayer` / `featured` | `true` / `false` | — |
| `min_rating` | `0`–`5` | — |
| `year` | release years, e.g. `2025,2026` | — |
| `sort` | `trending`, `popular`, `rating`, `newest`, `price_low`, `price_high`, `title` | `trending` |
| `page` | 1–10000 | `1` |
| `limit` | 1–50 | `20` |

**Platform + price are checked on the same offer.** `?platform=pc&price=discounted`
means *discounted on PC*, not "on PC, and discounted somewhere". The offer that matched
is returned as `matched_offer` — show that price in the app. The top-level
`price_cents` / `is_free` fields are the game's cheapest price on any platform.

On `/categories/{id}/games` the path sets the category (a `category` param is
ignored); on `/platforms/{id}/games` the path sets the platform.

## Status codes

| Code | When |
|---|---|
| 200 | Success — including an empty list (`"data": []`) or a page past the end |
| 400 | Malformed JSON body on `POST /games/search` |
| 404 | Unknown game / category / platform, or unknown URL |
| 405 | Wrong method (e.g. `POST /api/games`); `Allow` header lists valid ones |
| 413 | JSON body too large |
| 415 | `POST` body isn't `application/json` |
| 422 | A parameter is invalid — `details` names every bad field at once |
| 500 | Unexpected server error (details are logged, never sent) |
| 503 | Database unavailable |

## Response shapes

List:   `{"data": [ ...games ], "meta": {page, limit, total, total_pages, sort, filters}}`
Single: `{"data": { ... }}`
Error:  `{"error": {"code", "message", "details"?}, "request_id"}`

## Sample requests and responses

Captured from the running server.

### Combined filters

```
GET /api/games?category=racing&platform=pc&price=discounted&sort=price_low&limit=1
```
```json
{
  "data": [
    {
      "id": "8f5d…",
      "slug": "velocity-rush",
      "title": "Velocity Rush",
      "developer": "Turbo Forge",
      "publisher": "Turbo Forge",
      "categories": ["racing", "multiplayer"],
      "release_date": "2025-11-20",
      "rating": 4.5,
      "rating_count": 240000,
      "trending_score": 92,
      "is_featured": true,
      "connectivity": "online_only",
      "player_modes": ["single_player", "online_multiplayer"],
      "offers": [
        {"platform": "android", "price_cents": 0,    "discount_price_cents": null, "currency": "USD", "store_url": null},
        {"platform": "ios",     "price_cents": 0,    "discount_price_cents": null, "currency": "USD", "store_url": null},
        {"platform": "windows", "price_cents": 1999, "discount_price_cents": 999,  "currency": "USD", "store_url": null},
        {"platform": "macos",   "price_cents": 1999, "discount_price_cents": 999,  "currency": "USD", "store_url": null}
      ],
      "price_cents": 0,
      "is_free": true,
      "currency": "USD",
      "official_store_url": null,
      "system_requirements": {
        "os": ["Windows 10/11 (64-bit)", "macOS 13 Ventura or later"],
        "storage_gb": 18,
        "minimum":     {"processor": "Intel Core i3-6100 / AMD Ryzen 3 1200", "memory_gb": 4, "graphics": "Intel UHD 620 / 2 GB VRAM"},
        "recommended": {"processor": "Intel Core i5-8400 / AMD Ryzen 5 2600", "memory_gb": 8, "graphics": "NVIDIA GTX 1050 / AMD RX 560"}
      },
      "screenshots": [ … ],
      "matched_offer": {"platform": "windows", "price_cents": 1999, "discount_price_cents": 999, "currency": "USD", "store_url": null}
    }
  ],
  "meta": {
    "page": 1, "limit": 1, "total": 2, "total_pages": 2, "sort": "price_low",
    "filters": {"category": ["racing"], "platform": ["pc"], "price": ["discounted"]}
  }
}
```

(Some fields trimmed. Free on mobile, but `matched_offer` is the discounted Windows
price because the request asked for PC + discounted.)

### Categories / platforms

```
GET /api/categories
```
```json
{"data": [
  {"id": 1, "slug": "battle-royale", "name": "Battle Royale", "tagline": "Drop in. Last one standing wins.", "description": "…", "game_count": 2},
  {"id": 2, "slug": "1v1", "name": "1v1 Games", "tagline": "Head-to-head duels of pure skill.", "description": "…", "game_count": 3}
]}
```
```
GET /api/platforms
```
```json
{"data": [
  {"id": 1, "slug": "android", "name": "Android", "family": "mobile", "game_count": 10},
  {"id": 2, "slug": "ios",     "name": "iOS",     "family": "mobile", "game_count": 9}
]}
```

### Other endpoints (summaries)

| Request | Result |
|---|---|
| `GET /api/games/featured` | 200 — 5 games, all `is_featured: true` |
| `GET /api/games/popular?limit=3` | 200 — `last-zone-royale`, `kingdom-clash-duel`, `striker-nation` (by rating_count) |
| `GET /api/games/search?q=drift` | 200 — `neon-drift` |
| `GET /api/categories/racing/games` (or `/4/games`) | 200 — 3 racing games |
| `GET /api/platforms/android/games?price=free` | 200 — 9 games free on Android |
| `GET /api/games?price=free` / `paid` / `discounted` | 200 — 10 / 15 / 7 games |

### Errors

```
GET /api/games?price=cheap&limit=500
```
```json
HTTP/1.1 422 Unprocessable Entity
{
  "error": {
    "code": "validation_failed",
    "message": "Some fields are invalid.",
    "details": {
      "limit": "must be between 1 and 50",
      "price": "contains cheap; allowed: free, paid, discounted"
    }
  },
  "request_id": "5ae19205cb0fd38f"
}
```
```
GET /api/games?category=not-real
```
```json
HTTP/1.1 422 Unprocessable Entity
{"error": {"code": "validation_failed", "message": "Some fields are invalid.",
  "details": {"category": "unknown category: not-real (see GET /api/categories)"}}, "request_id": "…"}
```
```
GET /api/games/nope
```
```json
HTTP/1.1 404 Not Found
{"error": {"code": "not_found", "message": "Game not found."}, "request_id": "49c9dc7de7177383"}
```

## Testing the API

- **VS Code REST Client / JetBrains HTTP client:** open `backend/api-requests.http` and
  click "Send Request" above any request.
- **Every endpoint at once:** start the API, then in Git Bash run
  `bash backend/scripts/smoke-test.sh` (30 checks of status code + content).
- **Automated tests:** `go test ./...` in `backend/`. The route tests run on the sample
  data; the PostgreSQL tests (including a check that SQL results exactly match the
  sample-data results for 26 filter/sort/page combinations, and every endpoint through
  the real router) run when `backend/.env` points at a database.
- **Postman / Insomnia / Thunder Client:** import the requests from `api-requests.http`
  or type them in — no auth is needed for these endpoints.

## Authentication

| Method | Path | Auth | Result |
|---|---|---|---|
| POST | `/api/v1/auth/register` | — | 201 + session. Body: `email`, `username`, `password`, optional `display_name` |
| POST | `/api/v1/auth/login` | — | 200 + session. Body: `login` (email **or** username), `password` |
| POST | `/api/v1/auth/forgot-password` | — | 202, same reply whether or not the email exists. Body: `email` |
| POST | `/api/v1/auth/reset-password` | — | 204. Body: `code`, `password`. Signs out every device |
| GET | `/api/v1/auth/me` | 🔒 | 200, the signed-in user |
| POST | `/api/v1/auth/logout` | 🔒 | 204, this token stops working immediately |
| POST | `/api/v1/auth/logout-all` | 🔒 | 204, every session of the user ends |
| POST | `/api/v1/auth/change-password` | 🔒 | 204. Body: `current_password`, `new_password`. Other devices signed out |

🔒 = send `Authorization: Bearer <token>`. Missing/invalid/expired → **401** with a
`WWW-Authenticate: Bearer` header; signed in but wrong role → **403**.

Session response (register / login):
```json
{"data": {
  "token": "gds_bfpb3ppwi61BtxuOM1L7xYFwiJqBzaDbh-dxeBan2p0",
  "token_type": "Bearer",
  "expires_at": "2026-12-01T12:04:38Z",
  "user": {"id": "935ede52-…", "email": "ada@example.com", "username": "ada", "display_name": "Ada",
           "role": "user", "bio": "", "avatar_url": null, "avatar_style": 0, "settings": {}, "created_at": "…", "updated_at": "…"}
}}
```

Errors: wrong password and unknown account both give
`401 {"error": {"code": "unauthorized", "message": "Incorrect email/username or password."}}`.
Taken email/username → `409` with `details: {"email": "is already taken"}`. Invalid fields
→ `422` naming each field. Too many attempts → `429` with a `Retry-After` header.

### How it's secured

- **Passwords** are hashed with **Argon2id** (19 MiB, 2 passes, random 16-byte salt per
  password; OWASP settings) and stored in `users.password_hash`. Plain passwords are
  never stored or logged, and no endpoint returns the hash. Hashes made with weaker
  settings are upgraded automatically at the next login.
- **Password rules:** 8–128 characters with at least one letter and one number, and not
  the same as the email or username. The Flutter forms check the same rules.
- **Tokens** are 256-bit random values. Only their SHA-256 hash is stored, in
  `user_sessions`, so a database leak exposes no usable tokens. There is no signing
  secret that could leak, in the backend or in Flutter.
- **Session lifetime:** a session ends after 30 days, or after 7 days without use
  (`AUTH_SESSION_TTL`, `AUTH_IDLE_TIMEOUT`).
- **Logout and invalidation:** logout deletes the session row, so the token is rejected
  on the very next request. Logout-all, password reset and password change end the
  user's other sessions the same way.
- **Brute force:** 5 failed logins per account per 15 minutes, and 30 auth requests per
  IP per 15 minutes, then `429`.
- **No account discovery:** login failures look the same and take the same time
  (unknown accounts are still checked against a dummy hash). Forgot-password gives
  the same reply for every email.
- **Reset codes** are single-use, expire after 30 minutes, and only the latest one works.
  There is no email service yet: with `APP_ENV=development` the code is printed in the
  API log (`DEV ONLY — password reset code`). In production the code is never logged.
- **Authorization:** registration always creates a `user`; the request can't set a role
  (unknown fields are rejected). Make an admin from the command line:
  `go run ./cmd/dbtool make-admin you@example.com`.
- **Responses** to auth requests send `Cache-Control: no-store`.

### Testing sign-in

- `go test ./internal/auth/` covers registration, login, invalid credentials, lockout,
  protected and admin endpoints, logout, idle expiry, and password reset/change. It
  runs against the in-memory store, and also against PostgreSQL when `backend/.env`
  is set.
- With the API running: `bash backend/scripts/auth-smoke-test.sh` (19 HTTP checks).
- Flutter: `flutter test test/auth_test.dart`.

## Admin dashboard API

Every route below needs an **admin** token: guests get **401**, signed-in normal users
**403**. The role is read from the database on every request, so promoting or demoting
someone takes effect immediately, even on tokens they already hold. Registration always
creates a normal user. Make the first admin with
`go run ./cmd/dbtool make-admin you@example.com`; after that, admins can promote others.
All admin routes need PostgreSQL; without it they answer 503.

| Method | Path | Result |
|---|---|---|
| GET | `/api/v1/admin/stats` | Totals (users, admins, new users in 7 days, sessions, games, published, featured, free, on sale, reviews, average review, wishlist saves), top wishlisted games, newest users, latest reviews |
| GET | `/api/v1/admin/games?q=&featured=&published=&page=&limit=` | Games table, **including hidden games** |
| POST | `/api/v1/admin/games` | Create → 201. Same fields as a game (title, slug, descriptions, developer, publisher, release_date, categories, offers, player_modes, connectivity, system_requirements JSON, age_rating, official_store_url, is_featured, is_published, …) |
| GET / PUT | `/api/v1/admin/games/{id}` | Read / replace every detail. Omitting `is_published` keeps the current visibility |
| PATCH | `/api/v1/admin/games/{id}` | `{"is_featured": true}` and/or `{"is_published": false}`, for quick feature and hide toggles |
| PUT | `/api/v1/admin/games/{id}/offers` | `{"offers": [{"platform": "windows", "price_cents": 1999, "discount_price_cents": 999}]}`. A price of 0 makes the game free there; the headline price and free/paid status are recalculated |
| GET, POST, PATCH, DELETE, PUT | `/api/v1/admin/games/{id}/store-links[/{linkID}]`, `/platforms/{platform}`, `/studios` | Official store links per platform, their availability and the trusted publisher domains — see [Store links](#store-links-get-the-game) |
| DELETE | `/api/v1/admin/games/{id}` | 204. Also removes prices, screenshots, reviews, wishlist entries and uploaded images |
| POST | `/api/v1/admin/games/{id}/cover` | multipart, file field `image` → the updated game. Replaces the old cover file |
| POST | `/api/v1/admin/games/{id}/screenshots` | multipart `image` + optional `caption` → 201 |
| PATCH / DELETE | `/api/v1/admin/games/{id}/screenshots/{sid}` | Edit `caption` / `sort_order`, or delete (the file too) |
| POST · PUT · DELETE | `/api/v1/admin/categories[/{id}]` | `name`, optional `slug`, `tagline`, `description`, `sort_order`. Deleting removes the category from its games (the games stay) |
| POST · PUT · DELETE | `/api/v1/admin/platforms[/{id}]` | `name`, optional `slug`, `family` (`mobile`/`pc`/`console`). New platforms work in game prices and in the public filters right away. Deleting a platform that still has games → **409** |
| GET | `/api/v1/admin/users?q=&page=` | Users with review and wishlist counts and last login. Password hashes are never included |
| PATCH | `/api/v1/admin/users/{id}` | `{"role": "admin"}` or `{"role": "user"}` |
| DELETE | `/api/v1/admin/users/{id}` | 204. Removes the account and ends its sessions |
| GET | `/api/v1/admin/reviews?q=&game=&page=` | Reviews with game and author |
| DELETE | `/api/v1/admin/reviews/{id}` | 204. The game's review count updates automatically |

**Rules enforced on the server**
- Admins can't change their own role or delete their own account (403), and the last admin
  can't be removed, so the dashboard can never be locked out.
- Every field is validated, and all problems are reported at once (422 with `details`).
  Duplicate slugs give 409. Unknown categories or platforms give 422.
- **Uploads:** PNG, JPEG or WebP only, judged by the file's actual bytes rather than its
  name. The file must decode as a real image, 16–8000 px per side, at most 5 MB (413 if
  larger). Files get server-generated names under `uploads/games/<game-id>/`, so uploads
  can't overwrite or escape that folder.
- Every admin change is written to the server log (`admin action`, with the admin's id).

**Flutter:** admins see "Admin dashboard" in Account settings and on the Profile screen.
The dashboard has Overview, Games, Categories, Platforms, Users and Reviews sections.
Normal users who open `/admin` see "Admins only"; guests are sent to sign in. Every
delete asks for confirmation first, and server errors are shown on the matching form
field or in a message.

**Tests:** `go test ./internal/admin/` (needs PostgreSQL, see below), `flutter test
test/admin_test.dart`, and with the API running against PostgreSQL:
`bash backend/scripts/admin-smoke-test.sh` (32 checks).

## Store links ("Get the game")

Official places to get a game, **one link per game + platform + store** (table
`game_store_links`, see [DATABASE.md](DATABASE.md#store-links-game_store_links)). Game Discovery
never hosts game files and never invents store URLs: the sample games are fictional, so they have
no links until an admin adds real ones.

### Public endpoints

| Method | Path | Returns |
|---|---|---|
| GET | `/api/v1/games/{id}/store-links` | `{game_id, slug, title, links: [...], launch_uri}` — links marked `unavailable` are left out |
| GET | `/api/v1/games/{id}/available-platforms` | one entry per platform the game is sold on: `platform`, `name`, `family`, `operating_system`, `offer` (price), `availability` (`available` / `coming_soon` / `no_store_link`) and `stores` |
| GET | `/api/v1/games/{id}/download-options?platform=android` (or `?os=android`) | the buttons for that device, best first — see below. 422 without `platform`/`os` or with an unknown one |
| GET | `/api/v1/platforms` | every platform, now with `operating_system` (`android`, `ios`, `windows`, `macos`, `linux`, `playstation`) |
| GET | `/api/v1/stores` | every store: `store`, `name`, `platforms` and its general `homepage_url` (null where none is recorded) |

`{id}` is the game's UUID or slug. `GET /api/v1/games/{id}` also embeds the same `store_links`.

A link:

```json
{
  "id": "ef7aa08e-748e-4b62-b8c9-2ea4cc893412",
  "platform": "android", "operating_system": "android",
  "store": "google_play", "store_name": "Google Play Store",
  "url": "https://play.google.com/store/apps/details?id=com.example.velocityrush",
  "package_id": "com.example.velocityrush", "product_id": null,
  "status": "available", "last_verified_at": "2026-11-02T11:45:41.406002+05:30"
}
```

Download options for Windows (live response, trimmed):

```json
{
  "platform": "windows", "available": true, "message": null,
  "options": [
    {"store": "steam", "label": "Download on Steam", "url": "https://store.steampowered.com/app/4242/",
     "app_url": "steam://store/4242", "primary": true, "status": "available"},
    {"store": "epic", "label": "Get on Epic Games", "app_url": "com.epicgames.launcher://store/p/velocity-rush", "primary": false},
    {"store": "official_website", "label": "Visit Official Website", "app_url": null, "primary": false}
  ],
  "console": null, "other_platforms": ["android", "ios", "macos"], "launch_uri": null
}
```

- Order per platform: Android → Google Play; iOS → App Store; Windows → Steam, Epic, Microsoft
  Store, GOG, itch.io; macOS → Mac App Store, Steam, Epic, GOG, itch.io; Linux → Steam, GOG,
  itch.io; PS5 → PlayStation Store; the official website last everywhere.
- `primary` is the first **available** option; `coming_soon` options are listed with the label
  "Coming soon on …" and are never primary.
- `app_url` opens the store app (try it first, fall back to `url`).
- A game not sold on the platform: `"available": false`, `"message": "This game is not available
  for your current platform."` and `other_platforms`.
- `console`: the PlayStation Store listing, offered on other devices too (buy there, it downloads
  to the PS5 — never an install button on a phone or PC).
- `launch_uri`: the game's own app link. Show **Play Now** only if the device confirms an
  installed app handles it.
- `fallbacks`: a store's **general home page**, offered only when the game is sold on this
  platform but has **no available listing there yet** (never for a store already listed, e.g.
  "coming soon"). Label `"Browse Steam"`, plus a `note` that the game's own page hasn't been added.
  Windows/macOS → Steam and Epic; Linux → Steam; Android → Google Play; iOS → App Store.
- `console_fallback`: the PlayStation Store home page for a PS5 game with no PS Store listing.

| Store | Home page used as fallback |
|---|---|
| Google Play Store | https://play.google.com/store/games |
| Apple App Store | https://www.apple.com/app-store/ |
| Steam | https://store.steampowered.com/ |
| Epic Games Store | https://store.epicgames.com/ |
| PlayStation Store | https://store.playstation.com/ |

Home pages are never stored as game links and never labelled "Download". Stores without a recorded
home page (Mac App Store, Microsoft Store, GOG, itch.io, official websites) get no fallback.

### Admin endpoints (admin role)

| Method | Path | Body / result |
|---|---|---|
| GET | `/api/v1/admin/games/{id}/store-links` | every link (all statuses), `launch_uri`, `game_platforms`, `trusted_domains` |
| POST | `/api/v1/admin/games/{id}/store-links` | `{"platform", "store", "url"?, "package_id"?, "product_id"?, "status"?}` → 201 link |
| PATCH | `/api/v1/admin/games/{id}/store-links/{linkID}` | any of `url`, `package_id`, `product_id`, `status`, `"mark_verified": true` → 200 link |
| DELETE | `/api/v1/admin/games/{id}/store-links/{linkID}` | 204 |
| PATCH | `/api/v1/admin/games/{id}/platforms/{platform}` | `{"status": "available" \| "coming_soon" \| "unavailable"}` for every link on that platform → links |
| PATCH | `/api/v1/admin/games/{id}/studios` | `{"developer_website"?, "publisher_website"?}` (https; `""` clears) → the trusted domains for official-website links |
| PUT | `/api/v1/admin/games/{id}/store-links` | `{"links": [...], "launch_uri": "mygame://play"}` replaces everything (`[]` / `""` clear) |

**Adding by id.** For Google Play (`package_id`), App Store / Mac App Store / Steam (numeric
`product_id`) the URL may be left out: the API builds the store's standard listing address from the
id you entered (`https://play.google.com/store/apps/details?id=…`, `https://apps.apple.com/app/id…`,
`https://store.steampowered.com/app/…/`). Epic, PlayStation, Microsoft, GOG, itch.io and official
websites need the URL. When both are given, the id must match the one in the URL. Ids are always
stored as read from the URL.

`last_verified_at` is set when a link is added, when its URL or id changes, and on `mark_verified`.

### Validation

| Rule | Example refused |
|---|---|
| `https://` only, no user name/password, port or `#fragment` | `http://store.steampowered.com/app/1` |
| The store's own domain — exact host, no look-alikes | `https://store.steampowered.com.evil.example/app/1` |
| A listing page in that store's format | `https://store.steampowered.com/search?term=x` |
| Only that store's known query parameters (e.g. Google Play `id`, `hl`, `gl`) | `…/app/570?ref=affiliate` |
| **No redirects:** no `url=`, `redirect=`, `next=`, `goto=` … parameters, no web address inside any parameter | `https://studio.example/out?url=https://evil.example` |
| **No path tricks:** no `//`, `..`, `%2e`, `%2f`, backslashes, IP addresses or localhost | `https://studio.example/a/%2e%2e/b` |
| **No executables or archives:** `.exe .msi .apk .ipa .dmg .pkg .deb .rpm .AppImage .zip .torrent …` | `https://studio.example/Setup.exe` |
| Official website only on the developer's or publisher's own domain (set via `/studios`; subdomains allowed) | `https://random-mirror.example/download` |
| The store must sell for the platform, and the game must be sold there | Google Play on `windows`; PlayStation for a PC-only game |
| One link per store per platform | a second Steam link for Windows → **409** |
| Ids in the right format, and consistent with the URL | `"product_id": "abc"` for Steam |

| Store (`store`) | Listing format | Platforms |
|---|---|---|
| `google_play` — Google Play Store | `play.google.com/store/apps/details?id=<package>` | Android |
| `app_store` — Apple App Store | `apps.apple.com/[cc/]app/[name/]id<digits>` | iOS |
| `mac_app_store` — Mac App Store | `apps.apple.com/[cc/]app/[name/]id<digits>` | macOS |
| `steam` — Steam | `store.steampowered.com/app/<digits>[/name]` | Windows, macOS, Linux |
| `epic` — Epic Games Store | `store.epicgames.com/[locale/]p/<slug>` | Windows, macOS |
| `microsoft_store` — Microsoft Store | `apps.microsoft.com/detail/<id>` | Windows |
| `gog` — GOG | `www.gog.com/[cc/]game/<slug>` | Windows, macOS, Linux |
| `itch` — itch.io | `<creator>.itch.io/<game>` | Windows, macOS, Linux |
| `playstation` — PlayStation Store | `store.playstation.com/<cc-lang>/(product\|concept)/<id>` | PS5 |
| `official_website` — Official Publisher Website | a page on the developer's/publisher's domain | every platform |

### How the app uses them

The Flutter game page (`screens/game_detail/get_game_section.dart`) calls
`GET /games/{slug}/store-links`, keeps only `available` links that pass the same official-domain,
redirect and executable checks on the device, and matches them **per platform** (a Steam link for
Windows isn't offered on Linux). It then shows this device's stores, tries the store app before the
web page, and shows Play Now only when an installed app handles `launch_uri`. On Android 11+ / iOS
the game's scheme must also be declared (see `AndroidManifest.xml` / `Info.plist`).

## Health and uploads

`GET /api/health` (also `/api/v1/health`) answers **200** when healthy:

```json
{"status": "ok", "service": "game-discovery-api", "version": "0.1.0", "environment": "production",
 "time": "2026-11-01T12:00:00Z", "uptime_seconds": 3600, "checks": {"api": "ok", "database": "ok"}}
```

`checks.database` is `ok`, `not_configured` (still 200), or `unavailable`. In that last case
the status is `"degraded"` and the response is **503**, so load balancers stop sending
traffic to an instance that can't reach its database. Point load-balancer and uptime checks here.

`GET /uploads/<path>` serves uploaded images (covers and screenshots) from `UPLOAD_DIR`
with a one-day cache header. Only files are served: directory paths, dotfiles and anything
outside the folder answer the usual JSON 404. Directory listings are disabled because they
would reveal every game id, including hidden games.

## CORS and security headers

- Browsers may call the API only from the origins in `CORS_ALLOWED_ORIGINS`. Development
  allows `http://localhost:*`. Production requires an explicit `https://` list. Preflight
  requests from other origins get 403.
- No cookies are used, so cross-site request forgery doesn't apply. The only credential is
  the `Authorization: Bearer` header, which browsers never attach on their own.
- Every response sends `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`,
  `Referrer-Policy: no-referrer` and an `X-Request-ID` (also echoed in error bodies, for
  support). Auth and admin responses add `Cache-Control: no-store`.
- Request bodies are capped (`MAX_BODY_BYTES`, default 1 MB; 5 MB for image uploads).
  Unknown JSON fields are rejected, and the server has read, write and idle timeouts.

Running the database tests: see [DATABASE.md](DATABASE.md#6-running-the-database-tests).
