# Production builds and deployment

> **Status: not deployed.** Nothing in this project has been deployed to a server yet.
> This guide describes how to do it. Each step says whether it was **verified** on the
> development machine or is an **example** that hasn't yet been run against real hosting.

## 1. Deployment architecture

```
                         ┌──────────────────────────────┐
  Browser ──HTTPS──────▶ │ Static host / CDN            │  frontend/build/web
                         │ app.example.com              │  (Flutter web, any static host)
                         └──────────────┬───────────────┘
                                        │  JavaScript calls (CORS: https://app.example.com)
  Android / iOS app ──HTTPS──┐          ▼
                             │   ┌──────────────────────────────┐
                             └─▶ │ Reverse proxy (TLS)          │  Caddy or nginx
                                 │ api.example.com :443         │  terminates HTTPS
                                 └──────────────┬───────────────┘
                                                │ http://127.0.0.1:8080
                                 ┌──────────────▼───────────────┐
                                 │ Go API  (game-discovery-api) │  one static binary, systemd service
                                 │  /api/v1/*   /uploads/*      │  APP_ENV=production
                                 └───────┬──────────────┬───────┘
                                         │ TLS          │ files
                          ┌──────────────▼───┐    ┌─────▼──────────────────┐
                          │ PostgreSQL 14+   │    │ UPLOAD_DIR             │
                          │ (managed or VM)  │    │ persistent volume      │
                          │ daily backups    │    │ (backed up)            │
                          └──────────────────┘    └────────────────────────┘
```

| Piece | Recommended | Why |
|---|---|---|
| Web app | Any static host: Netlify, Cloudflare Pages, Firebase Hosting, S3 + CloudFront, or nginx | `flutter build web` output is plain files. The app uses hash URLs (`/#/search`), so no rewrite rules are needed. |
| API | A small Linux VM (1 vCPU / 1 GB is plenty to start) running the binary under systemd, behind Caddy | One self-contained binary with no runtime dependencies. Caddy obtains and renews HTTPS certificates automatically. |
| Database | Managed PostgreSQL (AWS RDS, DigitalOcean, Supabase, Neon, Azure, …) or PostgreSQL on the VM | Managed services handle backups, patching and TLS. |
| Images | A persistent disk mounted at `UPLOAD_DIR` | The API stores uploads on local disk. |

### Scaling limits to know about

Run **one API instance** at first. Two things are per-process today:

1. **Uploads** are written to local disk. Several instances would each have different
   files, so they'd need a shared volume or object storage (S3/R2) first.
2. **Login rate limits** are counted in memory, so each instance counts separately.

Sessions are stored in PostgreSQL, so sticky sessions are never needed. Once uploads move
to object storage and the limiter to Redis, the API can run as many instances as you like.

## 2. Production configuration

Copy `backend/.env.example` and set at least these values. **The API refuses to start in
production if any of them is unsafe**, listing every problem:

