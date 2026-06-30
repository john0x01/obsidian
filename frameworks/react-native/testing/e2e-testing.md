# E2E Testing

## What E2E Covers

End-to-end tests drive a built app on a simulator/emulator/device through realistic user flows: login, navigate, submit a form, verify server response. They exercise JS + native + platform integration — the whole stack — in a way unit and component tests cannot. The signal they provide is "does the critical path still work?"; the cost is slowness, flakiness, and heavy maintenance. Senior teams invest in E2E selectively: cover the top 5 flows thoroughly, not every screen.

## E2E vs Integration vs Unit

Unit: isolated function or hook; milliseconds to run. Component: React tree without platform; tens of milliseconds. Integration: multiple components + real state/router; hundreds of ms. E2E: built app on a device; seconds to minutes. The test pyramid holds for RN — more unit/component tests than E2E, but E2E is disproportionately valuable for mobile because the integration layer is enormous and error-prone.

## Detox vs Maestro vs Appium

Detox: gray-box (knows app internals, synchronizes with JS and animations automatically); tight RN integration; JavaScript test code. Maestro: flow-based YAML, cloud-executable, cross-platform identical syntax, simpler but less powerful. Appium: protocol-based (WebDriver), polyglot clients, broad platform support including cross-framework; black-box. Pick Detox for JS-heavy RN projects, Maestro for fast smoke tests, Appium when cross-framework or non-RN testing matters.

## E2E Environments (Simulators, Emulators, Real Devices)

Simulators/emulators are fast, reliable, and what CI uses. Real devices surface issues emulators miss: actual performance characteristics, hardware sensors, network behavior. Senior teams run E2E on both — simulator for every PR, real devices nightly via a device farm. Don't skip real devices; regressions that only appear there are common.

## Test Data Management

E2E tests need accounts, data, feature flags. Options: seed a shared test backend before the run, use test-only endpoints that seed/reset data per-test, maintain a pool of test accounts in various states. Per-test seeding is most reliable but slow; shared seed is fast but flaky when tests contaminate each other. Invest in a data reset story before scaling E2E.

## Authentication in E2E

Real login flows are slow and fragile (captchas, 2FA). Use test-only authentication: a backend endpoint that issues tokens for known test users, bypassing the UI login. Gate behind environment flags — never ship the backdoor to production. Detox can then navigate to the authenticated state in one step.

## Backend Mocking and Fixture Data

Options: hit a real test backend, intercept requests on the device (Charles, mitmproxy), or mock at the JS layer (MSW, msw-native). Real backend is realistic but flaky (backend outages fail mobile tests). Interception is powerful but adds infrastructure. JS-layer mocking is fast and reliable but tests less integration. Most teams mix: real backend for critical paths, mocked for edge cases.

## Deep Link-Based Entry Points

Jumping directly to the screen under test (via a deep link) saves minutes per test over navigating from the launch screen. Requires deep link coverage across the app — this is a forcing function for better deep linking, which also benefits users. Detox: `device.launchApp({ url: 'myapp://screen/123' })`.

## Navigation Flow Testing

Test that navigation state is correct after user actions: tapped link leads to the right screen, back button pops correctly, state survives backgrounding. Cover "navigate → close → reopen → resume" lifecycles — these are where real user bugs hide. Named routes with typed params make assertions clean.

## Flake Reduction Strategies

E2E flake sources: animations not complete when assertion runs, network variability, async effects racing, emulator resource contention. Mitigations: Detox's automatic sync (waits for animations and network); explicit `waitFor` on elements; deterministic test data; running tests serially on a single device rather than in parallel on shared resources; retrying failed tests once before marking failed.

## Retry Policies

Retry a failed test 1-2 times before reporting. Legitimate flakes pass on retry; real bugs fail every time. Don't retry infinitely — tests that routinely need 3+ retries are unreliable and the signal from CI becomes useless. Track per-test flake rate and quarantine the worst offenders.

## Video and Screenshot Capture

Detox captures video and screenshots on failure automatically when configured. Both are essential for debugging CI failures that don't reproduce locally. Store artifacts in CI for 7-14 days; beyond that they're usually not needed and consume disk.

## Test Parallelization

Detox supports parallel runners with multiple simulators/emulators. Each worker gets its own simulator; tests within a worker run serially. Ideal worker count matches CI machine cores minus overhead. Beware of shared state (test accounts, backend data) — parallel tests corrupting each other is a common parallelism bug.

## CI Integration

Typical pipeline: build app → boot simulator → run Detox → publish JUnit results + screenshots + videos → teardown. macOS runners required for iOS E2E; Linux works for Android. Cache derived data and Gradle/Pods across runs. Budget: iOS E2E suites run 10-20× slower than unit tests on similar hardware.

## Execution on Device Farms

Firebase Test Lab, AWS Device Farm, BrowserStack, Sauce Labs run your binaries on real-device pools. Useful for broad device/OS matrix coverage (the long tail of Android fragmentation). Trade-off: slower (queuing), more expensive. Run nightly rather than per-PR.

## Maintenance Cost Reality

Budget 20-40% of E2E test effort for maintenance: updating selectors after UI changes, debugging new flakiness, aligning with backend changes. E2E suites that nobody maintains rot into worse-than-useless (green-always because of ignored retries, red-always because of known failures). A small, well-maintained suite beats a large, neglected one.

## Contract Testing as a Complement

Consumer-driven contract tests (Pact) between the mobile app and backend catch API contract violations faster than E2E. The mobile app declares what it expects; the backend's CI verifies it provides that. E2E still catches integration bugs contract tests miss; the two together are cheaper than E2E alone.

## What Not to E2E (and Why)

Edge cases, error paths, complex business logic, UI details — all poor E2E candidates. E2E cost scales linearly with test count while unit/component tests scale better. Reserve E2E for: login, critical checkout/submit flows, core navigation, any flow where bugs would cost user trust. Everything else belongs in faster tests.
