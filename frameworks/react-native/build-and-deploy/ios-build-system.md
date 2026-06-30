# iOS Build System

## Xcode Projects vs Workspaces

An Xcode *project* (`.xcodeproj`) describes a single app/target graph; a *workspace* (`.xcworkspace`) aggregates multiple projects. RN uses a workspace because CocoaPods generates `Pods.xcodeproj` alongside your app project and links them via the workspace. Always open the `.xcworkspace`, never the `.xcodeproj` — opening the wrong one is the #1 cause of "pod not found" errors.

## Schemes and Build Configurations

A *scheme* is a named build recipe: which target, which configuration (Debug/Release), which executable environment. *Configurations* are named sets of build settings (Debug, Release, Staging, Production) — Xcode defaults to two, most RN apps add more. Schemes reference configurations; you can have a "Staging" scheme that uses a "Staging" config. Shared schemes (committed to Git) are what CI uses; personal schemes stay local.

## Build Phases

Each target has an ordered list of build phases: compile sources, link binary, copy bundle resources, copy frameworks, run scripts. RN adds scripts: "Bundle React Native code and images" runs Metro to bundle JS and assets into the app bundle, "Start Packager" launches Metro in dev builds. Custom script phases run during every build unless gated with input/output file checks.

## Entitlements and Capabilities

An entitlements file (`.entitlements`) declares what the app is allowed to do: Push Notifications, Keychain Sharing, App Groups, Associated Domains, HealthKit. Capabilities in Xcode's "Signing & Capabilities" tab edit this file plus Info.plist. Mismatches between entitlements and the provisioning profile's allowed entitlements cause signing failures. Production entitlements differ from development — keep them version-controlled.

## Provisioning Profiles

A provisioning profile binds a developer account, an app bundle identifier, and allowed devices/entitlements into a signed blob Xcode uses at build time. Development profiles permit debug installs on registered devices; distribution profiles permit App Store or ad-hoc builds. Profiles expire (yearly); expired CI pipelines break silently on Friday nights without proactive rotation.

## Code Signing (Developer, Distribution, Enterprise)

Apple Developer certificates sign debug/TestFlight builds; App Store Distribution certificates sign App Store submissions; Enterprise Distribution (Apple Developer Enterprise Program) signs in-house apps bypassing the store. Each has different provisioning profile types. Misuse — using enterprise certs for public apps — violates Apple's terms and results in revocation.

## App Store Connect API

ASC's REST API (requires a key ID, issuer ID, and private key) automates build uploads, metadata, TestFlight, phased release. Fastlane uses it internally. The key is account-wide with configurable scopes ("Admin", "App Manager", "Developer"); generate the narrowest scope your CI needs. Rotate on team departures.

## TestFlight Internal and External Testing

Internal: up to 100 Apple Developer team members; no review; available minutes after processing. External: up to 10,000 non-team testers; requires review (first build 24h, subsequent builds minutes); invited by email or public link. Builds expire after 90 days — automate a TestFlight sweep so old builds disappear rather than accumulate.

## dSYM Generation and Symbolication

A dSYM (`.dSYM`) contains debug symbols separated from the binary — required to symbolicate crash reports into function + line. Xcode generates dSYMs when "Debug Information Format" is set to "DWARF with dSYM File". Upload them to your crash reporter (Crashlytics, Sentry) — Fastlane's `upload_symbols_to_crashlytics` action handles this. Bitcode recompilation historically produced dSYMs at App Store submission, which complicated matching.

## Bitcode (Deprecated)

Bitcode was Apple's LLVM-IR intermediate format, uploaded to the store so they could re-optimize for future CPUs. It's deprecated as of Xcode 14 — remove the "Enable Bitcode" build setting. Third-party libraries may still ship bitcode; it's silently ignored at link time on current SDKs.

## Build Settings Inheritance (.xcconfig)

`.xcconfig` files declare build settings as plain text (`KEY = value`). Reference one from a target's configuration to keep settings outside the project file, which makes diffs and review sane. Hierarchy: project config → target config → `.xcconfig` → Xcode UI; later levels override earlier. `#include` one `.xcconfig` from another to compose.

## CocoaPods Podfile Structure

The `Podfile` declares dependencies and platform targets. RN's default `Podfile` uses `use_frameworks!` (off by default now that static linking is preferred), includes generated RN-specific config (`react-native-xcode.sh`), and iterates `install! 'cocoapods'` options. `Podfile.lock` pins exact versions — commit it; `pod install` without a lock file is non-deterministic.

## Pod Install Performance and Caching

`pod install` resolves and downloads specs every run; on CI this is slow. Cache `~/.cocoapods`, the `Pods/` directory (when appropriate), and `Podfile.lock`-based derived data. `pod install --repo-update=false` skips the slow specs-repo refresh when you know the cache is fresh. `cocoapods-cache` plugin pre-fetches pods into a hermetic cache.

## Custom Build Scripts

Run-Script build phases execute shell during build. Use cases: generating a build-time version string, copying env-specific `.plist`, invoking Codegen. Always set "Based on dependency analysis" inputs/outputs — without them, the script runs every build, which kills incremental build time. For cross-platform logic, prefer a tool invoked from both Xcode and Gradle (Node script, Ruby) over duplicating logic.

## App Thinning and On-Demand Resources

The App Store slices a submitted archive into device-specific downloads (thinning) so a user only gets binaries and resources for their device. This happens automatically. On-Demand Resources (ODR) mark asset bundles as downloadable on-demand after install; useful for games and apps with large optional content but rarely used in typical RN apps.

## Universal Binaries and Mac Catalyst

A universal binary contains multiple CPU slices (arm64, x86_64) in one Mach-O file — RN supports arm64 device + arm64 simulator (Apple Silicon) + x86_64 simulator. Mac Catalyst lets an iPad app run on macOS with some adaptation; RN support is limited and rarely worth the complexity for app makers targeting primarily mobile.

## Automatic vs Manual Signing

Automatic signing lets Xcode create and refresh profiles on demand — convenient on a dev Mac, flaky on CI because it requires interactive Apple ID auth. Manual signing uses pre-downloaded profiles via `match` or similar. For any shared build environment (CI, multi-dev teams), manual signing is the production answer.
