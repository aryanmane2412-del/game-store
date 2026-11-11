# Database setup guide

The API stores everything in **PostgreSQL** (tested with 18.4; 14 or newer is required for
the features used). Without a database the API still runs, using built-in sample games and
in-memory accounts. That's handy for a first look, but accounts vanish on restart and the
admin dashboard is disabled. Production refuses to start without a database.

## 1. Install PostgreSQL (Windows)

1. Download the installer from <https://www.postgresql.org/download/windows/> (EDB).
2. Run it with the defaults: port **5432**, components *PostgreSQL Server*, *pgAdmin* and
   *Command Line Tools*. Choose a password for the `postgres` superuser and keep it safe.
3. The tools are not on `PATH` by default. Either add `C:\Program Files\PostgreSQL\18\bin`
   to `PATH`, or call them by full path as below.
4. Check that it's running:
   ```powershell
   & "C:\Program Files\PostgreSQL\18\bin\pg_isready.exe" -h localhost -p 5432
   # localhost:5432 - accepting connections
   ```
   (Windows service name: `postgresql-x64-18`.)

macOS: `brew install postgresql@18 && brew services start postgresql@18`.
Linux: your distribution's `postgresql` package, or the official apt/yum repositories.

## 2. Create the database

### Quick (development)

Use the `postgres` superuser, so the `dbtool` can create the database for you:

```powershell
cd backend
Copy-Item .env.example .env
notepad .env          # set DB_PASSWORD=<your postgres password>
go run ./cmd/dbtool setup
```

`setup` creates the `game_discovery` database if needed, applies all migrations and loads
the sample data (23 games, 15 categories, 6 platforms, 2 demo users, 3 reviews).

### Recommended: a dedicated app role (always in production)

