# OTA Updates

## What OTA Can and Cannot Update (JS/Assets, Not Native)

Over-the-air updates ship only the JS bundle + bundled assets (images, fonts) to already-installed apps. Native code, native dependencies, new permissions, and `Info.plist`/`AndroidManifest.xml` changes require a binary release. An OTA that would need newer native code than the installed binary must be rejected or fall back to the embedded bundle. Forgetting this is the classic RN OTA bug: "why does my update crash?".

## EAS Update Architecture

Expo's EAS Update publishes JS bundles to a CDN-backed service. Each publish is an *update* with a channel assignment (production, staging, etc.). The runtime fetches a manifest at app launch, compares its runtime version to the update's, and applies the update if compatible. Updates are incremental — only changed assets redownload. Integrates deeply with Expo Router and typed navigation.

## CodePush Architecture

Microsoft's CodePush (App Center) is the pre-EAS option. Similar model: a service stores bundles, the client SDK fetches a manifest, applies updates. CodePush supports mandatory updates (block app until applied), deferred updates (apply next launch), and per-user rollouts. App Center's sunset timeline has made EAS Update the default recommendation for new projects.

## Update Manifests and Channels

A manifest describes the update: runtime version, bundle URL, asset URLs, rollout percentage, required vs optional. Channels (`production`, `staging`, `preview`) route different builds to different manifests without rebuilding the binary. A production binary can have a staging channel enabled via build flavor for internal testing.

## Staged Rollout

Publish to 5% of users, monitor crash-free rate + error rate for an hour, expand to 25%, then 100%. EAS and CodePush both support percentage-based rollout. Without staged rollout, an OTA bug reaches 100% of active users in minutes — faster than any binary release. The whole point of OTA speed cuts both ways.

## Rollback Strategies

"Roll back" in OTA terms means republishing the previous version as the new update. Clients already on the bad update receive the correction on next fetch. EAS supports explicit rollback via `eas update:republish`; CodePush has a rollback command. Rollback is fast (minutes) but not instant — clients fetching the bad manifest first will apply the bad update.

## Signing and Verification

EAS Update can sign manifests with a private key; the embedded bundle contains the matching public key and verifies signatures on fetch. Without signing, an attacker with DNS or CDN control could inject malicious bundles. For high-security apps (fintech, health), signing is required. Rotate keys on team departures; include rotation in OTA runbook.

## Update Size Optimization (Diff Patches)

A full bundle for a modest RN app is 5–15 MB; re-downloading every update is wasteful. EAS generates binary diff patches between versions and serves only the delta — typical diffs are 10× smaller than full bundles. CodePush historically did similar. Smaller updates install faster and save user data; enable diffing by default.

## Fallback to Embedded Bundle

Every binary ships with an embedded bundle (`main.jsbundle` on iOS, `index.android.bundle` on Android). If OTA fetch fails or the downloaded update is corrupt/incompatible, the runtime falls back to embedded. Users on spotty networks launch successfully with stale-but-working JS. This is non-negotiable — never ship without an embedded bundle.

## Hermes Bytecode in OTA Updates

Hermes-enabled builds ship bytecode, not JS text; OTA updates must ship bytecode too. EAS and CodePush handle this automatically when configured. A Hermes runtime receiving a plain-JS bundle will fail to load — runtime compatibility checks prevent this. If you toggle Hermes per flavor, each flavor needs separate OTA channel logic.

## Native Version Compatibility Checks

Every update has a *runtime version* (expo: `runtimeVersion`) or CodePush *target version* (`targetBinaryVersion`). Clients only apply updates whose runtime matches their binary's. Schemes: exact-match (`1.2.3`), sdk-version (tied to Expo SDK), app-version (matches binary semver major.minor), or custom (commit SHA). Mismatch between binary and update versions is the #1 cause of silent OTA failures.

## App Store and Play Store Policy Compliance

Apple's App Store Review Guideline 3.3.2 allows OTA updates that change behavior, fix bugs, and update content — but not updates that change app purpose, add features requiring new permissions, or bypass review for fundamental UX changes. Play Store is more permissive but similar. Keep OTA changes in the spirit of the binary submitted for review.

## Emergency Kill Switches

A feature flag checked on launch can disable a just-shipped feature that's breaking in production, without an OTA bundle round-trip. EAS Update + feature flags are layered safety: flags for instant response, OTA for small code fixes, binary release for deeper fixes. Senior teams always have at least one kill switch active per major feature.

## Adoption and Error Analytics Post-Update

Track: % of active users on the latest update (adoption curve), crash-free rate by update version, error rate for key flows per update. An update with 10× the error rate of its predecessor is a rollback signal. Dashboards per update version catch regressions invisible at the binary level.

## Testing Update Flows

Test the fetch-apply-restart cycle explicitly: install binary version N, publish update N+1 to a test channel, verify the device fetches and applies. Test rollback: publish N+1, observe adoption, roll back to N, verify devices downgrade. Test offline behavior: airplane-mode launch should fall back cleanly. Test corrupted-update behavior: intercept the bundle, corrupt it, verify the runtime rejects and uses embedded.

## Handling Interrupted Updates

Download is chunked; a mid-download network failure resumes on next launch (EAS) or abandons and retries later (CodePush). Apply-on-restart is atomic — the app doesn't run with a half-applied update. Disk space exhaustion during apply falls back to embedded. The runtime's error handling here matters more than any user-visible UX.

## Multi-Channel (Staging, Prod, Canary) Strategies

Use channels as an orthogonal dimension to binary versions. A single binary can be pointed at staging for QA, canary for 1% of production, or full prod — channel-switching doesn't require rebuild. Map channels to environments, test thoroughly in staging, canary for a percentage before full prod promotion. Name channels explicitly — `prod` vs `production` confusion is a real footgun.
