---
name: testing-animations-deterministically
description: Use this skill to write non-flaky Compose animation tests by setting mainClock.autoAdvance = false and stepping frames by hand with advanceTimeByFrame and advanceTimeBy(durationMillis). Explains why InfiniteAnimationPolicy cancels indeterminate animations the moment the test starts unless autoAdvance is disabled, why the first frame after a state toggle does not initialize playTime, and why state mutations during the paused-clock window must go through runOnUiThread instead of runOnIdle. If the user mentions "animation never finishes", "InfiniteAnimationPolicy CancellationException", "Crossfade test flaky", "indeterminate progress indicator", "Thread.sleep(500) waiting for animation", advanceTimeByFrame, advanceTimeBy, or autoAdvance, use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - animation-testing
  - main-test-clock
  - auto-advance
  - infinite-animation-policy
  - crossfade
  - flaky-animation-test
---

# Testing Animations Deterministically — `autoAdvance = false` Or Bust

A Compose animation test that does not pause the clock is by definition flaky. With `mainClock.autoAdvance = true` (the default), the framework's `InfiniteAnimationPolicy` throws `CancellationException` as soon as an indeterminate animation starts, and finite animations finish in a single auto-advanced burst with no observable intermediate state. This skill teaches the five-line recipe that fixes that, plus the runOnUiThread-vs-runOnIdle gotcha that bites every developer who tries it for the first time.

## When to use this skill

- The test asserts an intermediate frame of an animation (mid-fade alpha, mid-scroll offset, mid-Crossfade dispose).
- The composable contains an indeterminate animation (`LinearProgressIndicator()` with no progress argument, `rememberInfiniteTransition`, looping `withFrameMillis`).
- The developer reports `CancellationException("Infinite animations are disabled on tests")` and asks how to make it stop.
- The developer reaches for `Thread.sleep(500)` to "wait for the animation".
- The developer's animation test passes locally and fails on CI, or vice versa.
- The developer mentions `autoAdvance`, `advanceTimeByFrame`, `advanceTimeBy`, `Crossfade`, "flaky animation test", or `InfiniteAnimationPolicy`.

## When NOT to use this skill

