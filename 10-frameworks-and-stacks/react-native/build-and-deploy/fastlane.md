# Fastlane

## What Fastlane Is

**Fastlane** is a Ruby-based automation suite for iOS and Android release pipelines. It wraps the platform tools (`xcodebuild`, `gradle`, `altool`, the Play Developer API) behind consistent **actions** and chains them into **lanes**. The alternative is a collection of bash scripts, which inevitably rot; Fastlane standardizes authentication, artifact handoff, and error reporting across iOS and Android so the same CI pipeline structure serves both platforms.

## Fastfile Structure and Lanes

A **`Fastfile`** lives under `fastlane/` at the repo root (or per-platform under `ios/fastlane` and `android/fastlane`). **Lanes** are named blocks — `lane :beta do ... end`, invoked with `fastlane beta`. Platform blocks (`platform :ios do ... end`) namespace lanes and let the same name coexist. Share logic via private lanes or helper methods in the `Fastfile`.

## iOS Workflows (Build, Sign, Upload)

A standard **iOS lane** runs `match` to fetch signing profiles, then `build_app` (gym) to compile and archive, then `upload_to_testflight` to ship. Variants tweak the scheme, configuration, and destination. For ad-hoc distribution, swap `upload_to_testflight` for `upload_to_firebase_app_distribution` or a custom S3 upload. Keep each step's output path explicit — implicit temp paths break debugging.

## Android Workflows (Assemble, Bundle, Upload)

An **Android lane** typically runs `gradle(task: "bundleRelease")` to produce an AAB, then `upload_to_play_store` for the Play Console. For APK builds (internal distribution), use `gradle(task: "assembleRelease")`. Version management runs alongside: increment `versionCode` via `google_play_track_version_codes` to avoid upload conflicts.

## match for Code Signing Management

**`match`** stores certificates and provisioning profiles in a private Git repo, encrypted with a passphrase. Every machine and CI runner fetches them identically — no more "works on my Mac". Setup: `match init` configures the repo URL, then `match development` / `match appstore` populate it. The trade-off is that the repo becomes a shared secret; treat it with the same rigor as production keys.

## supply for Play Store Metadata

**`supply`** uploads metadata (descriptions, screenshots, feature graphics) via the Play Developer API. Declare locales under `fastlane/metadata/android/<locale>/` and Fastlane syncs them to the store. This keeps copy in version control and makes changes diffable across releases.

## pilot for TestFlight

**`pilot`** (alias `upload_to_testflight`) manages TestFlight builds and testers. Lane options include `skip_waiting_for_build_processing: true` (fire-and-forget, saves CI time), `changelog` for "What to Test" notes, and `groups` to auto-distribute to a named test group. Handle the "build still processing" case — TestFlight can take 10–60 minutes to accept a build.

## deliver for App Store Connect

**`deliver`** (alias `upload_to_app_store`) uploads store metadata and screenshots, and optionally submits for review. For ongoing releases, metadata often matches the previous version — use `skip_metadata: true` and handle store text separately. Submission for review has irreversible side effects; gate it behind an explicit lane, and never auto-submit from CI on every build.

## gym and scan Lanes

**`gym`** (alias `build_app`) builds and archives an iOS app with sensible defaults. **`scan`** (alias `run_tests`) runs Xcode tests with formatted output. Both read `Gymfile` / `Scanfile` for defaults — use these to keep the Fastfile slim. `scan` integrates with JUnit/JSON output for CI reporting.

## Environment and CI Integration

Fastlane reads **environment variables** like `APP_STORE_CONNECT_API_KEY_PATH`, `MATCH_PASSWORD`, and `FASTLANE_USER`. On CI, inject these from secret stores (GitHub Actions secrets, CircleCI contexts, Bitrise secrets). Never commit `.env` files. The `dotenv` plugin is handy locally but should never be active on CI.

## Secrets Management for Fastlane

**App Store Connect API keys** (JSON files from ASC) replace session-based auth and work headlessly on CI. The match passphrase, signing certs, and keystores are all secrets — rotate them on team departures. For Play uploads, use a service account JSON with "Release manager" scope. Audit key usage via ASC/Play audit logs; unexpected activity means compromise.

## Per-Environment Lanes (dev, staging, prod)

Structure **per-environment lanes** as `lane :beta` (staging → TestFlight/Play Internal), `lane :release` (production), and `lane :alpha` (developer testing). Shared logic lives in private lanes:

```ruby
private_lane :build_ios do |options|
  # ...
end
```

Pass the environment as an option or read from env; avoid parallel lanes with slight copies of the same logic.

## Screenshot Automation (snapshot, screengrab)

**`snapshot`** drives iOS simulators to produce App Store screenshots; **`screengrab`** does the same on Android emulators. Maintaining the scripts is ongoing work but pays off on new locale launches. Alternatively, use the platform-native screenshot test frameworks (XCTest screenshots, Espresso Screenshots) and convert to Fastlane-compatible paths.

## Release Notes Automation

Generate "What's New" from git commits with `conventional_changelog` or a custom script. Feeding `changelog` to `pilot`/`supply` keeps test notes in sync with code. For App Store public-facing release notes, a human still needs to write user-friendly copy — version-control the markdown, not the store text.

## Slack / Webhook Integration

The **`slack`** action posts build results to a channel — useful for release announcements and CI visibility, but spammy when overused. Target a single release channel and one alert channel for failures. Include the build number, lane, commit, and a link to the CI job in every message.

## Debugging Fastlane Lane Failures

The **`--verbose`** flag shows the underlying commands Fastlane runs — that's the first step when a lane fails mysteriously. Many failures come from expired certificates, rotated API keys, or Xcode/Gradle version drift. Keep a runbook per lane: "if match fails, run `match nuke` and regenerate"; "if upload fails with 409, bump the build number".

## Fastlane Caching on CI

Cache `~/.cocoapods`, `ios/Pods`, and Gradle's `~/.gradle/caches` and `.gradle/wrapper`. Use Fastlane's **`setup_ci`** action on CircleCI/GitHub Actions to handle keychain setup and temporary signing config. `bundle install --path vendor/bundle` isolates the Ruby environment per project — cache `vendor/bundle` between runs.

## Self-Hosted vs Managed CI Considerations

**Managed CI** (GitHub Actions, CircleCI, Bitrise) handles macOS runners, Xcode provisioning, and Android SDKs out of the box — expensive per-minute but low overhead. **Self-hosted CI** (Mac minis in a rack) is cheaper at scale but requires Xcode upgrades, keychain hygiene, and fleet management. Fastlane runs identically on both; the choice is operational.

## See Also
- [[releasing]] — fastlane drives releases
- [[ios-build-system]] — automates iOS builds
- [[android-build-system]] — automates Android builds
