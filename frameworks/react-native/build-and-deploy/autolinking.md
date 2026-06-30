# Autolinking

## How Autolinking Works (RN CLI)

Autolinking means native dependencies are discovered and wired into the native projects without manual `link` steps. The RN CLI runs a discovery script at `pod install` (iOS) and during Gradle sync (Android), scanning `node_modules` for packages with a `react-native` config entry or a `react-native.config.js`. Matching packages have their native sources added to the build graph automatically.

## react-native.config.js

A package's `react-native.config.js` (or the `react-native` field in `package.json`) tells the CLI where native code lives: `dependencies.<pkgName>.platforms.ios.sourceDir`, `.podspecPath`, `.project.sourceDir` for Android. App-side `react-native.config.js` can override or exclude per-package settings. Most packages rely on defaults (standard folder layout) — explicit config is mostly for non-standard layouts.

## iOS Autolinking via CocoaPods

The `@react-native-community/cli` generates a `react-native.podspec` that imports each autolinked module's Podspec. `pod install` reads this and adds the pods to `Pods.xcodeproj`. The output is deterministic given a fixed `package-lock.json` and `Podfile.lock`. Run `pod install` after any native-dep install/remove — even when tooling claims it's unnecessary.

## Android Autolinking via Gradle

A generated `settings.gradle` snippet includes `:MyModule` from each autolinked package. The app-level `build.gradle` imports their Gradle files. Gradle's `implementation project(':MyModule')` links the module's sources into the build. Sync Gradle after native-dep changes — Android Studio usually prompts, CI needs explicit `./gradlew preBuild`.

## Native Dependency Graph Resolution

Autolinking respects the package tree: dependencies-of-dependencies are linked too, but each module is linked once (by the closest definition). Deep dependency trees are a source of version conflicts — two packages depending on different versions of the same native lib (`react-native-async-storage`, `@react-native-community/netinfo`, etc.) force a resolution strategy.

## Version Conflicts in Native Dependencies

CocoaPods will fail with "Unable to satisfy the following requirements" if two pods need incompatible versions of a shared dep; resolve via `pod 'SharedLib', :modular_headers => true` or an explicit version pin in the Podfile. Gradle will silently pick the higher version by default; force via `resolutionStrategy.force`. The underlying fix is usually to align the JS-package versions.

## Overriding Autolinked Modules

Disable autolinking for a package via `dependencies.<pkgName>.platforms.ios = null` in `react-native.config.js`; then link it manually. Useful when a package's shipped podspec is broken or you need to pin to a fork. Overriding is an escape hatch — every override is a maintenance surface for upgrades.

## Expo Modules Autolinking

Expo modules use a separate autolinking system (`expo-modules-autolinking`) running alongside RN's. Both are active in bare Expo workflows; Expo-managed apps use only Expo's. The two coexist but duplicate some work — expect two discovery passes at `pod install`.

## Peer Dependency Issues

Many RN packages declare `react-native` as a peer dep; npm/yarn install may warn about version mismatches. Peer deps on `react-native-reanimated`, `react-native-gesture-handler`, `react-native-safe-area-context` are common. `yarn install --strict-peer-dependencies` fails loudly; relaxed installs silently allow mismatches and then crash at runtime.

## Manual Linking Fallback

For packages predating autolinking or those with non-standard native code (hand-crafted C++, custom Podspec dance), manual linking is: edit `Podfile` to add the pod, edit `settings.gradle` and `app/build.gradle` to add the module, ensure `MainApplication.java/kt` registers the package in `getPackages()`. Rare in modern RN, but inherited codebases may still use it.

## Troubleshooting Autolinking Failures

Start with `npx react-native config` — dumps the CLI's view of the dep graph. If a package is missing, check its `package.json` for the `react-native` field. Clean builds (`cd ios && pod deintegrate && pod install`, `cd android && ./gradlew clean`) before chasing exotic causes. Yarn/npm cache corruption is a surprisingly common root cause.

## Monorepo Autolinking (Yarn Workspaces, pnpm)

Monorepos hoist packages up the tree, breaking the CLI's assumption that `node_modules` lives next to the app. Fixes: `nohoist` patterns for RN packages, or `react-native.config.js` overrides pointing at the hoisted location. pnpm's strict mode requires extra config (`node-linker=hoisted` or explicit per-app `node_modules`). Plan the monorepo layout before the first RN native dep is added.

## Platform-Specific Exclusions

Mark a dep iOS-only or Android-only: `dependencies.<pkgName>.platforms.android = null`. Applied automatically for packages that ship only one platform. Important when a native dep's transitive reqs conflict on the other platform.

## Autolinking in New Architecture

TurboModules and Fabric components are autolinked the same way as legacy native modules, with additional Codegen steps. The Codegen output (C++ headers, Java interfaces) is regenerated during build; autolinking tells Codegen where to find spec files. For hybrid-architecture apps (some modules new, some old), both paths coexist.

## Autolinking Performance on Large Projects

Autolinking discovery scales linearly with `node_modules` size; large monorepos can spend 30+ seconds on `pod install` just scanning packages. Mitigation: restrict search roots via config, cache the discovery output across CI runs, or hand-curate the dep list for build-critical workflows.