The API doesn't need superuser rights. Create a role that owns only its own database.
Run this as `postgres` (in pgAdmin's Query Tool or `psql -U postgres`):

```sql
CREATE ROLE game_discovery_app LOGIN PASSWORD 'choose-a-long-random-password';
CREATE DATABASE game_discovery OWNER game_discovery_app;
```

Then, in `backend/.env`:

```env
DB_USER=game_discovery_app
DB_PASSWORD=choose-a-long-random-password
DB_NAME=game_discovery
```

…and run `go run ./cmd/dbtool setup` (it finds the database already exists and just migrates
and seeds). The owner role can create the `citext` and `pg_trgm` extensions the migrations
need, because both are "trusted" extensions in PostgreSQL 13 and later.

## 3. The `dbtool` commands

Run from `backend/`. They use the same `.env` / environment as the API.

| Command | What it does |
|---|---|
| `go run ./cmd/dbtool setup` | create-db (if missing) + migrate + seed |
| `go run ./cmd/dbtool create-db` | create the database only |
| `go run ./cmd/dbtool migrate` | apply pending migrations |
| `go run ./cmd/dbtool status` | list migrations and when each was applied |
| `go run ./cmd/dbtool rollback [n]` | undo the last *n* migrations (default 1) |
| `go run ./cmd/dbtool seed` | (re)load sample data. Safe to repeat: rows are upserted, never duplicated |
| `go run ./cmd/dbtool ping` | test the connection and print the server version |
| `go run ./cmd/dbtool make-admin you@example.com` | give an existing account the admin role |

The API also warns at startup if migrations are pending.

`setup` and `seed` **refuse to run when `APP_ENV=production`**. The seed is a demo catalogue
plus demo accounts (one with the admin role), which must never reach a live database. In
production use `migrate`, then `make-admin` for your first admin.

### First admin

Registration always creates a normal user, and nobody can make themselves an admin
through the API. To create the first admin:

1. Register in the app (or `POST /api/v1/auth/register`).
2. `go run ./cmd/dbtool make-admin you@example.com`

After that, admins can promote others from the dashboard. The two seeded demo users
(`player_one`, `admin_demo`) have **no password** and can't sign in; they exist only as
sample owners of wishlists and reviews.

## 4. Schema

Migrations live in `backend/migrations/` as numbered pairs (`NNNNNN_name.up.sql` /
`.down.sql`). They are embedded in the binaries, and applied in order inside a transaction
each. The applied versions are recorded in `schema_migrations`.

| # | Migration | Creates |
|---|---|---|
| 1 | `foundation` | extensions, `set_updated_at()` trigger function, enums (`user_role`, `platform_family`, `connectivity`, `player_mode`), `developers`, `publishers`, `categories`, `platforms` (+ the 6 built-in platforms) |
| 2 | `users` | `users` (email/username are case-insensitive `citext`, `password_hash`, `role`, JSONB `settings`) |
| 3 | `games` | `games` (prices in cents, generated `is_free`, JSONB `system_requirements`, full-text `search_vector`), `game_categories`, `game_platforms` (per-platform price), `screenshots` |
| 4 | `user_library` | `wishlists`, `favourites`, `recently_viewed`, `game_reviews` (+ trigger keeping `games.review_count/review_average` in sync) |
| 5 | `auth` | `user_sessions` (SHA-256 of session tokens only), `password_resets` |
| 6 | `fk_indexes` | indexes for three foreign keys that had none (found in the QA audit) |
| 7 | `store_links` | `game_store_links` (official listings per game: Google Play, App Store, Steam, Epic, PlayStation Store, …; https only, one per store) and `games.launch_uri` (the game's own app link for Play Now; web, script and file schemes refused by a CHECK) |
| 8 | `store_link_details` | `game_store_links` rebuilt: one row per game + platform + store, with `package_id`, `product_id`, `status`, `last_verified_at` and a foreign key to `game_platforms`; `platforms.operating_system`. Existing links are converted (see below) |
| 9 | `stores` | `stores` lookup table (slug, display name, general home page) seeded with the 10 supported stores; `game_store_links.store` becomes a foreign key to it instead of a fixed CHECK list |
| 10 | `story_mode` | adds the **Story Mode** category (`story-mode`, sort order 15). Data only; rolling back deletes the category and its game links |

### Relationships and delete rules

```
developers ─┐                         ┌─< game_categories >── categories   (CASCADE both ways)
publishers ─┴─< games (RESTRICT) ─────┼─< game_platforms  >── platforms    (platform: RESTRICT)
                                      ├─< screenshots                       (CASCADE)
users ──< user_sessions, password_resets (CASCADE)
users ──< wishlists / favourites / recently_viewed / game_reviews >── games (CASCADE from both sides;
                                                                             platform_id → SET NULL)
```

- Deleting a **game** removes its prices, categories links, screenshots, wishlist entries,
  history and reviews.
- Deleting a **user** removes their sessions, reset codes, wishlist, history and reviews.
- A **developer, publisher or platform** that's still in use can't be deleted (the API
  answers 409).
- Deleting a **category** removes it from its games; the games stay.

### Integrity rules in the database (not just the app)

- Every table has a primary key: UUIDs for users, games, studios, screenshots, reviews and
  sessions; small integers for categories and platforms.
- There are 39 `CHECK` constraints, 10 `UNIQUE` constraints and 19 foreign keys. For example:
  slugs are `lowercase-with-dashes`; prices are ≥ 0 and a sale price is below the regular
  price; ratings are 0–5 (reviews 1–5); store URLs must be `https://`; system requirements
  must be a JSON object; one review per user per game.
- Money is integer cents with a 3-letter currency code, never floating point.
  `games.is_free` is a generated column, so it can't disagree with the price.
- Triggers keep every `updated_at` current and keep review counts and averages correct.

### Store links (`game_store_links`)

Official places to get a game, added by admins (migrations 7 and 8). One row per
**game + platform + store**, so a game can have Steam *and* Epic on Windows, and a Steam link for
Windows says nothing about Linux.

| Column | Type | Meaning |
|---|---|---|
| `id` | UUID, PK | the link's id (used by the admin edit/delete endpoints) |
| `game_id` | UUID → `games(id)` ON DELETE CASCADE | the game |
| `platform_id` | SMALLINT → `game_platforms(game_id, platform_id)` ON DELETE CASCADE | the platform the listing installs on. The composite foreign key means **a link can only exist for a platform the game is sold on**: removing a platform from the game removes its links |
| `store` | TEXT, CHECK in list | `google_play`, `app_store`, `mac_app_store`, `steam`, `epic`, `microsoft_store`, `gog`, `itch`, `playstation`, `official_website` (shown as "Google Play Store", "Apple App Store", …, "Official Publisher Website") |
| `url` | TEXT, CHECK `^https://…`, ≤ 500 | the official listing page. The API checks the domain, page format, redirects and file types; the database refuses anything that isn't https or contains spaces, credentials or a fragment |
| `package_id` | TEXT, nullable | Android application id (`com.example.game`); Google Play only (CHECK) |
| `product_id` | TEXT, nullable | the store's own id: App Store id, Steam app id, Epic product slug, PlayStation concept/product id |
| `status` | TEXT, CHECK, default `available` | `available`, `coming_soon` or `unavailable` (delisted/broken; hidden from players) |
| `last_verified_at` | TIMESTAMPTZ, nullable | when an admin last confirmed the listing; null for links imported by the migration from older data |
| `created_at`, `updated_at` | TIMESTAMPTZ | `updated_at` kept current by trigger |

Constraints and indexes:

- `store` → `stores(slug)` (migration 9). `stores` holds each store's display name and general
  `homepage_url` — the fallback shown when a game has no listing on a platform. Home pages are
  recorded for Google Play, the App Store, Steam, Epic and the PlayStation Store; the others are
  NULL. A test checks this table matches the store list in the Go code exactly.

- `UNIQUE (game_id, platform_id, store)` — no duplicate store links. It also serves as the index
  for both foreign keys (its first columns are `game_id` and `game_id, platform_id`).
- The operating system is **not stored twice**: it comes from the new
  `platforms.operating_system` column (`android`, `ios`, `windows`, `macos`, `linux`,
  `playstation`, `other` — CHECK constrained, filled for the six built-in platforms).
- Trusted publisher domains are the existing `developers.website_url` / `publishers.website_url`
  columns (set with `PATCH /admin/games/{id}/studios`). Official-website links must be on one of
  them.
- `games.launch_uri` (migration 7): the game's own app link for Play Now; web, script and file
  schemes are refused by a CHECK.

Migration 8 converted the data it found: each 000007 link (one per store) became one row per
matching platform of the game, with ids read from the URL; recognisable older
`game_platforms.store_url` values were imported as unverified links. `game_platforms.store_url`
and `games.official_store_url` are kept for compatibility but the app no longer reads them. The
down migration collapses the rows back to one per store.

Saving prices (`PUT /admin/games/{id}/offers` or a game edit) now **updates** existing
`game_platforms` rows instead of deleting and re-inserting them, so editing a price never deletes
that platform's store links.

### Indexes

Every foreign key has an index (verified against `pg_constraint`/`pg_index`). There are also
indexes for the app's sort orders (trending score, rating count, rating, release date),
partial indexes for featured, free and published games, a GIN full-text index
(`search_vector`), a trigram index on `title` for fuzzy search, and a GIN `jsonb_path_ops`
index on `system_requirements`.

## 5. Backups and resets

```powershell
# Backup (custom format, compressed)
& "C:\Program Files\PostgreSQL\18\bin\pg_dump.exe" -U postgres -Fc -f game_discovery.dump game_discovery
# Restore into an empty database
& "C:\Program Files\PostgreSQL\18\bin\pg_restore.exe" -U postgres -d game_discovery --clean game_discovery.dump
```

Uploaded images live on disk in `UPLOAD_DIR` (default `backend/uploads/`), not in the
database, so back that folder up too.

To start over in development: `go run ./cmd/dbtool rollback 10`, then `go run ./cmd/dbtool setup`.

## 6. Running the database tests

The PostgreSQL tests skip themselves unless a database is configured, so a plain
`go test ./...` without one is **not** a full test run. They create and wipe their own
`<DB_NAME>_test_<package>` databases, so they never touch your data. They need a role
that may create databases (e.g. `postgres`). Options:

- With `backend/.env` set up as above: `go test -p 1 ./...`
- Against a throwaway server instead (no password, nothing shared with your real one):

```bash
PG="/c/Program Files/PostgreSQL/18/bin"
"$PG/initdb.exe" -D /tmp/gdpg -U postgres -A trust -E UTF8 --no-locale
"$PG/pg_ctl.exe" -D /tmp/gdpg -o "-p 55432 -c listen_addresses=localhost" -l /tmp/gdpg.log start
"$PG/createdb.exe" -h localhost -p 55432 -U postgres gdtest
cd backend && TEST_DATABASE_URL="postgres://postgres@localhost:55432/gdtest?sslmode=disable" go test -p 1 ./...
"$PG/pg_ctl.exe" -D /tmp/gdpg stop
```

`-p 1` runs packages one at a time, because each package resets its own test database.
