---
name: synchronizing-with-idle
description: Use this skill to choose the right idle-synchronization primitive in Compose UI tests — waitForIdle, awaitIdle, waitUntil(conditionDescription, timeoutMillis, condition), waitUntilNodeCount, waitUntilExactlyOneExists, waitUntilAtLeastOneExists, waitUntilDoesNotExist, runOnIdle, runOnUiThread, runWhenIdle, awaitAndRunWhenIdle, hasPendingWork, and IdlingResource. Explains what "idle" means (no pending recomposition or draw, every IdlingResource isIdleNow), the wall-clock vs test-clock timeout split, the Espresso/Robolectric strategy bridge, and why direct state reads from the test thread race the recomposer. If the user mentions "test is flaky", "test passes locally fails on CI", "Thread.sleep waiting for state", "Espresso IdlingResource", "ComposeTimeoutException waitUntil", "AndroidComposeUiTestTimeoutException", waitUntilExactlyOneExists, runOnIdle, runWhenIdle, or "register IdlingResource", use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - wait-for-idle
  - wait-until
  - idling-resource
  - run-on-idle
  - run-on-ui-thread
  - flaky-test
  - espresso-bridge
---

# Synchronizing With Idle — Pick the Right Wait

A Compose test that waits on the wrong primitive is the #1 source of flakiness. This skill enumerates every idle/wait API on `ComposeTestRule` and `ComposeUiTest`, locks down what "idle" actually means, and gives a decision matrix for choosing among them. Animation-specific waits live in `../testing-animations-deterministically/SKILL.md`; the underlying clock semantics live in `../controlling-the-test-clock/SKILL.md`.

## When to use this skill

- The test occasionally fails with "node not found" or "node count mismatch" but the production code is correct.
- The developer reaches for `Thread.sleep(2000)` to wait for a screen, a snackbar, a navigation transition, or an IO-backed state.
- A `ViewModel` posts state from a coroutine and the test needs to wait until the UI reflects it.
- The developer asks about `IdlingResource`, the Espresso bridge, `runOnIdle` vs `runOnUiThread`, or whether `runWhenIdle` is faster.
- The developer mentions `waitUntil`, `waitUntilExactlyOneExists`, `waitUntilNodeCount`, `AndroidComposeUiTestTimeoutException`, or `IdlingPolicies.setMasterPolicyTimeout`.
- The developer reads or mutates state from the test thread (e.g. `state.firstVisibleItemIndex`) and asks why it sometimes returns stale data.

## When NOT to use this skill

- The condition is a paused-clock animation. Use `../testing-animations-deterministically/SKILL.md` and `mainClock.advanceTimeUntil`.
- The mechanics of `MainTestClock` itself (frames, rounding, dispatchers) are unclear. Read `../controlling-the-test-clock/SKILL.md` first.
- The test is failing because of test-tag/finder issues, not async timing. Use `../../debug/printing-the-semantics-tree/SKILL.md` and `../../finders/finding-nodes-by-tag-text-content/SKILL.md`.
- Espresso interop is the actual question. Use `../../interop/testing-with-espresso-interop/SKILL.md` (sibling skill, separate scope).

## Prerequisites

- `androidx.compose.ui:ui-test` and `androidx.compose.ui:ui-test-junit4` (or `runComposeUiTest`).
- A `ComposeContestTestRule` from `createComposeRule()` (PREFERRED v2: `androidx.compose.ui.test.junit4.v2.createComposeRule`).
- For experimental `waitUntilNodeCount`/`waitUntilExactlyOneExists`/`waitUntilAtLeastOneExists`/`waitUntilDoesNotExist`: `@OptIn(ExperimentalTestApi::class)` on the test class or method.

## What "idle" means

Three conditions, all true at once (`compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/ComposeUiTest.kt:174-186`):

