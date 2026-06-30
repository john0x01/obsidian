# RN Version Upgrades

## RN Release Cadence and Lifecycle

RN ships roughly one minor release per quarter with patches in between. Recent releases have been stabilization-heavy (0.73 → 0.74 → 0.75 → 0.76 brought bridgeless default, debugger replacement). Each version is supported for ~6 months; pre-release announcements come via the RN blog and Discord. Plan to upgrade once per release cycle unless on LTS via a fork or vendor (Expo's SDK mapping is the typical escape).

## React Version Compatibility Matrix

Each RN version pins a React version range. RN 0.75 → React 18.x; RN with concurrent features ships with React 18+. Upgrading RN typically forces a React upgrade. Libraries with React peer deps (navigation, reanimated) must align — check their compatibility matrices before bumping RN.

## rn-diff-purge and Upgrade Helper

The [Upgrade Helper](https://react-native-community.github.io/upgrade-helper/) diffs any two RN template versions. You apply the diff to your `ios/` and `android/` folders manually. `react-native upgrade` tries to automate the same, with mixed results — it doesn't know your local modifications and can clobber or miss changes. Senior advice: read the diff first, apply changes hand-picked.

## Native File Drift (ios/ and android/ Diverging)

Every RN version's template changes native files. Your `ios/` and `android/` accumulate customizations (signing config, extra schemes, custom Gradle), so merging template changes is a three-way merge with conflicts. This is the single biggest pain point of upgrades. Mitigations: minimize native customization (prefer RN APIs + Expo Config Plugins), document every native file change with a comment explaining why, keep customizations in separate files when possible (`.xcconfig`, extra Gradle files).

## Breaking Changes Per Minor

Each RN minor has 5–20 breaking changes documented in the release notes. Common types: renamed props, removed deprecated APIs, changed threading contracts, platform-version bumps. Pre-upgrade: read the full release notes for every version between your current and target. Skipping versions compounds pain — the helper shows cumulative diff, but semantic changes stack too.

## Deprecation Pipeline

RN marks APIs deprecated one version before removal. Deprecated usage emits a yellow warning in dev. These warnings are not decoration — address them before the next upgrade cycle. A codebase that ignores deprecation warnings will hit a big bang when a future version drops the API.

## Migration to the New Architecture

Turning on `RCT_NEW_ARCH_ENABLED=1` (iOS) / `newArchEnabled=true` (Android) enables Fabric + TurboModules. This requires every native dep to support the new arch — most popular libraries do as of 2024, but check each dep's README. Test thoroughly: subtle behavior differences (prop ordering, animation timing, event bubbling) can surface at runtime. Some teams run both architectures via build variants during migration.

## Expo SDK Version Pinning

Expo SDK pins a specific RN version; upgrading Expo SDK means upgrading RN. `expo upgrade` handles both plus related peer deps (reanimated, gesture handler, navigation). Bare Expo workflows can upgrade RN independently but lose some of Expo's compatibility guarantees. Sticking to SDK-aligned RN versions is the path of least resistance.

## Third-Party Library Compatibility

Every native library (Reanimated, Gesture Handler, MMKV, Navigation) must support the target RN version. Check their release notes. A library that hasn't been updated recently may not work with the latest RN — check the `peerDependencies` and the most recent release date. Replace unmaintained libs before upgrading RN, not after.

## Hermes Version Upgrades

Hermes is shipped as part of RN; each RN version bundles a specific Hermes. Hermes upgrades sometimes reveal latent JS bugs (stricter spec conformance, tighter memory) that older Hermes tolerated. Re-run test suites after upgrade; profile startup and memory — Hermes improvements often shave TTI.

## iOS / Android Minimum Version Changes

RN periodically drops support for older iOS and Android versions. iOS: RN 0.75 requires iOS 13.4+; Android: RN 0.75 requires API 23+ (Marshmallow). Raising `minSdkVersion` simplifies code but excludes users — check analytics for affected install base before upgrading. Communicate the drop to stakeholders (support, marketing) ahead of release.

## Upgrade Testing Strategy

Order: smoke-test dev build → run full unit test suite → E2E test suite → manual test of top user flows → staging build with internal distribution → staged rollout. Catch regressions early. A successful local build is necessary but nowhere near sufficient — upgrades fail at runtime in ways unit tests miss.

## Rolling Back a Failed Upgrade

`git revert` the upgrade PR if it hasn't been released. Post-release, rollback means shipping a new binary pinned to the old RN version; users on the new version stay there until they update. OTA can hotfix specific JS bugs but can't downgrade RN. Keep the pre-upgrade branch reachable and tested for a week post-release.

## Patch Management (patch-package)

`patch-package` applies custom diffs to `node_modules` contents on install — useful when a third-party package has a bug or missing feature you can't wait on. Patches live in `patches/` and commit to source control. Each patch is a maintenance burden across upgrades: after an RN upgrade, audit every patch and re-verify it still applies. More than a handful of patches usually means picking a different library or forking.

## Upgrade in Monorepos

Monorepo RN upgrades touch many apps at once if they share the RN version. Sequence: upgrade one app first, verify, propagate to siblings. Shared native libraries (a C++ TurboModule used by multiple apps) must version-bump atomically to stay compatible. Monorepos amplify upgrade pain but also make it tractable — tests run across all consumers.