| Variable | Production value |
|---|---|
| `APP_ENV` | `production` |
| `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` (or `DATABASE_URL`) | your database. Use a dedicated app role (see [DATABASE.md](DATABASE.md#recommended-a-dedicated-app-role-always-in-production)). Required: production never falls back to sample data. |
| `DB_SSLMODE` | `require` or `verify-full` (`disable` is rejected) |
| `CORS_ALLOWED_ORIGINS` | your web app's exact origin(s), e.g. `https://app.example.com`. Must be `https://`; `*` and `:*` are rejected. |
| `LOG_FORMAT` | `json` |
| `UPLOAD_DIR` | a persistent, backed-up path, e.g. `/var/lib/game-discovery/uploads` |
| `PORT` | `8080` (the proxy forwards to it) |
| `TRUSTED_PROXIES` | the proxy's address, e.g. `127.0.0.1` when Caddy runs on the same VM. Without it, every client appears to come from the proxy's IP and they all share one login rate limit |

Keep `DB_PASSWORD` / `DATABASE_URL` in the host's environment or secret store, never in
git. The Flutter app needs no secrets at all; its only build setting is the public API URL.

## 3. Build

### Backend (verified)

Cross-compile static Linux binaries from Windows (PowerShell):

```powershell
cd backend
$env:GOOS="linux"; $env:GOARCH="amd64"; $env:CGO_ENABLED="0"
go build -trimpath -ldflags "-s -w" -o dist/game-discovery-api ./cmd/api
go build -trimpath -ldflags "-s -w" -o dist/game-discovery-dbtool ./cmd/dbtool
Remove-Item Env:GOOS, Env:GOARCH
```

This was verified: both binaries built (11.4 MB and 9.7 MB). They have no runtime
dependencies. The SQL migrations and seed data are embedded, so the binaries are all you copy.

### Web app (verified)

```powershell
cd frontend
flutter build web --release --dart-define=API_BASE_URL=https://api.example.com/api/v1
```

Upload the contents of `frontend/build/web/` to the static host.

- This was verified: the build succeeds, the API URL is compiled in, and no secrets appear
  in `main.dart.js`.
- Add `--no-web-resources-cdn` to bundle Flutter's rendering engine instead of loading it
  from Google's CDN. That makes the output larger (about 41 MB on disk; each browser
  downloads only one engine variant), but the app no longer depends on gstatic.com.

### Android (example, not run here)

```powershell
flutter build appbundle --release --dart-define=API_BASE_URL=https://api.example.com/api/v1   # Play Store
flutter build apk --release --dart-define=API_BASE_URL=https://api.example.com/api/v1         # direct install
```

This needs the Android SDK and a release signing key (`android/key.properties`, kept out
of git). The app id is `com.gamediscovery.game_discovery`.

### iOS (example)

`flutter build ipa --release --dart-define=...`. This requires a Mac with Xcode and an
Apple Developer account.

## 4. Deploy the API on a Linux VM (example)

```bash
# On the server, once:
sudo useradd --system --home /opt/game-discovery --shell /usr/sbin/nologin gamedisc
sudo mkdir -p /opt/game-discovery /var/lib/game-discovery/uploads /etc/game-discovery
sudo chown gamedisc: /var/lib/game-discovery/uploads
sudo install -m 600 -o root /dev/null /etc/game-discovery/api.env   # then edit: the variables from section 2

# Each release (copy the two binaries to /opt/game-discovery first):
sudo -u gamedisc env $(sudo cat /etc/game-discovery/api.env | xargs) /opt/game-discovery/game-discovery-dbtool migrate
sudo systemctl restart game-discovery-api
curl -fsS https://api.example.com/api/health
```

`/etc/systemd/system/game-discovery-api.service`:

```ini
[Unit]
Description=Game Discovery API
After=network-online.target
Wants=network-online.target

[Service]
User=gamedisc
EnvironmentFile=/etc/game-discovery/api.env
ExecStart=/opt/game-discovery/game-discovery-api
Restart=on-failure
# The API stops gracefully on SIGTERM (finishes in-flight requests within SHUTDOWN_TIMEOUT).
TimeoutStopSec=20
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/lib/game-discovery/uploads
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

`/etc/caddy/Caddyfile` (HTTPS certificates are automatic):

```
api.example.com {
    encode gzip
    request_body {
        max_size 6MB
    }
    reverse_proxy 127.0.0.1:8080
}
```

Then `sudo systemctl enable --now game-discovery-api caddy`. Create the first admin with
`game-discovery-dbtool make-admin you@example.com` after registering.

> Behind Caddy, set `TRUSTED_PROXIES=127.0.0.1` in `api.env`. The API then takes the
> client IP from `X-Forwarded-For`, but only on requests that arrive from that address,
> so clients can't spoof it. Without it, every user shares the proxy's IP and therefore
> one per-IP login limit. The per-account limit works either way.

## 5. Release checklist

- [ ] `go test -p 1 ./...` with a database configured, so the PostgreSQL tests actually
      run (see [DATABASE.md](DATABASE.md#6-running-the-database-tests)).
- [ ] `flutter analyze` and `flutter test` pass.
- [ ] Production env vars set; the API starts, which proves the safety checks passed.
- [ ] `dbtool migrate` run before the new binary starts.
- [ ] `GET /api/health` → 200 with `"database": "ok"`.
- [ ] Web build made with the production `API_BASE_URL`, and the origin is in `CORS_ALLOWED_ORIGINS`.
- [ ] First admin created with `dbtool make-admin`. (`dbtool seed`/`setup` refuse to run in
      production, so demo data can't be loaded by accident.)
- [ ] `TRUSTED_PROXIES` set to the reverse proxy's address.
- [ ] Known gaps reviewed in [QA_REPORT.md](QA_REPORT.md#open-items). In particular, the
      app's catalogue still uses bundled sample data rather than the API.
- [ ] Database backups enabled, and the `UPLOAD_DIR` volume backed up.
- [ ] Email delivery for password resets configured. Until then, reset codes are **not
      sent** in production; they're only printed to the log in development.