1. **No pending recompositions.** No snapshot apply notifications outstanding, no recomposer pending work, no `withFrameNanos` awaiters on the test frame clock.
2. **No pending draw call.** Measure and layout passes have run; on Android the draw pass has been requested by the framework (Choreographer drives draw on Android, the test clock does not — `MainTestClock.kt:36-41`).
3. **All registered `IdlingResource`s report `isIdleNow == true`.** This is how non-Compose async work (an `OkHttp` call, a `Room` query) participates in synchronization.

Compose's own work registers automatically through `ComposeIdlingResource` (`compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeIdlingResource.android.kt`), which drains the recomposer + snapshot + frame-clock awaiters in a loop **capped at 100 frames per call** (line 109). A test that needs more than 100 frames to settle is doing too much per `waitForIdle`.

## The primitive surface

Every entry below is from `ComposeUiTest.kt` / `ComposeTestRule.jvmAndAndroid.kt`.

| API | Effect | Timeout source |
|---|---|---|
| `waitForIdle()` | Blocks current thread until the three idle conditions hold. With `autoAdvance = true`, also auto-advances the clock to drain pending work. | Wall clock — `IdlingPolicies.getMasterIdlingPolicy()` |
| `awaitIdle()` | `suspend` variant of `waitForIdle`. | Same |
| `waitUntil(conditionDescription, timeoutMillis = 1_000, condition)` | Blocks until `condition()` returns `true`. Each iteration calls `mainClock.advanceTimeByFrame()` (when `autoAdvance == true`) AND `Thread.sleep(10)` wall clock (`ComposeUiTest.android.kt:899-902`). The timeout is measured against `System.nanoTime()`, so it expires on wall-clock time even though the test clock also advances. | **Wall clock** — per-call `timeoutMillis` |
| `waitUntilNodeCount(matcher, count, timeoutMillis = 1_000L)` | `@ExperimentalTestApi`. `waitUntil` over `onAllNodes(matcher).fetchSemanticsNodes(atLeastOneRootRequired = false).size == count` (`ComposeUiTest.kt:275-285`). | **Wall clock** |
| `waitUntilExactlyOneExists(matcher, timeoutMillis = 1_000L)` | `@ExperimentalTestApi`. Sugar for `waitUntilNodeCount(matcher, 1, timeoutMillis)`. | **Wall clock** |
| `waitUntilAtLeastOneExists(matcher, timeoutMillis = 1_000L)` | `@ExperimentalTestApi`. (`ComposeUiTest.kt:298-305`) | **Wall clock** |
| `waitUntilDoesNotExist(matcher, timeoutMillis = 1_000L)` | `@ExperimentalTestApi`. Sugar for `waitUntilNodeCount(matcher, 0, timeoutMillis)`. | **Wall clock** |
| `runOnIdle { … }` | `waitForIdle()` then `runOnUiThread { … }`. The default for state mutations and reads. | Wall clock (the inner `waitForIdle`) |
| `runOnUiThread { … }` | Posts a `FutureTask` via `Instrumentation.runOnMainSync` (`AndroidSynchronization.android.kt`). If already on the UI thread, runs in-place. **Does NOT wait for idle.** | None |
| `runWhenIdle { … }` | `waitForIdle()` then `runOnUiThread { … }`, but suppresses the implicit `waitForIdle` triggered by node queries inside the lambda. Faster for assert-only blocks. **MUST NOT mutate state inside.** | Wall clock |
| `awaitAndRunWhenIdle { … }` | `suspend` variant of `runWhenIdle`. | Wall clock |
| `hasPendingWork(): Boolean` | Passive snapshot: are there awaiters on the main clock, snapshot changes, or recomposer pending work? **Reliable only when `autoAdvance = false`** (`ComposeUiTest.kt:251-256`). | None |
| `IdlingResource` registration | `rule.registerIdlingResource(myResource)` / `unregisterIdlingResource`. | Wall clock |

## Timeout sources — do not mix them