- The animation is a side effect; the test only cares about the final state. Default `autoAdvance = true` is faster — see `../synchronizing-with-idle/SKILL.md`.
- The clock semantics themselves are unclear (frame model, rounding rules, v1 vs v2 dispatcher). Read `../controlling-the-test-clock/SKILL.md` first.
- The condition under test is not Compose state (a `Job.isCompleted`, a `Mockito.verify`). Use `waitUntil` or an `IdlingResource` from `../synchronizing-with-idle/SKILL.md`.
- The test uses screenshot comparison waiting on the RenderThread for ripple/elevation pixels. That is the one legitimate `Thread.sleep` site (skydoves hot take #7).

## Prerequisites

- `androidx.compose.ui:ui-test` and `androidx.compose.ui:ui-test-junit4` (or the JUnit-less `runComposeUiTest`).
- A `ComposeContentTestRule` from `createComposeRule()` (PREFERRED v2: `androidx.compose.ui.test.junit4.v2.createComposeRule`).
- Knowledge of the frame model from `../controlling-the-test-clock/SKILL.md`. In particular: the toggle frame schedules an animation; the next frame initializes `playTime = 0`.

## Why the default fails

When the test rule constructs its environment, it installs an `InfiniteAnimationPolicy` on the recomposer. Quoted from `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeUiTest.android.kt:567-575`:

```kotlin
infiniteAnimationPolicy =
    object : InfiniteAnimationPolicy {
        override suspend fun <R> onInfiniteOperation(block: suspend () -> R): R {
            if (mainClockImpl.autoAdvance) {
                throw CancellationException("Infinite animations are disabled on tests")
            }
            return block()
        }
    }
```

Any call to `withInfiniteAnimationFrameNanos` (used by `rememberInfiniteTransition`, indeterminate progress, and a handful of looping APIs) routes through this policy. While `autoAdvance == true`, the policy throws — the framework would otherwise loop forever waiting for an animation that has no end.

The fix is to disable `autoAdvance` **before** `setContent` so the policy permits the animation, and then drive frames by hand. **Skydoves hot take #3:** animation tests require `mainClock.autoAdvance = false`. There is no other supported path.

## The recipe (five lines)

```kotlin
@Test fun anyAnimation() {
    rule.mainClock.autoAdvance = false                     // 1. pause the clock FIRST
    rule.setContent { /* the composable */ }               // 2. set content
    rule.mainClock.advanceTimeByFrame()                    // 3. kick-off frame
    rule.mainClock.advanceTimeBy(durationMillis = 300)     // 4. step the animation
    rule.onNodeWithTag("…").assertIsDisplayed()            // 5. assert
}
```

The order matters. Setting `autoAdvance = false` after `setContent` is wrong: the very first composition can already kick off an indeterminate animation, and the framework will have begun auto-advancing before the test took control.

### Why the kick-off frame

From `MainTestClock.kt:53-60`:

> Because animations receive their frame time _before_ recomposition, an animation will not get its start time in the first frame after kicking it off by toggling a state variable. … The animation gets its first frame time and initialize the play time to `t=0` [in the next frame].

In other words: the toggle frame schedules. The next frame is `playTime = 0`. Without the kick-off `advanceTimeByFrame()`, the first 16 ms of the animation duration is consumed by setup and the test reads a state behind expectation. This is the root cause of "my animation is off by one frame".

## Workflow

### 1. Pause the clock before `setContent`

```kotlin
val rule = androidx.compose.ui.test.junit4.v2.createComposeRule()

@Test fun crossfade_disposes_old_content() {
    rule.mainClock.autoAdvance = false                     // <-- before setContent
    var showFirst by mutableStateOf(true)
    var disposed = false
    rule.setContent {
        Crossfade(showFirst) {
            BasicText(if (it) "First" else "Second")
            DisposableEffect(Unit) { onDispose { disposed = true } }
        }
    }
    rule.mainClock.advanceTimeByFrame()                    // kick off
    rule.mainClock.advanceTimeBy(durationMillis = 300)
    rule.runOnUiThread { showFirst = false }
    rule.mainClock.advanceTimeUntil { disposed }           // test-clock wait
    rule.onNodeWithText("First").assertDoesNotExist()
    rule.onNodeWithText("Second").assertExists()
}
```

This is `compose/animation/animation/src/androidDeviceTest/kotlin/androidx/compose/animation/CrossfadeTest.kt:70-93`, the canonical Crossfade test.

### 2. Per-frame fraction assertions: the `onAnimationFrame` helper

For tests that need to assert at every frame of a fixed-duration animation, lift this helper from `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/LazyListItemPlacementAnimationTest.kt:1724-1738`:

```kotlin
private fun onAnimationFrame(duration: Long = Duration, onFrame: (fraction: Float) -> Unit) {
    require(duration.mod(FrameDuration) == 0L)
    rule.waitForIdle()
    rule.mainClock.advanceTimeByFrame()
    var expectedTime = rule.mainClock.currentTime
    for (i in 0..duration step FrameDuration) {
        val fraction = i / duration.toFloat()
        onFrame(fraction)
        if (i < duration) {
            rule.mainClock.advanceTimeBy(FrameDuration)
            expectedTime += FrameDuration
            assertThat(expectedTime).isEqualTo(rule.mainClock.currentTime)
        }
    }
}
```

Notes on this helper:
- `FrameDuration` is a `const val 16L` companion. Always pass durations that are multiples of 16 ms — it `require(duration.mod(FrameDuration) == 0L)` to keep the math honest.
- The leading `waitForIdle()` is allowed because `autoAdvance = false` blocks frame advancement; `waitForIdle` will wait only for measure/layout passes already triggered by `setContent`.

### 3. Indeterminate progress indicators

```kotlin
// compose/material3/material3/src/androidDeviceTest/.../ProgressIndicatorTest.kt:132-156
@Test fun indeterminateLinearProgressIndicator_Progress() {
    val tag = "linear"
    rule.mainClock.autoAdvance = false                     // 1
    rule.setMaterialContent(lightColorScheme()) {
        LinearProgressIndicator(modifier = Modifier.testTag(tag))
    }
    rule.mainClock.advanceTimeByFrame()                    // kick off the infinite animation
    rule.onNodeWithTag(tag).assertRangeInfoEquals(ProgressBarRangeInfo.Indeterminate)
}
```

Without `autoAdvance = false` this test fails immediately with `CancellationException("Infinite animations are disabled on tests")` — the indicator's looping `rememberInfiniteTransition` hits the policy on the first composition.

### 4. Mutating state during the paused-clock window

`runOnIdle { … }` calls `waitForIdle()` first. With `autoAdvance = false`, `waitForIdle()` ignores frame-clock awaiters and pending recomposition (`MainTestClock.kt:107-115`) — so it does NOT deadlock on a paused animation, but it still waits for measure / layout / draw and any registered `IdlingResource`. The net effect mid-animation is a redundant idle check that adds noise: any pending `IdlingResource` (the Espresso link, a custom resource) can stall the test mid-frame. Use `runOnUiThread` for state mutations during the paused-clock window — it skips the idle check entirely:

```kotlin
// CrossfadeTest.kt:86 — the canonical pattern
rule.runOnUiThread { showFirst = false }
```

`runOnUiThread` posts the lambda to the UI thread via `Instrumentation.runOnMainSync` (`AndroidSynchronization.android.kt`), without any idle observation. This is exactly what is needed mid-animation.

## Patterns

### Pattern: WRONG — `Thread.sleep` to wait for an animation

```kotlin
// WRONG
@Test fun fade_in() {
    var visible by mutableStateOf(false)
    rule.setContent {
        val alpha by animateFloatAsState(if (visible) 1f else 0f, tween(300))
        Box(Modifier.alpha(alpha).testTag("box"))
    }
    rule.runOnUiThread { visible = true }
    Thread.sleep(500)
    rule.onNodeWithTag("box").assertIsDisplayed()
}
// WRONG because: Thread.sleep desyncs from MainTestClock. The test thread sleeps; the
// scheduler sleeps with it (autoAdvance is still true, but no one is calling waitForIdle,
// so no frames are produced). The test is racing against host-driven measure/layout, not
// against the animation. (skydoves hot take #7 — Thread.sleep is a smell.)
```

```kotlin
// RIGHT
@Test fun fade_in() {
    rule.mainClock.autoAdvance = false
    var visible by mutableStateOf(false)
    rule.setContent {
        val alpha by animateFloatAsState(if (visible) 1f else 0f, tween(300))
        Box(Modifier.alpha(alpha).testTag("box"))
    }
    rule.runOnUiThread { visible = true }
    rule.mainClock.advanceTimeByFrame()                    // kick off
    rule.mainClock.advanceTimeBy(durationMillis = 300)     // step
    rule.onNodeWithTag("box").assertIsDisplayed()
}
```

### Pattern: WRONG — leaving `autoAdvance = true` with an infinite animation

```kotlin
// WRONG
@Test fun progressIsIndeterminate() {
    rule.setContent { LinearProgressIndicator(modifier = Modifier.testTag("p")) }
    rule.onNodeWithTag("p").assertRangeInfoEquals(ProgressBarRangeInfo.Indeterminate)
}
// WRONG because: with autoAdvance = true, the InfiniteAnimationPolicy throws
// CancellationException("Infinite animations are disabled on tests") on the first frame.
// The test fails before it reaches the assertion.
```

```kotlin
// RIGHT
@Test fun progressIsIndeterminate() {
    rule.mainClock.autoAdvance = false
    rule.setContent { LinearProgressIndicator(modifier = Modifier.testTag("p")) }
    rule.mainClock.advanceTimeByFrame()                    // kick off
    rule.onNodeWithTag("p").assertRangeInfoEquals(ProgressBarRangeInfo.Indeterminate)
}
```

### Pattern: WRONG — `runOnIdle` mid-animation

```kotlin
// WRONG
rule.mainClock.autoAdvance = false
rule.setContent { /* … animation … */ }
rule.mainClock.advanceTimeByFrame()
rule.mainClock.advanceTimeBy(150)
rule.runOnIdle { showFirst = false }                       // mid-animation mutation
rule.mainClock.advanceTimeBy(150)
// WRONG because: runOnIdle calls waitForIdle() first. Under autoAdvance = false,
// waitForIdle is a no-op for clock-driven work (MainTestClock.kt:107-115) — so it
// does NOT deadlock on a paused animation, but it still waits on measure/layout/draw
// AND on every registered IdlingResource. Any pending IdlingResource (e.g. the
// Espresso link, a custom resource) can stall the test mid-frame for no benefit.
// The CrossfadeTest reference implementation uses runOnUiThread for this reason.
```

```kotlin
// RIGHT
rule.runOnUiThread { showFirst = false }                   // direct post, no idle observation
rule.mainClock.advanceTimeBy(150)
```

(skydoves hot take #5 — funnel state mutations through `runOnIdle` OR `runOnUiThread`. Mid-paused-clock, choose `runOnUiThread`.)

### Pattern: state-driven wait inside the paused-clock window

```kotlin
// PREFERRED — test-clock wait
rule.mainClock.advanceTimeUntil(timeoutMillis = 1_000) { disposed }
```

```kotlin
// LESS PREFERRED — wall-clock; sleeps 10 ms per iteration
rule.waitUntil(timeoutMillis = 1_000) { disposed }
```

**Skydoves hot take #4:** `waitUntil` timeouts are wall clock; `advanceTimeUntil` is test clock. Mixing them up produces flaky tests. For Compose-state-observable conditions during a paused-clock animation test, `advanceTimeUntil` is always correct.

## Mandatory rules

- **MUST** set `mainClock.autoAdvance = false` **before** `setContent` for any test that asserts an intermediate animation state or contains an indeterminate animation. Otherwise `InfiniteAnimationPolicy` throws `CancellationException` on the first frame (skydoves hot take #3).
- **MUST** call `mainClock.advanceTimeByFrame()` after `setContent` (or after any state toggle that starts a new animation) before stepping by `advanceTimeBy(duration)`. The toggle frame only schedules; the next frame initializes `playTime = 0` (`MainTestClock.kt:53-60`).
- **MUST** mutate state via `runOnUiThread { … }` while the clock is paused. **MUST NOT** use `runOnIdle { … }` mid-animation (skydoves hot take #5; see `CrossfadeTest.kt:86`).
- **MUST** prefer `mainClock.advanceTimeUntil { state }` over `rule.waitUntil { state }` for Compose-state-observable conditions inside the paused-clock window (skydoves hot take #4).
- **MUST NOT** use `Thread.sleep` to wait for an animation. It desyncs from the test clock entirely (skydoves hot take #7).
- **MUST NOT** leave `autoAdvance = false` set across tests. The default is `true` for a reason — re-enable it (or rely on the per-test rule lifecycle) so other tests aren't surprised.
- **PREFERRED:** durations that are multiples of `FrameDuration` (16 ms) so the per-frame math is exact. Half-frame durations round up and surprise readers.
- **PREFERRED:** the v2 entry points so composition uses `StandardTestDispatcher`. See `../controlling-the-test-clock/SKILL.md` for the migration note.

## Verification

- [ ] Every animation test sets `mainClock.autoAdvance = false` **before** `setContent`.
- [ ] Every animation test contains exactly one `advanceTimeByFrame()` immediately after `setContent` (the kick-off) and zero or more `advanceTimeBy(durationMillis = …)` calls thereafter.
- [ ] No `Thread.sleep` exists in any animation test method (search the test file: `grep -n 'Thread\.sleep' src/androidTest/**/*.kt`).
- [ ] No `CancellationException("Infinite animations are disabled on tests")` appears in CI logs.
- [ ] Mid-animation state mutations use `runOnUiThread { … }`, not `runOnIdle { … }`.
- [ ] State-observable waits inside the paused-clock window use `mainClock.advanceTimeUntil`, not `waitUntil`.
- [ ] Test passes 50 consecutive runs locally and in CI without a flake.

## References

- Android Developers — Testing animations: https://developer.android.com/develop/ui/compose/animation/testing
- Android Developers — Compose testing: https://developer.android.com/develop/ui/compose/testing
- `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeUiTest.android.kt:567-575` — the `InfiniteAnimationPolicy` block.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/MainTestClock.kt:43-78` — the per-frame ordering and the kick-off explanation.
- `compose/animation/animation/src/androidDeviceTest/kotlin/androidx/compose/animation/CrossfadeTest.kt:70-93` — canonical `autoAdvance = false` + `runOnUiThread { state = … }` test.
- `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/LazyListItemPlacementAnimationTest.kt:1724-1738` — `onAnimationFrame` helper for per-frame fraction assertions.
- `compose/material3/material3/src/androidDeviceTest/kotlin/androidx/compose/material3/ProgressIndicatorTest.kt:132-156` — the indeterminate progress recipe.
- Sibling skill: `../controlling-the-test-clock/SKILL.md` — the frame model, rounding, and v1/v2 dispatcher difference.
- Sibling skill: `../synchronizing-with-idle/SKILL.md` — `waitUntil` vs `advanceTimeUntil`, `IdlingResource` for non-Compose async work.
- Sibling skill: `../../patterns/structuring-a-compose-test/SKILL.md` — JUnit rule wiring around an animation test.
