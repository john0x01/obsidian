# Releasing

## Release Process Overview

A release converts a commit into artifacts in users' hands. Steps: cut a release branch → run full regression → build signed artifacts → upload to store test tracks → promote to production → monitor → respond. Senior teams treat each step as a gate with explicit criteria, not a ceremony. The worst release processes are implicit — new engineers learn by shadowing someone and by breaking prod.

## Versioning Strategy (SemVer for Mobile)

SemVer's major/minor/patch semantics map awkwardly onto mobile because users run whichever version they have — you can't "force an upgrade" like a web deploy. Most teams use marketing version (`X.Y.Z`) for user-facing release notes and a monotonic build number for store uniqueness. Breaking changes in a mobile context usually mean API changes to the server's mobile endpoints, not the mobile app itself.

## Version Codes and Build Numbers

Android `versionCode` is a 32-bit integer, must monotonically increase per upload. iOS `CFBundleVersion` is a dotted-quad string, also monotonic within a marketing version. CI strategies: use build number from CI (Jenkins/Actions run number), or compute from Git commit count (`git rev-list --count HEAD`), or query the store for the highest and increment. Out-of-order build numbers break store uploads.

## Release Branch Workflow

Cut `release/x.y.z` from `main` at code freeze. Only bug fixes merge into the release branch; features continue on `main`. When the release goes out, tag it (`v1.2.3`) and cherry-pick or merge the release branch back. For trunk-based teams, replace the branch with a release tag plus feature flags that gate unfinished work.

## Changelog Discipline

Maintain a CHANGELOG.md grouped by version, with sections (Added, Changed, Fixed, Removed). Enforce via PR template or lint — every user-visible change gets an entry. This becomes the source for TestFlight "What to Test", store release notes, and engineering history. A neglected changelog is the first sign of release-hygiene decay.

## Code Freeze and Regression Testing

A freeze blocks non-critical merges to the release branch for N days. Run the full regression suite: E2E, manual smoke test of top user flows, crash-free baseline against a canary build. Skipping freeze is tempting for small releases; it's also how regressions reach production. Non-negotiable for major releases.

## TestFlight Internal and External

Internal testers (up to 100, no review) get builds within minutes. External testers (up to 10,000, requires App Store review) take 1–24 hours for the first review, minutes afterwards. Use internal for engineers + QA, external for stakeholders and a wider beta. The external review is stricter than App Store review in some dimensions (metadata accuracy) and laxer in others (no automatic UI review).

## Google Play Internal and Closed Testing

Internal testing (up to 100 testers) is near-instant. Closed testing (public or invite-list) is longer-lived and supports percentage rollout. Open testing is a full public beta. Each track has its own app version history — a build in internal does not automatically appear in closed unless promoted. Pre-launch reports on alpha/beta tracks catch crashes before production.

## Staged Rollout Percentages

Play supports halted-percentage rollouts (1%, 5%, 20%, 50%, 100%) with per-day advancement. App Store has phased release (automatic 7-day ramp, pause/resume controls). Rollout order: 1% day 1, monitor crash-free + key business metrics for 24h, expand on green. Halt on regression — stores make halting easy but don't automate the decision.

## Release Notes for Stores

Store release notes are short (iOS: 4000 chars but only first ~300 visible; Play: 500 chars). Users don't read them but reviewers might; keep factual and non-marketing. Localize for top markets. For minor versions, "Bug fixes and performance improvements" is fine — don't contrive novelty.

## Crash-Free Rate Gates

Track crash-free session rate (Firebase Crashlytics, Sentry, Bugsnag) per release. Block rollout advance if the new version's crash-free rate is below N% or significantly worse than the previous version. Set the bar explicitly (e.g., "advance only if crash-free > 99.5% for 24h after 5% rollout"). Without a gate, staged rollouts are theater.

## Rollback and Halting Rollouts

Play: "halt rollout" button stops new installs but leaves existing installs unchanged. App Store: phased release has pause/resume. Neither "rolls back" — users on the bad version stay there. For real rollback, ship a new patch release with the fix. OTA updates (EAS/CodePush) enable real rollbacks for JS-only bugs.

## Hotfix Workflow

Cut `hotfix/x.y.z+1` from the last release tag, not from `main`. Include only the fix and its tests. Fast-track through review and store submission (use the expedited review path on App Store sparingly — Apple tracks abuse). Merge the hotfix back to `main` immediately to avoid divergence.

## OTA vs Binary Release Decision

OTA (EAS Update, CodePush) ships JS and asset changes to existing installs in minutes; binary releases require store review. Use OTA for: JS-only bug fixes, copy changes, experiment flag updates. Use binary for: native code changes, new native permissions, significant behavior changes, anything that needs store review for compliance. Mixing strategies requires per-version native compatibility matrices.

## Release Coordination Across Teams

Backend, data, marketing, CS, and legal all have release dependencies. Backend API changes must ship before the mobile version that consumes them (or be feature-flagged). Marketing campaigns tied to feature launches need aligned rollout timing. A release checklist that names a single owner per dependency is the minimum; a release manager role is worth it at scale.

## Post-Release Monitoring Window

For 24–72 hours after a release, monitor crash-free rate, key API error rates, funnel conversion, and user feedback (app store reviews, support tickets). Name the on-call for the release explicitly. Most post-release incidents happen in this window; resources should concentrate here rather than on normal on-call.

## Store Review Timelines and Planning

App Store review: usually 24h, can stretch to 48h or reject + re-submit; critical dates (WWDC, iOS launches) slow reviews. Play Console review: much faster (hours), but policy updates trigger re-review of existing listings. Plan releases around known slow windows. Avoid Friday submissions before long weekends — a rejection over the weekend is lost time.

## Phased Rollout Between iOS and Android

Rolling out iOS first lets you catch server-side issues with a subset of traffic; Android's faster review makes it the natural quick-iteration platform. Or the reverse — some teams validate on Android's 1% rollout then push iOS. The critical rule: don't ship both simultaneously without staging, because a shared bug doubles the blast radius.
