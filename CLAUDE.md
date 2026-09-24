# CLAUDE.md

Rules for Claude Code in this repository. Read these before every task.

## What this project is

Mum's the Calendar: a Flutter cycle and fertility tracker for iPhone.
No personal data ever leaves the phone.

The full spec is in `docs/SPEC.md`. It is the source of truth.
If a task conflicts with the spec, stop and ask.

## Hard rules (never break these)

1. **No networking code.** Do not write or import:
    - Dart: `HttpClient`, `Socket`, `RawSocket`, `SecureSocket`, `WebSocket`, `HttpServer`, `InternetAddress`
    - Packages: `http`, `dio`, `web_socket_channel`, `url_launcher`, `google_fonts`, Firebase, any analytics or crash reporting SDK
    - Swift or Objective-C: `URLSession`, `NSURLConnection`, `NWConnection`, `CFNetwork`, `CFSocket`
2. **No new packages without asking first.** Explain what the package does and why it is needed. Wait for approval.
3. **No cloud storage.** Never use iCloud, CloudKit, Apple Health, or iCloud Keychain sync.
4. **Keep the database out of backups.** The database file and its folder must always have `NSURLIsExcludedFromBackupKey` set.
5. **Keychain items use `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`.** No other setting.
6. **No contraception claims.** Never write "safe days", "green days", "birth control", or "contraception" as a feature claim in the app, README, or App Store text.
7. **Reminder text stays generic.** Never mention periods, fertility, or dates in notification text.
8. **No external links opened by the app.** Show web addresses as plain text.

## How to work

- One feature per branch and per pull request.
- Plan first. Show the plan before writing code.
- Every feature needs tests. `features/predictions`, `core/crypto`, and `features/backup` target 100% line coverage.
- `features/predictions` and the backup encryption code in `core/crypto` must not import Flutter.
- Run before finishing any task:
    1. `flutter analyze`
    2. `tool/check_no_network.sh` (once it exists)
    3. `flutter test`
- Calendar weeks start on Sunday by default.
- Temperatures: store in °C internally, display in the user's chosen unit.
- Text in the app: plain, short, and kind. No medical advice.

## Where things go

- `lib/core/storage/`: encrypted database, tables, delete all data
- `lib/core/crypto/`: key creation, Keychain access, backup encryption
- `lib/core/security/`: app lock, app switcher cover
- `lib/features/<feature>/`: one folder per feature (calendar, day_log, predictions, history, reminders, backup, settings)
- `lib/shared/`: shared widgets, theme, text strings
- `test/`: unit tests
- `integration_test/`: full app tests with the network trap
- `docs/`: spec and project documents

## Things to ask about instead of guessing

- Anything touching storage, encryption, or backups
- Any change to `pubspec.yaml`, `ios/Podfile`, `Info.plist`, or entitlements
- Any wording shown to the user about fertility or pregnancy
