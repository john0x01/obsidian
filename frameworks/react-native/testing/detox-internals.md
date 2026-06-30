# Detox Internals

## Gray-Box vs Black-Box Testing

Black-box testing (Appium, Maestro) drives the app through public accessibility APIs without app-internal knowledge. Gray-box (Detox) injects a helper into the app build that observes JS execution, animation state, and network activity. Gray-box enables *automatic synchronization* — Detox waits for the app to settle before advancing — dramatically reducing the flakiness that plagues black-box E2E. The cost is platform specificity: Detox needs RN-aware integration.

## Automatic Synchronization (Network, Animations, Queues)

Detox tracks active "idle resources": pending fetch requests, running animations, unprocessed JS queue items, active timers. `await expect(element).toBeVisible()` internally waits until every idle resource settles before running the assertion. The JS thread being busy is enough to pause Detox — no manual `sleep(1000)` required. This is Detox's killer feature.

## Idle Resource Tracking

Under the hood, Detox hooks into RN's bridge (old arch) or JSI (new arch) to observe JS thread idleness. It also patches `fetch`/XHR to track pending requests and integrates with React Navigation and Reanimated to track transitions. If a test hangs ("idle for X seconds, resource still busy"), a resource isn't reporting completion — usually an in-flight request, a polling interval, or a runaway animation.

## Test Runner Integration (Jest)

Detox runs under Jest by default (it used to support Mocha; Jest is the modern path). Each test file spawns a fresh app install or a clean app state; tests within a file share state unless reset. `jest.config.js` with a Detox preset handles the lifecycle. Jest's watch mode doesn't work well with Detox — treat E2E as a single-shot runner.

## Matchers (by.id, by.text, by.type)

`by.id('submit-button')` matches `testID` props; the most reliable because IDs are stable. `by.text('Submit')` matches visible text; brittle to copy changes. `by.type('RCTText')` matches native types; useful for library-rendered components without accessible identifiers. Combine matchers for specificity: `element(by.id('row').withAncestor(by.id('list')))`.

## Actions and waitFor

`await element(...).tap()`, `.typeText('...')`, `.scrollTo('bottom')`, `.swipe('up')`. `waitFor(element).toBeVisible().withTimeout(5000)` explicitly waits; rarely needed because auto-sync handles most cases. When auto-sync can't wait for the thing you need (a network request from native code that Detox can't see), explicit `waitFor` + manual polling bridges the gap.

## Headless Mode

`detox test --headless` runs simulators/emulators without UI windows. Required for CI; faster on local dev too, but loses the ability to visually debug a running test. Run in headed mode locally when debugging, headless otherwise.

## iOS Implementation (EarlGrey-Derived)

On iOS, Detox uses a fork of Google's EarlGrey. The app build links against a Detox native helper that inspects the UI hierarchy and synchronizes. iOS simulator performance is excellent; this is where Detox shines. Real-device iOS is supported but slower.

## Android Implementation (UIAutomator + Custom Sync)

On Android, Detox uses UIAutomator for UI interactions plus a custom idle-resource tracker in the app's managed runtime. Android emulator perf is historically worse than iOS — x86 emulators on Apple Silicon improved this dramatically. Real-device Android works well.

## Common Flakiness Sources

Animations missed by the tracker (worklets that don't register as idle resources), in-flight fetches that the fetch-patch doesn't see (native-level HTTP), timers set with `setInterval` that never terminate. When a test is flaky, the first question is: "what's busy that Detox doesn't know about?" — then either make it report idleness or wait explicitly.

## Debugging Synchronization Hangs

`--loglevel trace` dumps the idle-resource state every poll. Find the resource that never reports idle — it's almost always a known pattern: unhandled infinite animation, leaked timer, polling request. Fixing the app often makes the test reliable; adding `detox.device.disableSynchronization()` is a last resort because it removes the safety net.

## Custom Handlers for Long-Running Operations

For legitimate long operations (file upload, video processing), `device.disableSynchronization()` before, `device.enableSynchronization()` after. Inside the disabled window, use explicit `waitFor` with generous timeouts. Document these blocks with comments — they're trap zones for future maintenance.

## CI Integration

GitHub Actions / CircleCI / Bitrise run Detox on macOS runners (iOS) or Linux (Android with KVM). Typical job: install deps → build app → boot simulator → run detox → upload artifacts (JUnit, screenshots, videos). Expect 15-45 min for a modest E2E suite; budget accordingly. Cache `~/.gradle/caches`, `ios/Pods`, and built simulator app.

## Parallel Execution and Sharding

`-w N` runs N workers concurrently, each on its own simulator. On CI, `--shard 1/3` through `--shard 3/3` across three separate jobs sharards a large suite. Parallel + sharded E2E cuts wall time significantly; limit is usually CI machine count and simulator boot overhead.

## Artifacts (Screenshots, Videos, Logs)

Configure in `detox.config.js`: `artifacts.types.screenshot = 'failing'` (captures on fail), `video = 'failing'`, `log = 'all'`. Artifacts land in `artifacts/` by default. On CI, upload as job artifacts and retain for 7-14 days. Videos of failures are usually the single most useful debugging asset.

## Limitations and When to Reach for Maestro

Detox assumes RN — if you're migrating away or have a mixed native / RN app, gray-box integration gets messy. Maestro is black-box, faster to write (YAML not JS), cross-platform identical. For smoke tests where synchronization reliability matters less, Maestro wins. For deep coverage of RN-specific logic, Detox is still the more powerful tool.