| API | Source | Default | Override |
|---|---|---|---|
| `waitForIdle` / `awaitIdle` | Wall clock — Espresso `IdlingPolicies` master timeout | 26 s | `IdlingPolicies.setMasterPolicyTimeout(...)` in `@Before` |
| `waitUntil(…)` family | Wall clock | 1000 ms per call | per-call `timeoutMillis` |
| `MainTestClock.advanceTimeUntil(…)` | **Test clock** | 1000 ms per call | per-call `timeoutMillis` |
| `runComposeUiTest(testTimeout = …)` | Wall clock | `60.seconds` | `runComposeUiTest(testTimeout = 5.minutes) { … }`; throws `AndroidComposeUiTestTimeoutException` |

**Skydoves hot take #4:** `waitUntil` timeouts are wall clock; `advanceTimeUntil` is test clock. **Always** prefer `mainClock.advanceTimeUntil` when the awaited condition is observable through Compose state. Reserve `waitUntil` for conditions outside Compose's snapshot system — a `Job.isCompleted`, a counter incremented from a `LaunchedEffect`, an external service.

`RobolectricIdlingStrategy` reads the same Espresso `IdlingPolicies.getMasterIdlingPolicy()`, so a global `setMasterPolicyTimeout` lift applies to host (Robolectric) and device tests alike.

## Espresso bridge

Espresso has its own `androidx.test.espresso.IdlingResource` interface, which is **not** the same as Compose's `androidx.compose.ui.test.IdlingResource`. The bridge is `EspressoLink` (`compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/EspressoLink.android.kt`):

```text
Espresso (instrumentation tests)
        │ withStrategy { … }
        ▼
EspressoLink   ──implements──>  androidx.test.espresso.IdlingResource
        │
        │  delegates isIdleNow() to ──>  IdlingResourceRegistry
        ▼
IdlingResourceRegistry — Compose's registry
        │
        ├── ComposeIdlingResource   (recomposer + snapshot + frame-clock awaiters)
        └── any rule.registerIdlingResource(...)
```

The same registry is read by `RobolectricIdlingStrategy` for host tests. The developer registers a Compose `IdlingResource`; the framework does the bridging.

```kotlin
interface IdlingResource {
    val isIdleNow: Boolean
    fun getDiagnosticMessageIfBusy(): String? = null
}
```

(From `compose/ui/ui-test/src/jvmAndAndroidMain/kotlin/androidx/compose/ui/test/IdlingResource.kt:37-52`.) Override `getDiagnosticMessageIfBusy()` to surface a useful message when a wait times out — it is appended to the timeout exception text.

## Decision matrix

```
Need to wait for…                         | Use
------------------------------------------|----------------------------------------------
A node count to stabilize                 | waitUntilNodeCount / waitUntilExactlyOneExists
A node to appear                          | waitUntilAtLeastOneExists
A node to disappear                       | waitUntilDoesNotExist
A Compose state condition                 | mainClock.advanceTimeUntil { state }
A non-Compose condition (Job, counter)    | waitUntil(conditionDescription) { condition }
"Just settle the UI"                      | waitForIdle()  (or runOnIdle for read+act)
External async work (HTTP, DB)            | rule.registerIdlingResource(MyIdlingResource)
A paused-clock animation frame            | mainClock.advanceTimeBy(...)  -- different skill
A particular state.value to read safely   | rule.runOnIdle { state.value }
```

## Workflow

### 1. Default to `runOnIdle` for state interaction

Reading or mutating Compose state from the test thread races the recomposer. The recomposer applies snapshot writes on the main thread; the test thread can observe a half-applied state. `runOnIdle` solves this by waiting for idle then dispatching to the UI thread:

```kotlin
val firstVisibleIndex = rule.runOnIdle { state.firstVisibleItemIndex }
```

