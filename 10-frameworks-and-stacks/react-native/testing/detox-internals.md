# Detox Internals

## Gray-Box vs Black-Box Testing

**Black-box testing** (Appium, Maestro) drives the app through public accessibility APIs with no knowledge of app internals. **Gray-box testing** (Detox) injects a helper into the app build that observes JS execution, animation state, and network activity. Gray-box enables *automatic synchronization* — Detox waits for the app to settle before advancing — which dramatically reduces the flakiness that plagues black-box E2E. The cost is platform specificity: Detox needs RN-aware integration.

## Automatic Synchronization (Network, Animations, Queues)

**Automatic synchronization** is Detox's killer feature. Detox tracks active *idle resources*: pending fetch requests, running animations, unprocessed JS queue items, and active timers. A call such as `await expect(element).toBeVisible()` internally waits until every idle resource settles before running the assertion. A busy JS thread is enough to pause Detox — no manual `sleep(1000)` required.

## Idle Resource Tracking

Under the hood, **idle resource tracking** hooks into RN's bridge (old architecture) or JSI (new architecture) to observe JS thread idleness. Detox also patches `fetch` and XHR to track pending requests, and integrates with React Navigation and Reanimated to track transitions. When a test hangs ("idle for X seconds, resource still busy"), some resource isn't reporting completion — usually an in-flight request, a polling interval, or a runaway animation.

## Matchers (by.id, by.text, by.type)

**Matchers** select elements to act on or assert against:

- `by.id('submit-button')` matches `testID` props; the most reliable, because IDs are stable.
- `by.text('Submit')` matches visible text; brittle to copy changes.
- `by.type('RCTText')` matches native types; useful for library-rendered components without accessible identifiers.

Combine matchers for specificity, e.g. `element(by.id('row').withAncestor(by.id('list')))`.

## Actions and waitFor

**Actions** drive the UI: `await element(...).tap()`, `.typeText('...')`, `.scrollTo('bottom')`, `.swipe('up')`. **`waitFor`** explicitly waits, e.g. `waitFor(element).toBeVisible().withTimeout(5000)`; it is rarely needed because auto-sync handles most cases. When auto-sync can't observe what you need — for example a network request issued from native code that Detox can't see — explicit `waitFor` plus manual polling bridges the gap.

## Test Runner Integration (Jest)

Detox runs under **Jest** by default (it used to support Mocha; Jest is the modern path). Each test file spawns a fresh app install or a clean app state; tests within a file share state unless it is reset. A `jest.config.js` with a Detox preset handles the lifecycle. Jest's watch mode doesn't work well with Detox, so treat E2E as a single-shot runner.

## Headless Mode

**`detox test --headless`** runs simulators and emulators without UI windows. It is required for CI and faster on local dev too, but it loses the ability to visually debug a running test. Run headed locally when debugging, headless otherwise.

## iOS Implementation (EarlGrey-Derived)

On iOS, Detox uses a fork of Google's **EarlGrey**. The app build links against a Detox native helper that inspects the UI hierarchy and synchronizes. iOS simulator performance is excellent — this is where Detox shines. Real-device iOS is supported but slower.

## Android Implementation (UIAutomator + Custom Sync)

On Android, Detox uses **UIAutomator** for UI interactions plus a custom idle-resource tracker in the app's managed runtime. Android emulator performance has historically been worse than iOS, though x86 emulators on Apple Silicon improved this dramatically. Real-device Android works well.

## Common Flakiness Sources

The recurring **flakiness sources** are resources Detox can't see as busy:

- Animations missed by the tracker (worklets that don't register as idle resources).
- In-flight fetches the fetch-patch doesn't see (native-level HTTP).
- Timers set with `setInterval` that never terminate.

When a test is flaky, the first question is "what's busy that Detox doesn't know about?" — then either make it report idleness or wait explicitly.

## Debugging Synchronization Hangs

**`--loglevel trace`** dumps the idle-resource state on every poll. Find the resource that never reports idle; it's almost always a known pattern — an unhandled infinite animation, a leaked timer, or a polling request. Fixing the app usually makes the test reliable. Adding `detox.device.disableSynchronization()` is a last resort, because it removes the safety net.

## Custom Handlers for Long-Running Operations

For legitimate long operations (file upload, video processing), call `device.disableSynchronization()` before and `device.enableSynchronization()` after. Inside the disabled window, use explicit `waitFor` with generous timeouts. Document these blocks with comments — they are trap zones for future maintenance.

## CI Integration

CI providers (GitHub Actions, CircleCI, Bitrise) run Detox on macOS runners (iOS) or Linux (Android with KVM). A typical job is: install deps → build app → boot simulator → run Detox → upload artifacts (JUnit, screenshots, videos). Expect 15–45 minutes for a modest E2E suite, and budget accordingly. Cache `~/.gradle/caches`, `ios/Pods`, and the built simulator app.

## Parallel Execution and Sharding

**`-w N`** runs N workers concurrently, each on its own simulator. On CI, **sharding** with `--shard 1/3` through `--shard 3/3` splits a large suite across three separate jobs. Parallel and sharded E2E cuts wall time significantly; the limit is usually CI machine count and simulator boot overhead.

## Artifacts (Screenshots, Videos, Logs)

Configure **artifacts** in `detox.config.js`: `artifacts.types.screenshot = 'failing'` (captures on fail), `video = 'failing'`, `log = 'all'`. Artifacts land in `artifacts/` by default. On CI, upload them as job artifacts and retain for 7–14 days. Videos of failures are usually the single most useful debugging asset.

## Limitations and When to Reach for Maestro

Detox assumes RN — if you're migrating away or have a mixed native/RN app, gray-box integration gets messy. **Maestro** is black-box, faster to write (YAML rather than JS), and identical across platforms. For smoke tests where synchronization reliability matters less, Maestro wins. For deep coverage of RN-specific logic, Detox is still the more powerful tool.
