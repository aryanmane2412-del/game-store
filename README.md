# Game Discovery: Flutter app

The user-facing app and the admin dashboard, from one codebase (web, Windows, Android, iOS).
See the [root README](../README.md) for the full setup and
[docs/ARCHITECTURE.md](../docs/ARCHITECTURE.md#6-flutter-app-frontendlib) for the code layout.

## Run

```powershell
flutter pub get
flutter run -d chrome --no-web-resources-cdn   # or: -d windows, -d <android-device-id>
```

The app expects the API at `http://localhost:8080/api/v1`. To use another address, pass it
at build time (it is compiled in, not read at runtime):

```powershell
# One value
flutter run -d chrome --dart-define=API_BASE_URL=http://localhost:8080/api/v1

# Or from a file (copy dart_defines.example.json → dart_defines.json, which is git-ignored)
flutter run -d chrome --dart-define-from-file=dart_defines.json
```

| Where the app runs | `API_BASE_URL` |
|---|---|
| Chrome / Windows on the same PC as the API | `http://localhost:8080/api/v1` (default) |
| Android emulator | `http://10.0.2.2:8080/api/v1` |
| Physical phone on the same Wi-Fi | `http://<your PC's LAN IP>:8080/api/v1`, and allow port 8080 through Windows Firewall |

The app holds **no secrets**. Never put database credentials here.

## Test

```powershell
flutter analyze
flutter test                     # 251 tests: every screen, navigation, search, filters,
                                 # wishlist, auth flows, admin gating, responsive layouts
flutter test test/search_test.dart   # a single file
```

Tests use fakes (`test/fake_auth_api.dart`, `test/fake_admin_api.dart`) and need no running API.

## Production build

```powershell
flutter build web --release --dart-define=API_BASE_URL=https://api.yourdomain.com/api/v1
# → build/web/  (static files for any static host)
```

The web app's origin (e.g. `https://app.yourdomain.com`) must be listed in the API's
`CORS_ALLOWED_ORIGINS`. Android/iOS builds: [docs/DEPLOYMENT.md](../docs/DEPLOYMENT.md#3-build).

## Current limitations

Sign-in and the admin dashboard use the API. **The game catalogue (home, search, store,
categories, detail) still comes from bundled sample data**, and the wishlist lasts only while
the app is open. See [docs/QA_REPORT.md](../docs/QA_REPORT.md#open-items).