(skydoves hot take #5 — funnel state mutations through `runOnIdle` or `runOnUiThread`.)

### 2. Reach for `runWhenIdle` for read-only assertion blocks

```kotlin
rule.runWhenIdle {
    val node = rule.onNodeWithTag("counter").fetchSemanticsNode()
    assertEquals("3", node.config[SemanticsProperties.Text].first().text)
    val rect = node.boundsInRoot
    // ... more reads ...
}
```

Each node query inside a normal `runOnIdle` re-runs `waitForIdle()`. Inside `runWhenIdle`, those implicit waits are suppressed because the block already entered with idle witnessed. This matters for tests that step frames manually — node queries inside `runOnIdle` would call `waitForIdle()` repeatedly, which (under `autoAdvance = true`) auto-advances the clock and undoes the test's manual control. **MUST NOT** mutate state inside `runWhenIdle`.

### 3. Reach for the experimental `waitUntilExactlyOneExists` / friends for "wait until N nodes match"

```kotlin
@OptIn(ExperimentalTestApi::class)
@Test fun snackbarAppears() {
    rule.setContent { /* … triggers a snackbar after 200 ms … */ }
    rule.onNodeWithTag("trigger").performClick()
    rule.waitUntilExactlyOneExists(hasTestTag("snackbar"), timeoutMillis = 2_000)
    rule.onNodeWithTag("snackbar").assertTextEquals("Saved")
}
```

The default 1000 ms timeout (`ComposeUiTest.kt:278`, `ComposeUiTest.kt:301`, `ComposeUiTest.kt:321`, `ComposeUiTest.kt:334`) is often too tight for snackbars and bottom-sheet animations. Lift it explicitly per call.

### 4. Reach for `waitUntil { … }` only for non-Compose conditions

```kotlin
// SnackbarHostTest.kt:77-88
val job = scope.launch {
    hostState.showSnackbar("1")
    Truth.assertThat(resultedInvocation).isEqualTo("1")
    hostState.showSnackbar("2")
    Truth.assertThat(resultedInvocation).isEqualTo("12")
    hostState.showSnackbar("3")
    Truth.assertThat(resultedInvocation).isEqualTo("123")
}
rule.waitUntil { job.isCompleted }
```

`Job.isCompleted` is not Compose state; `mainClock.advanceTimeUntil { job.isCompleted }` would loop and time out because no amount of clock advancement makes the `Job` complete on its own.

### 5. Register an `IdlingResource` for genuine external async work

```kotlin
class HttpIdlingResource(private val client: OkHttpClient) : IdlingResource {
    override val isIdleNow: Boolean
        get() = client.dispatcher.runningCallsCount() == 0
    override fun getDiagnosticMessageIfBusy(): String? =
        "${client.dispatcher.runningCallsCount()} HTTP calls still in flight"
}

@Before fun setUp() { rule.registerIdlingResource(HttpIdlingResource(client)) }
@After  fun tearDown() { rule.unregisterIdlingResource(HttpIdlingResource(client)) }
```

After registration, every `waitForIdle` (and the implicit `waitForIdle` inside every node query) will block until `isIdleNow == true`. **MUST** keep `isIdleNow` lightweight — it is called from the main thread (`IdlingResource.kt:38-46`).

## Patterns

### Pattern: WRONG — `Thread.sleep` to wait for a node

```kotlin
// WRONG
@Test fun successScreen() {
    rule.setContent { App() }
    rule.onNodeWithTag("login").performClick()
    Thread.sleep(2_000)
    rule.onNodeWithTag("dashboard").assertExists()
}
// WRONG because: Thread.sleep is unrelated to Compose's synchronization. It either over-
// waits (slowing the test) or under-waits (flake when CI is slow). Compose's own
// IdlingResource is already wired; just wait on the node directly.
```

```kotlin
// RIGHT
@OptIn(ExperimentalTestApi::class)
@Test fun successScreen() {
    rule.setContent { App() }
    rule.onNodeWithTag("login").performClick()
    rule.waitUntilExactlyOneExists(hasTestTag("dashboard"), timeoutMillis = 2_000)
    rule.onNodeWithTag("dashboard").assertExists()
}
```

### Pattern: WRONG — `waitUntil` for a Compose-state-observable condition

```kotlin
// WRONG
rule.waitUntil(timeoutMillis = 5_000) { state.value == Phase.Done }
// WRONG because: timeout is measured in wall clock and each iteration burns ~10 ms
// of real time on Thread.sleep(10) (ComposeUiTest.android.kt:899-902). For a Compose
// state condition, advanceTimeUntil drives the test clock and is deterministic.
```

```kotlin
// RIGHT
rule.mainClock.advanceTimeUntil(timeoutMillis = 5_000) { state.value == Phase.Done }
```

(skydoves hot take #4.)

### Pattern: WRONG — reading state from the test thread

```kotlin
// WRONG
val i = state.firstVisibleItemIndex
assertEquals(2, i)
// WRONG because: the test thread races the recomposer. The read may observe a half-applied
// snapshot, especially right after a click or scroll. Compose's docs are explicit: state
// reads from outside the UI thread are not synchronized.
```

```kotlin
// RIGHT
val i = rule.runOnIdle { state.firstVisibleItemIndex }
assertEquals(2, i)
```

(skydoves hot take #5.)

### Pattern: `hasPendingWork()` — passive check, paused-clock only

```kotlin
rule.mainClock.autoAdvance = false
rule.setContent { /* … */ }
rule.mainClock.advanceTimeByFrame()
rule.runOnUiThread { trigger = true }
assertTrue(rule.hasPendingWork())              // recomposition queued, no frame yet
rule.mainClock.advanceTimeByFrame()
assertFalse(rule.hasPendingWork())             // frame applied
```

From `ComposeUiTest.kt:251-256`:

> If `autoAdvance` is `true`, the testing framework continuously processes pending work. In that scenario, calling this method acts as a momentary snapshot and will generally return `false`. It may briefly return `true` if work is queued but the framework hasn't auto-advanced yet, making the result fleeting and unreliable for driving test logic.

**MUST NOT** drive test logic off `hasPendingWork()` while `autoAdvance == true`.

### Pattern: lift the master idling timeout for slow CI

```kotlin
@Before fun setUp() {
    IdlingPolicies.setMasterPolicyTimeout(60, TimeUnit.SECONDS)
}
```

Apply when `waitForIdle` (not `waitUntil`) times out on CI but passes locally. Both `EspressoLink` and `RobolectricIdlingStrategy` honor this policy.

## Mandatory rules

- **MUST** prefer `mainClock.advanceTimeUntil { state }` over `rule.waitUntil { state }` whenever the awaited condition is a Compose-state read (skydoves hot take #4).
- **MUST** wrap state reads from the test thread in `runOnIdle { … }` or `runOnUiThread { … }`. Direct `state.value` reads from the test thread race the recomposer (skydoves hot take #5).
- **MUST** call `runOnUiThread` (not `runOnIdle`) for state mutations under a paused clock (`autoAdvance = false`); `runOnIdle`'s implicit `waitForIdle` is wrong for that mode. See `../testing-animations-deterministically/SKILL.md`.
- **MUST** use `runWhenIdle { … }` for assert-only blocks that do many node reads — it suppresses the redundant per-query `waitForIdle`. **MUST NOT** mutate state inside.
- **MUST** prefer `waitUntilExactlyOneExists` / `waitUntilAtLeastOneExists` / `waitUntilDoesNotExist` over hand-rolled `waitUntil { onAllNodes(...).fetchSemanticsNodes().isNotEmpty() }`. They are clearer and cite a `conditionDescription` in the timeout message.
- **MUST NOT** treat `hasPendingWork()` as actionable while `autoAdvance == true`. It is a passive snapshot; reliable only with the clock paused (`ComposeUiTest.kt:251-256`).
- **MUST NOT** use `Thread.sleep` to wait for state to settle. Replace with `waitUntil*`, `mainClock.advanceTimeUntil`, or an `IdlingResource` (skydoves hot take #7). The one legitimate `Thread.sleep` is on the RenderThread for ripple/screenshot tests.
- **PREFERRED:** an `IdlingResource` per genuine external async source (HTTP, DB, an external counter), registered in `@Before` and unregistered in `@After`. Override `getDiagnosticMessageIfBusy()` so timeout messages are useful.
- **PREFERRED:** `IdlingPolicies.setMasterPolicyTimeout(...)` in `@Before` for CI flakes due to slow agents — uniformly lifts both Espresso and Robolectric.

## Verification

- [ ] No `Thread.sleep` exists in any test method except the documented RenderThread/screenshot exceptions.
- [ ] Every state read from the test thread is wrapped in `runOnIdle { … }` (search: `grep -nP 'state\.\w+' src/androidTest`).
- [ ] Every `waitUntil { state.value … }` for a Compose-state condition has been migrated to `mainClock.advanceTimeUntil { state.value … }`.
- [ ] Every "wait for node" path uses `waitUntilExactlyOneExists` / `waitUntilAtLeastOneExists` / `waitUntilDoesNotExist` rather than a hand-rolled `waitUntil` over `fetchSemanticsNodes()`.
- [ ] Every external async source has a registered `IdlingResource`, registered in `@Before` and unregistered in `@After`.
- [ ] No call site relies on `hasPendingWork()` while `autoAdvance == true`.
- [ ] CI run completes 50 iterations without an `AndroidComposeUiTestTimeoutException` or `ComposeTimeoutException` flake.

## References

- Android Developers — Compose testing: https://developer.android.com/develop/ui/compose/testing
- Android Developers — Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- Espresso `IdlingPolicies` reference: https://developer.android.com/reference/androidx/test/espresso/IdlingPolicies
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/ComposeUiTest.kt:174-260` — `waitForIdle`, `awaitIdle`, `waitUntil`, `runOnIdle`, `runOnUiThread`, `runWhenIdle`, `awaitAndRunWhenIdle`, `hasPendingWork`.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/ComposeUiTest.kt:275-335` — the experimental `waitUntilNodeCount` / `waitUntilExactlyOneExists` / `waitUntilAtLeastOneExists` / `waitUntilDoesNotExist` extensions.
- `compose/ui/ui-test/src/jvmAndAndroidMain/kotlin/androidx/compose/ui/test/IdlingResource.kt` — the public `IdlingResource` interface.
- `compose/ui/ui-test/src/jvmAndAndroidMain/kotlin/androidx/compose/ui/test/IdlingResourceRegistry.jvmAndAndroid.kt` — the registry that aggregates user-registered resources + Compose's own.
- `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeIdlingResource.android.kt` — Compose's automatically-registered resource; 100-frame cap at line 109.
- `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/EspressoLink.android.kt` — the `androidx.test.espresso.IdlingResource` bridge.
- `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeUiTest.android.kt:885-908` — `waitUntil`'s 10 ms wall-clock poll loop.
- `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/AndroidSynchronization.android.kt` — `runOnUiThread` posts a `FutureTask` via `Instrumentation.runOnMainSync`.
- `compose/material3/material3/src/androidDeviceTest/kotlin/androidx/compose/material3/SnackbarHostTest.kt:77-88` — canonical `rule.waitUntil { job.isCompleted }`.
- Sibling skill: `../controlling-the-test-clock/SKILL.md` — `MainTestClock`, frame model, `advanceTimeUntil`.
- Sibling skill: `../testing-animations-deterministically/SKILL.md` — paused-clock animation tests.
- Sibling skill: `../../patterns/structuring-a-compose-test/SKILL.md` — JUnit lifecycle around idle waits.
- Sibling skill: `../../debug/printing-the-semantics-tree/SKILL.md` — when "node not found" turns out to be a finder issue, not an async issue.
