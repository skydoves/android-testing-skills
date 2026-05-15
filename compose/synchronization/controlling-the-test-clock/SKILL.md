---
name: controlling-the-test-clock
description: Use this skill to drive the Compose test clock by hand with MainTestClock — currentTime, autoAdvance, advanceTimeByFrame, advanceTimeBy(milliseconds, ignoreFrameDuration), and advanceTimeUntil. Explains the 16 ms frame delay used by TestMonotonicFrameClock, the per-frame ordering where withFrameNanos awaiters resume before recomposition, and how the recomposer and MainTestClock share one TestCoroutineScheduler. Covers v1 versus v2 entry-point dispatcher differences (UnconfinedTestDispatcher vs StandardTestDispatcher) and when an explicit runCurrent or advanceTimeBy(0) is required after migration. If the user mentions MainTestClock, mainClock.advanceTimeBy, mainClock.advanceTimeByFrame, mainClock.advanceTimeUntil, ComposeTimeoutException, frame delay, TestCoroutineScheduler, "test clock vs wall clock", or "v2 createComposeRule queues tasks", use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - main-test-clock
  - advance-time-by-frame
  - test-coroutine-scheduler
  - frame-delay
  - compose-timeout-exception
  - standard-test-dispatcher
---

# Controlling the Test Clock — Frames, Time, and the Recomposer

`MainTestClock` is the only knob that drives recomposition, animations, and `LaunchedEffect`s in a Compose test. This skill explains its surface, the 16 ms frame model behind it, and how `autoAdvance` flips the test from "framework drives the clock" to "the test drives the clock". The animation recipe and the idle/wait recipe live in sibling skills — this one establishes the mechanics they both depend on.

## When to use this skill

- The developer asks what `mainClock.advanceTimeBy(...)` actually does, why it rounds up, or why the animation seems off by one frame.
- The developer is migrating from `androidx.compose.ui.test.junit4.createComposeRule` (v1) to `androidx.compose.ui.test.junit4.v2.createComposeRule` (v2) and tests now require an explicit `mainClock.runCurrent()` or `advanceTimeBy(0)`.
- The developer mentions `MainTestClock`, `TestCoroutineScheduler`, `TestMonotonicFrameClock`, `ComposeTimeoutException`, `advanceTimeUntil`, or "test clock vs wall clock".
- The developer is writing a test that needs a particular frame count and is unsure whether `advanceTimeBy(5)` produces zero, one, or many frames.
- The developer asks why the animation's `playTime` is still `0` after one frame.

## When NOT to use this skill

- The goal is a deterministic animation test (the `autoAdvance = false` recipe). Use `../testing-animations-deterministically/SKILL.md`.
- The goal is to wait on `IdlingResource`s, async work, or "the UI is settled". Use `../synchronizing-with-idle/SKILL.md`.
- The goal is the higher-level test structure (rule wiring, `setContent`, JUnit lifecycle). Use `../../patterns/structuring-a-compose-test/SKILL.md`.
- The goal is choosing between `createComposeRule` and `runComposeUiTest`. Use `../../setup/choosing-test-rule-vs-runtest/SKILL.md`.

## Prerequisites

