# Android Build System

## Gradle Project Structure

The **Gradle build** is defined by three files: a root `build.gradle`, a `settings.gradle`, and a module-level `app/build.gradle`. `settings.gradle` lists the included modules; in RN this is where autolinking hooks add native dependency modules. The root `build.gradle` pins Gradle plugin versions and repository config. The app module declares dependencies, product flavors, signing configs, and build types. Changes to the root affect everything; changes to `app/` are scoped.

## Android Manifest Merging

Each Gradle module (your app, autolinked RN libraries, Android libraries) contributes an `AndroidManifest.xml`. **Manifest merging** combines them into the final manifest; conflicts (duplicate permissions, activities with the same name) require `tools:` attributes to resolve. The merged manifest is at `app/build/intermediates/merged_manifests/.../AndroidManifest.xml` — inspect it when permission or activity questions arise.

## Product Flavors and Build Types

**Build types** are Debug and Release (plus custom variants); **product flavors** are orthogonal axes like Free/Paid or Staging/Production. Combinations produce **variants** such as `freeDebug` or `paidRelease`. Each variant has its own source directories (`src/freeDebug/`), resources, and manifest partials.

For N build types × M flavors you get N×M variants. The default `buildTypes { debug; release }` plus flavors `{ staging; production }` gives 4. Each variant can have its own dependencies (e.g. `stagingImplementation 'some:lib:1.0'`), signing config, `resValues`, and manifest.

Variants are powerful but variant explosion (2 × 2 × 3 = 12 variants) makes CI matrices painful. Keep the matrix small — prefer build-time env flags over flavor proliferation, and remember that rarely-used variants bitrot.

## Keystores (Debug, Release, Upload)

Android apps are signed with a **keystore** containing a private key. The **debug keystore** (`~/.android/debug.keystore`) is auto-generated per-machine; debug builds from different machines have different signatures, which is why "install failed: signatures don't match" happens on team device swaps. The **release keystore** signs production builds; losing it locks you out of future updates unless you use Play App Signing.

## Play App Signing (Google-Held Key)

**Play App Signing** lets Google hold the app signing key; you upload signed AABs with an **upload key** (less catastrophic if lost — rotate via Play Console). This has been required for new apps since 2021. The upload key is your CI secret; the app signing key is Google's problem. Losing the upload key is inconvenient; losing the app signing key would have been fatal pre-Play App Signing.

## App Bundle (AAB) vs APK

Google Play requires an **AAB** (Android App Bundle) for new apps. An AAB bundles all resources, code, and configurations; Play dynamically generates per-device APKs via **bundletool**. Benefits: smaller user downloads (no unused locales, densities, ABIs). Costs: harder local testing (you need `bundletool build-apks` to simulate), and debugging requires reproducing the device-specific APK set.

## R8 / ProGuard Configuration

**R8** (which replaced ProGuard as of AGP 3.4) minifies, obfuscates, and shrinks bytecode. Minification renames identifiers to save space; this breaks reflection-based frameworks unless you add `-keep` rules. Common rules: keep JSON model classes (Gson/Moshi reflection), keep native method signatures, and keep crash reporter integrations. RN includes a default proguard-rules set; audit and extend it as you add native libs.

## Multi-APK and Split APKs

Pre-AAB, apps often shipped **multiple APKs** per ABI to minimize download size. With AAB this is automatic. For non-Play distribution channels (Huawei, Samsung, APK hosting), you may still need `splits { abi { enable true } }`. This produces per-ABI APKs; distribute the matching one to each channel.

## Native Libraries Per ABI

`libReact*.so` and autolinked native libraries are compiled **per-ABI** (arm64-v8a, armeabi-v7a, x86, x86_64). `android.defaultConfig.ndk.abiFilters` restricts which ABIs ship. A typical RN release ships arm64-v8a + armeabi-v7a for production (x86 only for emulators). With AAB, Play strips unused ABIs per device automatically.

## Version Code and Version Name Management

The **`versionCode`** is an integer that must increase on every upload; the **`versionName`** is the user-facing string ("1.2.3"). A typical CI pattern computes both from version control:

```gradle
versionCode = GIT_COMMIT_COUNT + BASE
versionName = GIT_TAG // or a package.json-driven value
```

Hardcoding the version in `build.gradle` breaks CI upload ordering — compute it at build time.

## Hermes Toggle Per Build

Setting `hermesEnabled=true` in `gradle.properties` enables **Hermes** for the app, and you can override it per-variant in `build.gradle`. Switching runtimes changes bytecode format, crash symbolication paths, and startup characteristics — profile both if uncertain. Hermes is the default for new RN apps; JSC remains supported.

## Gradle Caching and Daemon

Gradle's **build cache** (`org.gradle.caching=true`) reuses task outputs across builds; the **daemon** keeps a warm JVM between invocations (`org.gradle.daemon=true`). Together they cut incremental build times significantly. On CI, warm the cache across runs by caching `~/.gradle/caches`, `~/.gradle/wrapper`, and the project's `.gradle/` directory.

## Custom Gradle Tasks

Define **custom tasks** in `build.gradle` for project-specific automation: version bumping, generating env-specific resources, running Codegen. Register them via `tasks.register("myTask") { ... }`. Hook into the default build lifecycle with `preBuild.dependsOn("myTask")` when the output must be ready before compilation.

## Gradle Plugin Compatibility

The **Android Gradle Plugin (AGP)**, Gradle, the Kotlin plugin, and RN's own Gradle plugin share a compatibility matrix. AGP 8.x requires Gradle 8.x, and AGP features lag Gradle features by roughly 6 months. Upgrading RN often forces upgrades of all four — read the upgrade notes; don't piecemeal it.

## Minimum / Target / Compile SDK Versions

Three SDK levels govern the build: **`compileSdkVersion`** is the SDK your code compiles against, **`targetSdkVersion`** is the behavior the app promises, and **`minSdkVersion`** is the lowest supported OS. Play enforces target SDK recency — apps targeting too-old SDKs can't be uploaded after Play's yearly cutoff. Bumping `minSdkVersion` eliminates support for older devices but simplifies code (fewer API-level checks).

## Baseline Profiles for Startup

A **baseline profile** is a list of "hot" code paths compiled ahead-of-time when the app is installed, reducing JIT warm-up time on first run. Generate one via Macrobenchmark and ship it in `src/main/baseline-prof.txt`. A good baseline profile can shave 20–40% off cold-start time — measurably worth it for TTI-sensitive apps.

## See Also
- [[ios-build-system]] — iOS counterpart
- [[autolinking]] — links native deps into build
- [[fastlane]] — automates build and signing
- [[releasing]] — build feeds the release
