# E2E Testing

## What E2E Covers

**End-to-end (E2E) tests** drive a built app on a simulator, emulator, or device through realistic user flows: login, navigate, submit a form, verify the server response. They exercise JS, native, and platform integration — the whole stack — in a way unit and component tests cannot. The signal they provide is "does the critical path still work?"; the cost is slowness, flakiness, and heavy maintenance. Senior teams invest in E2E selectively, covering the top five flows thoroughly rather than every screen.

## E2E vs Integration vs Unit

The **test pyramid** holds for RN, ordered by scope and speed:

- **Unit**: an isolated function or hook; milliseconds to run.
- **Component**: a React tree without the platform; tens of milliseconds.
- **Integration**: multiple components plus real state and router; hundreds of milliseconds.
- **E2E**: the built app on a device; seconds to minutes.

You should write more unit and component tests than E2E tests, but E2E is disproportionately valuable for mobile because the integration layer is enormous and error-prone.

## Detox vs Maestro vs Appium

The main **E2E frameworks** differ in how much they know about the app:

- **Detox**: gray-box (knows app internals; synchronizes with JS and animations automatically); tight RN integration; JavaScript test code.
- **Maestro**: flow-based YAML, cloud-executable, identical syntax across platforms; simpler but less powerful.
- **Appium**: protocol-based (WebDriver), polyglot clients, broad platform support including cross-framework; black-box.

Pick Detox for JS-heavy RN projects, Maestro for fast smoke tests, and Appium when cross-framework or non-RN testing matters.

## E2E Environments (Simulators, Emulators, Real Devices)

**Simulators and emulators** are fast, reliable, and what CI uses. **Real devices** surface issues emulators miss: actual performance characteristics, hardware sensors, and network behavior. Senior teams run E2E on both — simulators for every PR, real devices nightly via a device farm. Don't skip real devices; regressions that only appear there are common.

## Test Data Management

E2E tests need accounts, data, and feature flags. The main **test-data strategies** are: seed a shared test backend before the run; use test-only endpoints that seed or reset data per-test; or maintain a pool of test accounts in various states. Per-test seeding is the most reliable but slow; a shared seed is fast but flaky when tests contaminate each other. Invest in a data-reset story before scaling E2E.

## Authentication in E2E

Real login flows are slow and fragile (captchas, 2FA). Use **test-only authentication**: a backend endpoint that issues tokens for known test users, bypassing the UI login. Gate it behind environment flags — never ship the backdoor to production. Detox can then navigate to the authenticated state in one step.

## Backend Mocking and Fixture Data

The main **backend-mocking options** are: hit a real test backend; intercept requests on the device (Charles, mitmproxy); or mock at the JS layer (MSW, msw-native). A real backend is realistic but flaky (a backend outage fails the mobile tests). Interception is powerful but adds infrastructure. JS-layer mocking is fast and reliable but tests less integration. Most teams mix: a real backend for critical paths, mocks for edge cases.

## Deep Link-Based Entry Points

Jumping directly to the screen under test via a **deep link** saves minutes per test versus navigating from the launch screen. It requires deep-link coverage across the app — a forcing function for better deep linking, which also benefits users. In Detox: `device.launchApp({ url: 'myapp://screen/123' })`.

## Navigation Flow Testing

**Navigation flow tests** verify that navigation state is correct after user actions: a tapped link leads to the right screen, the back button pops correctly, and state survives backgrounding. Cover "navigate → close → reopen → resume" lifecycles — these are where real user bugs hide. Named routes with typed params make assertions clean.

## Flake Reduction Strategies

The common E2E **flake sources** are animations that haven't completed when an assertion runs, network variability, async effects racing, and emulator resource contention. Mitigations:

- Lean on Detox's automatic sync (waits for animations and network).
- Add explicit `waitFor` on elements.
- Use deterministic test data.
- Run tests serially on a single device rather than in parallel on shared resources.
- Retry a failed test once before marking it failed.

## Retry Policies

A **retry policy** re-runs a failed test 1–2 times before reporting it. Legitimate flakes pass on retry; real bugs fail every time. Don't retry infinitely — tests that routinely need three or more retries are unreliable, and the CI signal becomes useless. Track per-test flake rate and quarantine the worst offenders.

## Video and Screenshot Capture

Detox captures **video and screenshots** on failure automatically when configured. Both are essential for debugging CI failures that don't reproduce locally. Store artifacts in CI for 7–14 days; beyond that they're usually not needed and consume disk.

## Test Parallelization

Detox supports **parallel runners** with multiple simulators or emulators. Each worker gets its own simulator, and tests within a worker run serially. The ideal worker count matches CI machine cores minus overhead. Beware of shared state (test accounts, backend data) — parallel tests corrupting each other is a common parallelism bug.

## CI Integration

A typical **CI pipeline** is: build app → boot simulator → run Detox → publish JUnit results, screenshots, and videos → teardown. macOS runners are required for iOS E2E; Linux works for Android. Cache derived data and Gradle/Pods across runs. Budget-wise, iOS E2E suites run 10–20× slower than unit tests on similar hardware.

## Execution on Device Farms

**Device farms** (Firebase Test Lab, AWS Device Farm, BrowserStack, Sauce Labs) run your binaries on pools of real devices. They are useful for broad device/OS matrix coverage — the long tail of Android fragmentation. The trade-off is that they are slower (queuing) and more expensive, so run them nightly rather than per-PR.

## Maintenance Cost Reality

Budget 20–40% of E2E test effort for **maintenance**: updating selectors after UI changes, debugging new flakiness, and aligning with backend changes. Unmaintained E2E suites rot into something worse than useless — green-always because of ignored retries, or red-always because of known failures. A small, well-maintained suite beats a large, neglected one.

## Contract Testing as a Complement

**Consumer-driven contract tests** (Pact) between the mobile app and backend catch API contract violations faster than E2E: the mobile app declares what it expects, and the backend's CI verifies it provides that. E2E still catches integration bugs that contract tests miss, so the two together are cheaper than E2E alone.

## What Not to E2E (and Why)

Edge cases, error paths, complex business logic, and fine UI details are all poor E2E candidates. E2E cost scales linearly with test count, while unit and component tests scale better. Reserve E2E for login, critical checkout or submit flows, core navigation, and any flow where bugs would cost user trust. Everything else belongs in faster tests.