- `androidx.compose.ui:ui-test` and `androidx.compose.ui:ui-test-junit4` (or the JUnit-less `runComposeUiTest`) on the test source set.
- A `ComposeContentTestRule` from `createComposeRule()` or a `ComposeUiTest` receiver from `runComposeUiTest { … }`.
- **PREFERRED:** the v2 entry points (`androidx.compose.ui.test.junit4.v2.createComposeRule`, `androidx.compose.ui.test.v2.runComposeUiTest`). The v1 forms are `@Deprecated(level = WARNING)` (skydoves hot take #6).

## The mental model

`MainTestClock` is **not** a `MonotonicFrameClock` — it is an interface whose advance methods drive a `kotlinx.coroutines.test.TestCoroutineScheduler`. The recomposer reads frames from a `TestMonotonicFrameClock` whose `withFrameNanos` is implemented by `delay(frameDelayMillis)` against the **same** scheduler (`TestMonotonicFrameClock.jvmAndAndroid.kt:105-117`). Advancing the scheduler causes the `delay` to fire, which causes a frame to be produced, which runs `onPerformTraversals` and triggers measure+layout on each compose root.

```
MainTestClock.advanceTimeBy(ms)
        │
        ▼  AbstractMainTestClock.advanceScheduler
TestCoroutineScheduler.advanceTimeBy(ms) → runCurrent()
        │
        ▼  delay(frameDelayMillis) inside TestMonotonicFrameClock fires
performFrame()  ── awaiters (withFrameNanos) resume FIRST
        │       ── then onPerformTraversals → recomposition → measure/layout
        ▼
Recomposer applies snapshot writes
```

This is why `autoAdvance = false` is enough to freeze the entire UI: nothing else ticks the scheduler unless an explicit `mainClock.advance*` call is made (`MainTestClock.kt:43-48` KDoc).

## The MainTestClock surface

Every entry comes from `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/MainTestClock.kt`.

| Member | Behavior |
|---|---|
| `currentTime: Long` | Test clock time in milliseconds. **NOT** wall clock. Reads `scheduler.currentTime` (`AbstractMainTestClock.kt:35-36`). |
| `scheduler: TestCoroutineScheduler` | The scheduler the recomposer and clock share. Useful for `runCurrent()` from outside (`MainTestClock.kt:84-99`). |
| `autoAdvance: Boolean` | Default `true`. When `true`, framework auto-advances the clock during `waitForIdle`/`waitUntil`. When `false`, only explicit `advanceTime*` calls move the clock (`MainTestClock.kt:101-115`). |
| `advanceTimeByFrame()` | Advances by exactly one frame (16 ms on Android/Desktop). Implemented as `advanceScheduler(frameDelayMillis)` (`AbstractMainTestClock.kt:40-42`). |
| `advanceTimeBy(milliseconds, ignoreFrameDuration = false)` | Rounds **up** to the nearest multiple of frame duration unless `ignoreFrameDuration = true` (`AbstractMainTestClock.kt:44-52`, `MainTestClock.kt:120-145`). |
| `advanceTimeUntil(timeoutMillis = 1_000, condition)` | Advances frame-by-frame until `condition()` is true. Timeout is **test clock**. Throws `ComposeTimeoutException` (`MainTestClock.kt:147-167`, `AbstractMainTestClock.kt:54-72`). |

`runCurrent()` is exposed on `AbstractMainTestClock` (`AbstractMainTestClock.kt:96-98`) — it executes all tasks due at the current time without advancing the clock. Reach for it after toggling state in v2 if a queued task must run before the next assertion.

## The frame model

- Frame duration: `DefaultFrameDelay = 16_000_000L` ns = 16 ms (`TestMonotonicFrameClock.jvmAndAndroid.kt:33`).
- Per-frame order inside `performFrame`:
  1. All `withFrameNanos` awaiters resume with the new frame time. **Animations live here.**
  2. `onPerformTraversals` runs — composition + measure + layout.
  3. Resumptions queued by the awaiters are dispatched.

Quoted from `MainTestClock.kt:49-60`:

> If there is both a pending recomposition and an animation awaiting a frame time, ticking this clock will _first_ send the new frame time to the animation, and _then_ perform recomposition. … Because animations receive their frame time _before_ recomposition, an animation will not get its start time in the first frame after kicking it off by toggling a state variable.

That last sentence is the reason every animation test does an extra `advanceTimeByFrame()` immediately after `setContent` to "kick off" the animation: the toggle frame schedules the animation; the next frame initializes its play time to `0` (skydoves hot take #3).

## Workflow

### 1. Read `currentTime` only as a witness, never as a synchronization mechanism

```kotlin
// RIGHT — assert the clock advanced as much as the test asked
val before = rule.mainClock.currentTime
rule.mainClock.advanceTimeBy(durationMillis = 320)
assertEquals(320, rule.mainClock.currentTime - before)
```

```kotlin
// WRONG
while (rule.mainClock.currentTime < target) { /* spin */ }
// WRONG because: nothing in this loop drives the scheduler. The loop runs forever.
```

### 2. Choose the right advance method

| Goal | Call |
|---|---|
| "Run exactly one frame." | `mainClock.advanceTimeByFrame()` |
| "Run N frames." | `mainClock.advanceTimeBy(N * 16L)` |
| "Run a known animation duration." | `mainClock.advanceTimeBy(durationMillis = 300)` (rounded up to next 16 ms boundary) |
| "Step time without producing a new frame." | `mainClock.advanceTimeBy(milliseconds = 8, ignoreFrameDuration = true)` |
| "Wait until a Compose state condition becomes true." | `mainClock.advanceTimeUntil(timeoutMillis = 5_000) { state.value == … }` |

### 3. After mutating state under v2, run pending tasks before asserting

The v2 `createComposeRule` uses a `StandardTestDispatcher` for composition; tasks are queued, not executed immediately. If `autoAdvance = false`, neither `waitForIdle` nor a node query will drain those tasks. The fix is `mainClock.runCurrent()` or `mainClock.advanceTimeBy(0)`:

```kotlin
rule.mainClock.autoAdvance = false
rule.runOnUiThread { uiState = newValue }
// Under v2, the snapshot write is queued. Drain it before asserting.
rule.mainClock.advanceTimeByFrame()
rule.onNodeWithTag("status").assertTextEquals("Updated")
```

`AbstractMainTestClock.advanceTimeUntil` already calls `scheduler.runCurrent()` first when `isStandardTestDispatcherSupportEnabled` is true (`AbstractMainTestClock.kt:59-61`).

### 4. Treat `ComposeTimeoutException` as a verdict on the condition, not on the clock

`advanceTimeUntil` throws `ComposeTimeoutException("Condition still not satisfied after $timeoutMillis ms")` when the test clock advances past `timeoutMillis` without `condition()` returning `true` (`AbstractMainTestClock.kt:65-68`). The exception means the condition will never be satisfied by clock-driven work — typically a missing snapshot write, a `LaunchedEffect` that never starts, or a state read on the wrong thread.

## Patterns

### Pattern: rounding up

```kotlin
// WRONG
rule.mainClock.advanceTimeBy(milliseconds = 5)
// expectation: "1 frame ran"
// WRONG because: the framework rounds 5 ms UP to the next 16 ms multiple — i.e. 16 ms.
// One frame did run, but the developer is reading the API as if it could advance by 5 ms.
// Use ignoreFrameDuration = true if sub-frame stepping is genuinely required.
```

```kotlin
// RIGHT
rule.mainClock.advanceTimeByFrame()             // exactly 16 ms, exactly one frame
// or
rule.mainClock.advanceTimeBy(milliseconds = 5, ignoreFrameDuration = true)  // 5 ms, 0 frames
```

### Pattern: the "kick-off" frame for animations

```kotlin
// WRONG
rule.mainClock.autoAdvance = false
rule.setContent { Crossfade(showFirst) { /* … */ } }
rule.mainClock.advanceTimeBy(300)                  // expect animation finished
rule.onNodeWithText("Second").assertExists()
// WRONG because: the toggle frame only schedules the animation; the animation's playTime is 0
// going into the next frame. The first 16 ms of the 300 are eaten by setup.
```

```kotlin
// RIGHT — see CrossfadeTest.kt:70-93
rule.mainClock.autoAdvance = false
rule.setContent { Crossfade(showFirst) { /* … */ } }
rule.mainClock.advanceTimeByFrame()                // kick off
rule.mainClock.advanceTimeBy(durationMillis = 300) // step
rule.onNodeWithText("Second").assertExists()
```

### Pattern: v1 → v2 migration leaves a state read stale

```kotlin
// WRONG (v2)
val rule = androidx.compose.ui.test.junit4.v2.createComposeRule()
@Test fun foo() {
    var counter by mutableStateOf(0)
    rule.setContent { Text(counter.toString()) }
    rule.runOnUiThread { counter = 1 }
    rule.onNodeWithText("1").assertExists()
    // WRONG because: with StandardTestDispatcher, the snapshot apply is queued.
    // No frame has been produced. The composition still reads `0`.
}
```

```kotlin
// RIGHT (v2)
@Test fun foo() {
    var counter by mutableStateOf(0)
    rule.setContent { Text(counter.toString()) }
    rule.runOnUiThread { counter = 1 }
    rule.mainClock.advanceTimeByFrame()
    rule.onNodeWithText("1").assertExists()
}
```

### Pattern: `advanceTimeUntil` saves wall time

```kotlin
// LESS PREFERRED — wall clock
rule.waitUntil(timeoutMillis = 5_000) { state.value == Phase.Done }
```

```kotlin
// PREFERRED — test clock; framework iterates frame-by-frame and the timeout is in
// test clock time (no real wall-clock burn between iterations).
rule.mainClock.advanceTimeUntil(timeoutMillis = 5_000) { state.value == Phase.Done }
```

Both variants advance the test clock when `autoAdvance == true`: `waitUntil` calls `mainClock.advanceTimeByFrame()` AND `Thread.sleep(10)` per iteration (`ComposeUiTest.android.kt:899-902`); `advanceTimeUntil` only advances the test clock by frames (`AbstractMainTestClock.kt:54-72`). The difference is the **timeout source** — wall vs test — and the absence of `Thread.sleep` overhead in the test-clock variant.

## Mandatory rules

- **MUST** treat `MainTestClock.currentTime` as test clock time only. Comparing it to `System.currentTimeMillis()` is meaningless.
- **MUST** call `advanceTimeByFrame()` (not `advanceTimeBy(16)`) when the intent is "exactly one frame." It documents intent and avoids reasoning about rounding.
- **MUST** know that `advanceTimeBy(milliseconds = N)` rounds **up** to the next multiple of `frameDelayMillis` (16 ms on Android/Desktop). Pass `ignoreFrameDuration = true` to suppress rounding (`AbstractMainTestClock.kt:44-52`).
- **MUST** call `advanceTimeByFrame()` (or `advanceTimeBy(0)` / `runCurrent()`) after toggling state under v2 (`StandardTestDispatcher`) before reading the resulting UI state. v1's `UnconfinedTestDispatcher` dispatches eagerly and does not need this; v2 does (skydoves hot take #6).
- **MUST NOT** use `advanceTimeUntil` for conditions outside Compose's snapshot system (e.g. a `Job.isCompleted`, an `OkHttp` callback). Those conditions will never become true by ticking the test clock — `waitUntil` (wall clock) or an `IdlingResource` is the right tool. See `../synchronizing-with-idle/SKILL.md`.
- **MUST NOT** confuse `MainTestClock` with `MonotonicFrameClock`. They are different types. `MainTestClock` exposes the scheduler that drives the `TestMonotonicFrameClock` the recomposer uses.
- **PREFERRED:** the v2 entry points so the dispatcher matches `kotlinx.coroutines.test.runTest` semantics. The v1 `createComposeRule` / `runComposeUiTest` forms are deprecated `WARNING`.
- **PREFERRED:** `advanceTimeUntil` over `waitUntil` whenever the awaited condition is observable through Compose state. Test clock is faster and deterministic; wall clock sleeps 10 ms per iteration.

## Verification

- [ ] Every call site uses `advanceTimeByFrame()` for "one frame" and `advanceTimeBy(...)` only when the duration is meaningful.
- [ ] No `advanceTimeBy(milliseconds = X)` where `X < 16` exists without a comment explaining the intent or an `ignoreFrameDuration = true` flag.
- [ ] After v1 → v2 migration, every state mutation followed by an assertion either calls `mainClock.advanceTimeByFrame()` / `advanceTimeBy(0)` / `runCurrent()` or relies on `runOnIdle { … }` to drain queued tasks.
- [ ] `advanceTimeUntil` sites read Compose state only — no `Job.isCompleted`, no external counters.
- [ ] `ComposeTimeoutException` from a clock test is treated as a missing snapshot write or wrong-thread state read, not a "raise the timeout" event.

## References

- Android Developers — Compose testing: https://developer.android.com/develop/ui/compose/testing
- Android Developers — Testing animations: https://developer.android.com/develop/ui/compose/animation/testing
- Compose UI release notes: https://developer.android.com/jetpack/androidx/releases/compose-ui
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/MainTestClock.kt` — the public interface and `ComposeTimeoutException`.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/AbstractMainTestClock.kt` — the rounding rule, `runCurrent`, `advanceScheduler`.
- `compose/ui/ui-test/src/jvmAndAndroidMain/kotlin/androidx/compose/ui/test/TestMonotonicFrameClock.jvmAndAndroid.kt` — `DefaultFrameDelay = 16_000_000L`, the `delay`-based frame loop.
- `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/MainTestClockImpl.android.kt` — the Android actual.
- `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeUiTest.android.kt` — `waitUntil`'s 10 ms wall-clock sleep at line 885-905.
- Sibling skill: `../testing-animations-deterministically/SKILL.md` — the full `autoAdvance = false` recipe.
- Sibling skill: `../synchronizing-with-idle/SKILL.md` — choosing between `waitForIdle`, `waitUntil`, and `advanceTimeUntil`.
