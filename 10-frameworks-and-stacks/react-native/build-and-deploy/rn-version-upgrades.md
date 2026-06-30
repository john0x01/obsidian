# RN Version Upgrades

## RN Release Cadence and Lifecycle

RN ships roughly one **minor release** per quarter with patches in between. Recent releases have been stabilization-heavy (0.73 → 0.74 → 0.75 → 0.76 brought bridgeless-by-default and the debugger replacement). Each version is supported for ~6 months; pre-release announcements come via the RN blog and Discord. Plan to upgrade once per release cycle unless you are on LTS via a fork or vendor (Expo's SDK mapping is the typical escape).

## React Version Compatibility Matrix

Each RN version pins a **React version range**. RN 0.75 maps to React 18.x; RN with concurrent features ships with React 18+. Upgrading RN typically forces a React upgrade. Libraries with React peer deps (navigation, reanimated) must align — check their compatibility matrices before bumping RN.

## rn-diff-purge and Upgrade Helper

The [**Upgrade Helper**](https://react-native-community.github.io/upgrade-helper/) diffs any two RN template versions; you apply the diff to your `ios/` and `android/` folders manually. `react-native upgrade` tries to automate the same with mixed results — it doesn't know your local modifications and can clobber or miss changes. Senior advice: read the diff first, then apply changes hand-picked.

## Native File Drift (ios/ and android/ Diverging)

Every RN version's template changes native files. Your `ios/` and `android/` accumulate customizations (signing config, extra schemes, custom Gradle), so merging template changes becomes a three-way merge with conflicts. This **native file drift** is the single biggest pain point of upgrades. Mitigations:

- Minimize native customization (prefer RN APIs and Expo Config Plugins).
- Document every native file change with a comment explaining why.
- Keep customizations in separate files when possible (`.xcconfig`, extra Gradle files).

## Breaking Changes Per Minor

Each RN minor has 5–20 **breaking changes** documented in the release notes. Common types: renamed props, removed deprecated APIs, changed threading contracts, and platform-version bumps. Before upgrading, read the full release notes for every version between your current and target. Skipping versions compounds the pain — the helper shows the cumulative diff, but semantic changes stack too.

## Deprecation Pipeline

RN marks APIs **deprecated** one version before removal, and deprecated usage emits a yellow warning in dev. These warnings are not decoration — address them before the next upgrade cycle. A codebase that ignores deprecation warnings will hit a big-bang break when a future version drops the API.

## Migration to the New Architecture

Setting `RCT_NEW_ARCH_ENABLED=1` (iOS) / `newArchEnabled=true` (Android) enables the **New Architecture** (Fabric + TurboModules). This requires every native dep to support it — most popular libraries do as of 2024, but check each dep's README. Test thoroughly: subtle behavior differences (prop ordering, animation timing, event bubbling) can surface at runtime. Some teams run both architectures via build variants during migration.

## Expo SDK Version Pinning

The **Expo SDK** pins a specific RN version, so upgrading the Expo SDK means upgrading RN. `expo upgrade` handles both plus related peer deps (reanimated, gesture handler, navigation). Bare Expo workflows can upgrade RN independently but lose some of Expo's compatibility guarantees. Sticking to SDK-aligned RN versions is the path of least resistance.

## Third-Party Library Compatibility

Every native library (Reanimated, Gesture Handler, MMKV, Navigation) must support the target RN version — check their release notes. A library that hasn't been updated recently may not work with the latest RN; check its `peerDependencies` and most recent release date. Replace unmaintained libs before upgrading RN, not after.

## Hermes Version Upgrades

**Hermes** is shipped as part of RN; each RN version bundles a specific Hermes. Hermes upgrades sometimes reveal latent JS bugs (from stricter spec conformance or tighter memory) that older Hermes tolerated. Re-run test suites after the upgrade, and profile startup and memory — Hermes improvements often shave TTI.

## iOS / Android Minimum Version Changes

RN periodically drops support for older OS versions. For example, RN 0.75 requires iOS 13.4+ and Android API 23+ (Marshmallow). Raising **`minSdkVersion`** simplifies code but excludes users — check analytics for the affected install base before upgrading. Communicate the drop to stakeholders (support, marketing) ahead of release.

## Upgrade Testing Strategy

Test in order: smoke-test a dev build → run the full unit test suite → run the E2E test suite → manually test top user flows → staging build with internal distribution → staged rollout. Catch regressions early. A successful local build is necessary but nowhere near sufficient — upgrades fail at runtime in ways unit tests miss.

## Rolling Back a Failed Upgrade

`git revert` the upgrade PR if it hasn't been released. Post-release, **rollback** means shipping a new binary pinned to the old RN version; users on the new version stay there until they update. OTA can hotfix specific JS bugs but can't downgrade RN. Keep the pre-upgrade branch reachable and tested for a week post-release.

## Patch Management (patch-package)

**`patch-package`** applies custom diffs to `node_modules` contents on install — useful when a third-party package has a bug or missing feature you can't wait on. Patches live in `patches/` and commit to source control. Each patch is a maintenance burden across upgrades: after an RN upgrade, audit every patch and re-verify it still applies. More than a handful of patches usually means picking a different library or forking.

## Upgrade in Monorepos

**Monorepo** RN upgrades touch many apps at once if they share the RN version. Sequence: upgrade one app first, verify, then propagate to siblings. Shared native libraries (a C++ TurboModule used by multiple apps) must version-bump atomically to stay compatible. Monorepos amplify upgrade pain but also make it tractable — tests run across all consumers.

## See Also
- [[autolinking]] — upgrades change autolinking
- [[ota-updates]] — version bumps gate OTA
- [[ios-build-system]] — upgrades touch iOS build
- [[android-build-system]] — upgrades touch Android build
