# Mum's the Calendar: Project Specification

Last updated: September 24, 2026

## Overview

Mum's the Calendar is an iPhone app for tracking a menstrual cycle and fertility, where no personal data ever leaves the phone.

The name plays on the phrase "mum's the word", meaning "keep it secret".

**Who it is for:** a person who wants to track her cycle but does not trust any company with that data.

**What she can use it for:** she picks one goal in settings, and can change it at any time.

- **Track only:** log periods and symptoms, see predicted next period.
- **Trying to conceive:** highlight the days with the highest chance of pregnancy.
- **Avoiding pregnancy:** highlight the days with the highest chance of pregnancy as days to be careful. See the Fertility section for the medical limits on this mode.

**Why it exists:** similar apps exist, but they require trust in the company behind them. This app replaces trust with proof. The user can check for herself that nothing is sent anywhere.

**Platforms:** iPhone first. Android may follow later.

**Source code:** public on GitHub.

## The privacy promise

The app makes this promise, word for word, in the app, on the App Store page, and in the GitHub README:

> This app contains no networking code and never connects to the internet. Your data stays on this phone. It leaves only if you export a backup yourself, and that backup is locked with a passphrase only you know.

Every rule in this spec exists to keep that promise true and checkable.

**What this means in practice:**

- No accounts, no sign-in, no servers.
- No analytics, crash reporting, ads, or tracking of any kind.
- No cloud sync, including Apple iCloud.
- The developer never receives any user data, so there is nothing that could be handed over to anyone.

**What the app cannot control, and says so openly:**

- Apple collects general App Store usage statistics from phones where the owner allowed it. This is about the phone, not the cycle data, and the app never touches it.
- If the phone itself is unlocked and in someone else's hands, the app lock (see On-device security) is the protection.

## Version 1 features

**Calendar**

- Month view, week starting Sunday by default (setting to start Monday).
- Each day shows: period days, predicted period days, predicted fertile window, predicted ovulation day, and a dot if anything was logged.
- Tap a day to open that day's log.

**Daily log**

- Period: none, spotting, light, medium, heavy.
- Basal body temperature (first reading on waking), in °F or °C per settings.
- Cervical mucus: dry, sticky, creamy, watery, egg white.
- Ovulation test result (home LH strip): not taken, negative, positive.
- Pregnancy test result: not taken, negative, positive.
- Intercourse: yes or no, and protected or unprotected.
- Symptoms from a fixed list (cramps, headache, bloating, breast tenderness, acne, fatigue, nausea, mood changes).
- Free text note.

**Predictions**

- Next period start date.
- Fertile window and likely ovulation day.
- Confirmed ovulation, once temperature data shows it (see Fertility section).

**Goal mode**

- Track only, Trying to conceive, or Avoiding pregnancy. Changes wording and highlighting only. The data stays the same.

**Reminders**

- Optional local reminders: period expected soon, take temperature, take ovulation test.
- Reminder text is generic ("Mum's the Calendar reminder") so nothing private shows on the lock screen.

**History and charts**

- List of past cycles with start date and length.
- Average cycle length and period length.
- Temperature chart per cycle.

**Settings**

- Goal mode, temperature unit, week start day, reminders, app lock, backup and restore, delete all data, How to verify privacy screen.

## Fertility tracking and predictions

The app shows estimates from published fertility awareness rules. It does not claim to be birth control.

**Prediction rules (version 1)**

1. **Next period:** last period start date plus the average of the last 6 cycle lengths. With fewer than 2 logged cycles, use 28 days and label it "estimate, needs more data".
2. **Predicted ovulation day:** predicted next period start minus 14 days.
3. **Fertile window:** the 5 days before predicted ovulation plus the ovulation day itself (6 days total).
4. **Confirmed ovulation (temperature shift):** 3 consecutive waking temperatures each at least 0.2 °C (0.36 °F) above the highest of the 6 readings before them. Ovulation is marked as the day before the first higher reading.
5. **Positive ovulation test:** moves predicted ovulation to 1 day after the positive test for that cycle.
6. **Irregular cycles** (lengths varying by more than 7 days): widen the fertile window shading and show a notice that predictions are less reliable.

All prediction code is plain Dart with no dependencies, fully unit tested, and each rule is explained in plain words on an in-app "How predictions work" screen.

**Medical and legal limits**

- In the US, an app that tells a user when she can have unprotected sex to prevent pregnancy is a regulated medical device. The FDA created a device type called "software application for contraception" in 2018, classed as Class II, when it cleared Natural Cycles. Meeting that requires clinical trials. ([FDA De Novo order DEN170052](https://www.accessdata.fda.gov/cdrh_docs/pdf17/DEN170052.pdf))
- So this app must not be presented as contraception. Rules for all wording in the app, App Store listing, and README:
    - Never use "safe days", "green days", "birth control", or "contraception" as a feature claim.
    - In Avoiding pregnancy mode, show the fertile window as "higher chance of pregnancy". Days outside it are not labelled as safe.
    - Show a one-time notice on first launch and in settings: "This app gives estimates for information only. It is not a medical device and is not birth control. Talk to a doctor about contraception."
- Get a short review from a lawyer before App Store release (see Open decisions).

## Platform and technology

Flutter on iOS, with the smallest possible list of dependencies, each one audited for network code.

| Need | Choice | Notes |
| --- | --- | --- |
| Framework | Flutter, stable channel | Pin the exact Flutter version in the repo |
| Language | Dart | |
| Database | Drift with `sqlcipher_flutter_libs` | Encrypted SQLite |
| Key storage | `flutter_secure_storage` | iOS Keychain |
| Backup encryption | `cryptography` package | Argon2id and AES-256-GCM |
| Reminders | `flutter_local_notifications` | Local only, no push |
| App lock | `local_auth` | Face ID, Touch ID, or device passcode |
| Charts | `fl_chart` | Drawn on device |
| Share and import files | `share_plus`, `file_picker` | Used only for user-started backup and restore |
| Fonts | Bundled font files in `assets/fonts` | Never load fonts from the internet |

**Before adding any package:**

1. Read its source for network calls.
2. Check its own dependencies the same way.
3. Record the review in `docs/DEPENDENCIES.md` (package, version, date reviewed, result).

**Not allowed:** Firebase, any analytics or crash reporting SDK, `google_fonts`, `http`, `dio`, any package that makes network calls.

## Data storage and encryption

All data lives in one encrypted database file on the phone, and that file is kept out of iCloud backups.

**Database**

- One SQLite file, encrypted with SQLCipher (AES-256).
- A random 256-bit key is created on first launch.
- The key is stored in the iOS Keychain with the setting `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`. This means:
    - The key is only readable while the phone is unlocked.
    - The key never syncs to iCloud Keychain and never moves to another phone.

**File protection**

- Set the iOS file protection level to Complete (`NSFileProtectionComplete`) on the database file. The file is unreadable while the phone is locked.

**Keep data out of iCloud backup**

- Set the "exclude from backup" flag (`NSURLIsExcludedFromBackupKey`) on the database file and its folder.
- This is the most important step. Without it, iPhone backups send a copy of the file to Apple.
- Even if the flag ever fails, the key stays behind on the phone, so a backed-up copy cannot be opened.
- Add an automated test that checks the flag is set after every app launch.

**Data tables**

| Table | Holds |
| --- | --- |
| `day_log` | One row per date: period flow, temperature, mucus, test results, intercourse, note |
| `day_symptom` | Symptoms per date |
| `cycle` | Cycle start dates, calculated from period logs |
| `settings` | Goal mode, units, week start, reminder times, app lock on or off |

**Delete all data**

- Settings button, confirmed twice.
- Deletes the database file and the Keychain key. Nothing can be recovered afterward.

## Backup and moving to a new phone

Because data never goes to iCloud, the only way to keep it when changing phones is a backup file the user makes and moves herself.

**Making a backup**

1. Settings > Backup > Create backup.
2. User enters a passphrase twice. Minimum 12 characters, with a strength meter.
3. The app turns the passphrase into a key with Argon2id, then encrypts all data with AES-256-GCM.
4. The app opens the iOS share sheet with one file named `mums-backup-YYYY-MM-DD.mumsbak`.
5. The user chooses where it goes: AirDrop, Files, a USB drive, or anywhere else. The app never picks a destination.

**Backup file contents**

- A short header: file format version, Argon2id settings, random salt, random nonce.
- The encrypted data. Without the passphrase, the file is unreadable.
- The file format is documented in `docs/BACKUP_FORMAT.md` so anyone can check it or write their own reader.

**Restoring**

1. Settings > Backup > Restore, or open a `.mumsbak` file from Files or AirDrop.
2. Enter the passphrase.
3. Choose replace current data or cancel.

**Warnings shown to the user**

- A lost passphrase means a lost backup. No one can recover it.
- If she saves the file to iCloud Drive, Apple stores the encrypted file. They still cannot read it without the passphrase.

**Reminder:** a monthly optional reminder to make a backup, off by default.

## Network policy and enforcement

The app has zero network code, and automated checks fail the build if any appears.

iOS has no setting that removes an app's internet access. So the rule is enforced in the code, in the build pipeline, and by testing the finished app.

**Check 1: forbidden code scan (every commit)**

- A script (`tool/check_no_network.sh`) scans `lib/` and `ios/` and fails if it finds:
    - Dart: `HttpClient`, `Socket`, `RawSocket`, `SecureSocket`, `WebSocket`, `HttpServer`, `InternetAddress`, or imports of `package:http`, `package:dio`, `package:web_socket_channel`, `package:url_launcher`.
    - Swift or Objective-C: `URLSession`, `NSURLConnection`, `NWConnection`, `CFNetwork`, `CFSocket`.

**Check 2: dependency allowlist (every commit)**

- `tool/allowed_packages.txt` lists every approved package and its exact version.
- CI compares `pubspec.lock` and `ios/Podfile.lock` against the list. Any new or changed package fails the build until it is reviewed and added (see Platform and technology).

**Check 3: network trap in tests (every commit)**

- Integration tests set Dart's `HttpOverrides.global` to a version that fails the test on any connection attempt.
- Tests step through every screen and feature, including backup and restore.

**Check 4: real iPhone check (every release)**

1. On a physical iPhone, turn on Settings > Privacy & Security > App Privacy Report.
2. Install the release build and use every screen and feature for 10 minutes.
3. Open App Privacy Report and confirm Mum's the Calendar lists no network activity.
4. Attach the screenshot to the GitHub release.

**Info.plist rules**

- No background modes.
- No local network permission (`NSLocalNetworkUsageDescription`) and no Bonjour services.
- No associated domains.
- No external links opened by the app. Web addresses appear as plain text the user can copy.

## How a user verifies the privacy promise

Any user can check the promise in about 5 minutes with no technical skill, using tools built into her iPhone.

These steps appear in the app (Settings > How to verify privacy) and in the README.

**Test 1: App Privacy Report (recommended)**

1. Open Settings > Privacy & Security > App Privacy Report and turn it on.
2. Use Mum's the Calendar normally for a few days.
3. Go back to App Privacy Report. Under network activity, Mum's the Calendar should not appear at all.

**Test 2: airplane mode**

1. Turn on airplane mode.
2. Use every feature. Everything works exactly the same, because nothing depends on the internet.

**Test 3: turn off its data access**

1. Settings > Apps > Mum's the Calendar. If a Cellular Data switch appears, turn it off.
2. The app keeps working exactly the same.

**Test 4: read the code (for technical users)**

- The full source code is on GitHub.
- The README links to the network checks, the dependency list, and the review notes for each dependency.

**Honest limit, stated in the app and README:**

- Apple encrypts every App Store app, so no one outside Apple can prove the App Store copy was built from the exact GitHub code.
- Tests 1 to 3 cover this gap: they check what the installed app actually does, not what the code says it does.
- Technical users can also build and install the app from source themselves with Xcode.

## On-device security

These features protect the data from someone holding the unlocked phone.

- **App lock:** optional Face ID, Touch ID, or device passcode on every app open. Off by default, offered during first setup.
- **Lock timing:** setting for lock immediately, after 1 minute, or after 5 minutes in the background.
- **App switcher cover:** when the app goes to the background, cover the screen with a plain logo screen so the app switcher preview shows no data.
- **Generic reminders:** reminder text never mentions periods, fertility, or dates.
- **No Apple Health link in version 1:** Apple Health can sync to iCloud, so the app does not read or write it.
- **No Siri, widgets, or Spotlight search results in version 1:** each of these can show app data outside the app.

## Code structure

The code is split by feature, with prediction logic and encryption kept separate from the screens so they can be tested and reviewed on their own.

```
mums-the-calendar/
  lib/
    main.dart
    core/
      storage/        encrypted database, tables, delete all data
      crypto/         key creation, Keychain access, backup encryption
      security/       app lock, app switcher cover
    features/
      calendar/       month view, day markers
      day_log/        daily log screen
      predictions/    prediction rules (plain Dart, no Flutter imports)
      history/        past cycles, averages, temperature chart
      reminders/      local notification scheduling
      backup/         create and restore backup files
      settings/       settings screens, How to verify privacy screen
    shared/           shared widgets, theme, text strings
  assets/
    fonts/            bundled font files
  test/               unit tests (predictions, crypto, backup format)
  integration_test/   full app tests with the network trap
  ios/                iOS project, Info.plist, privacy manifest
  tool/
    check_no_network.sh
    allowed_packages.txt
  docs/
    SPEC.md           this document
    PRIVACY.md        the privacy promise and how to verify it
    DEPENDENCIES.md   review record for each package
    BACKUP_FORMAT.md  backup file layout
    PREDICTIONS.md    prediction rules in plain words
    RELEASE.md        release checklist
  .claude/
    settings.json     Claude Code permissions and hooks
  .github/workflows/
    ci.yml
  CLAUDE.md           rules Claude Code reads at the start of every session
  LICENSE
  README.md
```

**Rules**

- `features/predictions` and the backup encryption code in `core/crypto` import nothing from Flutter, so they run as plain Dart tests.
- Target test coverage for `predictions`, `crypto`, and `backup`: 100% of lines.

## GitHub repository and releases

The full source is public on GitHub from the first commit, and every release is built from a tagged commit that passed all checks.

**Repository**

- Public repository named `mums-the-calendar`.
- License: GPL-3.0 (recommended). Anyone may copy and change the code, but any app they release from it must also publish its source. This stops a closed copy from using the project's name and trust. See Open decisions.
- Protect the `main` branch: changes only through pull requests with passing CI.
- Turn on GitHub Dependabot alerts (alerts only, so no package changes happen without review).

**CI (GitHub Actions, macOS runner, on every pull request)**

1. `flutter analyze`
2. `tool/check_no_network.sh`
3. Dependency allowlist check
4. Unit tests
5. Integration tests in the iOS Simulator with the network trap
6. Build the iOS app without signing, to confirm it compiles

**Release steps (written out in `docs/RELEASE.md`)**

1. All CI checks pass on `main`.
2. Tag the commit, for example `v1.0.0`.
3. Build and sign the release from that tag on the developer's Mac.
4. Run Check 4 (real iPhone App Privacy Report) from the Network policy section.
5. Submit to the App Store.
6. Create a GitHub release for the tag with: changes, the App Privacy Report screenshot, and the Flutter version used.

**README must include**

- The privacy promise, word for word.
- The How to verify steps.
- The not-birth-control notice.
- Links to `docs/PRIVACY.md`, `docs/DEPENDENCIES.md`, and `docs/PREDICTIONS.md`.

## App Store submission

The App Store listing must repeat the privacy promise exactly and must not make any contraception claim.

**Account**

- Apple Developer Program membership (paid yearly) under the developer's name or a company.

**Privacy details in App Store Connect**

- Privacy label: "Data Not Collected".
- Privacy policy web address: link to `docs/PRIVACY.md` on GitHub. App Store Connect asks every app for one.

**Privacy manifest (`ios/Runner/PrivacyInfo.xcprivacy`)**

- Declares no collected data and no tracking domains.
- Declares a reason for each "required reason" Apple system feature the app or its packages use, such as saved preferences (`UserDefaults`) and file dates. Apple rejects builds that leave these out. ([Expo guide to Apple privacy manifests](https://docs.expo.dev/guides/apple-privacy/))
- Check each package's own `PrivacyInfo.xcprivacy` and copy its declared reasons into the app's file.

**Listing text**

- Name: Mum's the Calendar.
- Subtitle idea: "Private cycle tracker".
- Description includes: the privacy promise, the How to verify steps in short form, and the not-birth-control notice.
- Category: Health & Fitness.

**Before first submission**

- Search the App Store and the US trademark database (USPTO) for the name.
- Lawyer review of the listing text and in-app wording (see Fertility section).

## Out of scope and future work

Version 1 is iPhone only and leaves out anything that would move data off the phone or show it outside the app.

**Not in version 1**

- Android, iPad, Apple Watch.
- Apple Health, Siri, widgets, Spotlight.
- Bluetooth thermometers.
- Partner sharing or syncing between two phones.
- Any cloud feature, ever.

**Android (later)**

- Android can prove the promise more strongly than iOS: an app without the `INTERNET` permission cannot connect at all, and users can see the permission list in the Play Store and in app info.
- Plan for Android release:
    - Leave out the `INTERNET` permission in `AndroidManifest.xml`, and add a CI check that fails if any package adds it back.
    - Turn off Android auto backup (`android:allowBackup="false"`).
    - Also publish on F-Droid, which builds apps from the public source itself. This lets anyone confirm the installed app matches the GitHub code.
- The backup file format stays the same, so a user can move from iPhone to Android with one backup file.

## Open decisions

- [ ] Final name and spelling: "Mum's the Calendar" or "Mum's: The Calendar".
- [ ] License: GPL-3.0 (forks must stay open) or MIT (anyone may make a closed copy).
- [ ] Keep the "Avoiding pregnancy" mode in version 1, or ship Track only and Trying to conceive first and add it after a lawyer review.
- [ ] Lowest iOS version supported.
- [ ] Free or a one-time App Store price. Avoid in-app purchases and tip jars: they require the app to talk to Apple's payment servers, which conflicts with the promise.
- [ ] Whose name the Apple Developer account is under (personal or a company).

## Glossary

| Term | Meaning |
| --- | --- |
| AES-256-GCM | A standard, widely trusted encryption method. Used for backup files. |
| Argon2id | A method that turns a passphrase into an encryption key, designed to make guessing passphrases slow. |
| Basal body temperature | Body temperature taken right after waking, before getting up. It rises slightly after ovulation. |
| CI | Continuous integration. Automatic checks GitHub runs on every code change. |
| Cervical mucus | Fluid that changes texture through the cycle. Clear and stretchy ("egg white") usually means high fertility. |
| Fertile window | The days in a cycle when pregnancy is possible, about 5 days before ovulation through ovulation day. |
| F-Droid | An Android app store that builds apps itself from public source code. |
| iOS Keychain | Apple's secure storage on the phone for passwords and keys. |
| LH test | Home ovulation test strip. Detects luteinizing hormone, which rises 1 to 2 days before ovulation. |
| Ovulation | Release of an egg, usually once per cycle. |
| Privacy manifest | A file Apple requires in every app listing what data it collects and why it uses certain system features. |
| SQLCipher | An encrypted version of the SQLite database. |
| SQLite | A small database stored as one file on the phone. |
