---
name: injecting-touch-gestures
description: Use this skill to drive Jetpack Compose UI with synthetic touch events through performTouchInput and the TouchInjectionScope DSL — click, longClick, doubleClick, swipe, swipeUp / swipeDown / swipeLeft / swipeRight, swipeWithVelocity, pinch, multiTouchSwipe, plus the low-level down / moveTo / moveBy / move / up / cancel primitives. Covers node-local coordinates, the geometry helpers (center, topLeft, bottomRight, percentOffset), event batching, splitting a gesture across multiple performTouchInput blocks, and why performGesture must not be used. Use when the developer asks "how do I simulate a swipe", "drag a node by 100 pixels", "long-press in a Compose test", "pinch to zoom", "test a fling with velocity", or reports "performGesture is deprecated", a Thread.sleep mid-gesture, or a flaky drag test using hardcoded screen coordinates.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - performTouchInput
  - touch-injection-scope
  - swipe
  - pinch
  - longClick
  - swipe-with-velocity
  - performGesture-deprecated
  - gesture-dsl
---

# Injecting Touch Gestures — performTouchInput, Not performGesture

`performTouchInput { … }` is the single entry point for synthetic touch events in Compose tests. It runs a block in a `TouchInjectionScope` whose coordinate system is node-local (origin at the node's top-left), batches every event the block enqueues, and flushes them as a single non-recomposing burst when the block returns. The legacy `performGesture { … }` is `@Deprecated("Replaced by performTouchInput")` (Actions.kt:337-355) — **MUST NOT** appear in any new code.

## When to use this skill

- The test must simulate a partial gesture, multi-touch, fling with controlled velocity, or pinch-to-zoom — anything richer than a single tap.
- The developer needs precise coordinates relative to a node (e.g. drag by exactly 100 px from `center`).
- A gesture must be split across two `performTouchInput` blocks while a finger is still "down".
- A test currently uses `performGesture { }` and must be migrated.
- A test simulates touch with hardcoded screen coordinates and is flaky on different display sizes.

## When NOT to use this skill

- A simple tap on a node — use `performClick()` from `../clicking-and-scrolling/SKILL.md`. It is platform-appropriate and shorter.
- The gesture is a hover, right-click, or scroll wheel — use `../injecting-mouse-and-keyboard/SKILL.md`.
- The flow is "tap a button, then assert a Snackbar appears" — assert via `../../assertions/asserting-node-state-and-text/SKILL.md`, not by inspecting touch state.
- The test is fighting a fling animation — pause the clock first per `../../synchronization/testing-animations-deterministically/SKILL.md`.

## Prerequisites

- `androidx.compose.ui:ui-test-junit4` (or `ui-test` for `runComposeUiTest`) configured per `../../setup/configuring-test-dependencies/SKILL.md`.
- The receiver node carries `Modifier.testTag("…")` from production code so the test selects it deterministically.
- For multi-touch gestures (`pinch`, `multiTouchSwipe`), the target composable accepts pointer input via `Modifier.pointerInput { … }` — typically `Modifier.transformable`, `Modifier.draggable2D`, or a custom `detectTransformGestures`.

## Workflow

1. **Acquire the receiver.** `performTouchInput` is an extension on `SemanticsNodeInteraction`. Find the node first.

```kotlin
rule.onNodeWithTag(DragSurfaceTag).performTouchInput { /* DSL */ }
```

2. **Use node-local coordinates.** Inside the block, `(0, 0)` is the node's top-left. Reach for the geometry helpers from `InjectionScope` (InjectionScope.kt:34-206): `center`, `topLeft`, `topCenter`, `topRight`, `centerLeft`, `centerRight`, `bottomLeft`, `bottomCenter`, `bottomRight`, `width`, `height`, plus `right == width - 1f` and `bottom == height - 1f` (pixels are 0-based — InjectionScope.kt:96-112). Use `percentOffset(.5f, .5f)` for relative positioning. **MUST NOT** hardcode screen coordinates.

3. **Pick the right level of abstraction.** High-level helpers cover most cases:

| Helper | Default | What it does |
|---|---|---|
| `click(position = center)` | tap at center | `down(); move(); up()` (TouchInjectionScope.kt:372) |
| `longClick(position = center, durationMillis = vc.longPressTimeoutMillis + 100)` | 100 ms past the long-press timeout | press, hold, release (TouchInjectionScope.kt:390) |
| `doubleClick(position = center, delayMillis = …)` | midway between min/max double-tap window | `click(); advanceEventTime(delay); click()` (TouchInjectionScope.kt:417) |
| `swipe(start, end, durationMillis = 200)` | 200 ms linear | linear interpolation between two points (TouchInjectionScope.kt:445) |
| `swipeUp / swipeDown / swipeLeft / swipeRight` | along axis through center | edge-to-edge sweeps (TouchInjectionScope.kt:649+) |
| `swipeWithVelocity(start, end, endVelocity, durationMillis)` | computed for feasibility | shapes the path so the final velocity matches (TouchInjectionScope.kt:617) |
| `pinch(start0, end0, start1, end1, durationMillis = 400)` | 400 ms two-pointer | two simultaneous linear paths (TouchInjectionScope.kt:574) |
| `multiTouchSwipe(curves, durationMillis, keyTimes)` | — | one curve per pointer id (TouchInjectionScope.kt:491) |

4. **Drop to low-level events for partial gestures.** All in `TouchInjectionScope` (TouchInjectionScope.kt):
   - `down(pointerId = 0, position)` — start a pointer at `position`.
   - `moveTo(pointerId = 0, position, delayMillis = eventPeriodMillis)` — enqueue a move event.
   - `moveBy(pointerId = 0, delta, delayMillis = eventPeriodMillis)` — relative form.
   - `updatePointerTo(pointerId, position)` / `updatePointerBy(pointerId, delta)` — adjust the pointer position **without** enqueueing a move; useful when batching multiple pointer updates into a single `move(delayMillis)` call.
   - `move(delayMillis = eventPeriodMillis)` — flush a move event combining every pending pointer update.
   - `up(pointerId = 0)` — release.
   - `cancel(delayMillis = eventPeriodMillis)` — emit a `MotionEvent.ACTION_CANCEL`-equivalent.
   - `currentPosition(pointerId = 0): Offset?` — query the last-known position of a pointer.

5. **Split gestures across blocks when recomposition matters mid-gesture.** Pointer state is shared across `performTouchInput` invocations on the same node — a pointer left "down" stays down. Use this to assert state at a frame the gesture passes through.

```kotlin
rule.onNodeWithTag(Tag).performTouchInput { down(center) }
// recomposition + assertions happen here
rule.onNodeWithTag(Tag).performTouchInput { moveBy(Offset(0f, 100f)); up() }
```

Subsequent invocations on a **different** node remap pointer positions into that node's local space (TouchInjectionScope.kt:50-55), so positions are still local, not screen-global.

6. **Trust event batching.** Every event enqueued inside one `performTouchInput` block is sent as one batch when the block returns; recomposition cannot interleave inside a block (Actions.kt:362-377). That is why all coordinates are resolved up front — and why a moving target does not invalidate the gesture inside one block.

## Patterns

### Pattern: Replace performGesture with performTouchInput

```kotlin
// WRONG
@Suppress("DEPRECATION")
rule.onNodeWithTag(Tag).performGesture { swipeUp() }
// WRONG because: performGesture is @Deprecated("Replaced by performTouchInput") (Actions.kt:337).
// The replaceWith metadata in the @Deprecated annotation explicitly suggests performTouchInput.
```

```kotlin
// RIGHT
rule.onNodeWithTag(Tag).performTouchInput { swipeUp() }
```

This is a non-negotiable migration. `performGesture` exists only for binary compatibility.

### Pattern: Hardcoded screen coordinates vs node-local

```kotlin
// WRONG
rule.onNodeWithTag(Tag).performTouchInput {
    down(Offset(500f, 800f))
    moveTo(Offset(500f, 700f))
    up()
}
// WRONG because: (500, 800) only happens to land on the node on certain device profiles.
// On a tablet or in landscape the node sits elsewhere, and on a small phone it may not even
// be inside the visible bounds — the test becomes device-shaped.
```

```kotlin
// RIGHT
rule.onNodeWithTag(Tag).performTouchInput {
    down(center)
    moveBy(Offset(0f, -100f))
    up()
}
```

`center`, `bottomCenter`, `topRight`, etc. are derived from `visibleSize` of the node (InjectionScope.kt:49-50, :118-185). The test runs unchanged across screen sizes.

### Pattern: Real swipe from DraggableTest

The canonical horizontal-drag pattern from `DraggableTest.kt:114-146`:

```kotlin
@Test
fun draggable_horizontalDrag() {
    var total = 0f
    setDraggableContent { Modifier.draggable(Orientation.Horizontal) { total += it } }
    rule.onNodeWithTag(draggableBoxTag).performTouchInput {
        this.swipe(
            start = this.center,
            end = Offset(this.center.x + 100f, this.center.y),
            durationMillis = 100,
        )
    }
    val lastTotal = rule.runOnIdle {
        assertThat(total).isGreaterThan(0)
        total
    }
}
```

Notes worth copying: `this.center` makes the receiver explicit (handy when nesting blocks); `durationMillis = 100` keeps the test fast; the assertion runs inside `runOnIdle` (skydoves hot take #5).

### Pattern: Pinch-to-zoom

From `TransformableTest.kt:128-141`:

```kotlin
rule.onNodeWithTag(TEST_TAG).performTouchInput {
    val leftStartX = center.x - 10
    val leftEndX = visibleSize.toSize().width * EDGE_FUZZ_FACTOR
    val rightStartX = center.x + 10
    val rightEndX = visibleSize.toSize().width * (1 - EDGE_FUZZ_FACTOR)

    pinch(
        Offset(leftStartX, center.y),
        Offset(leftEndX, center.y),
        Offset(rightStartX, center.y),
        Offset(rightEndX, center.y),
    )
}
```

`pinch(start0, end0, start1, end1)` interpolates two pointers in parallel for `durationMillis = 400` (TouchInjectionScope.kt:574-589). Use `EDGE_FUZZ_FACTOR` (e.g. `0.05f`) instead of `0f` to avoid landing exactly on the right/bottom edge — `right == width - 1f`, see InjectionScope.kt:96-102.

### Pattern: Split gesture for mid-gesture assertion

```kotlin
@Test fun longPress_thenAssert_thenRelease() {
    rule.setContent { /* … */ }

    // Hold finger down at center, do not release.
    rule.onNodeWithTag(Tag).performTouchInput { down(center) }

    // Wait for the long-press timeout to elapse via the test clock.
    rule.mainClock.advanceTimeBy(viewConfiguration.longPressTimeoutMillis + 100)
    rule.onNodeWithTag(Tag).assertHasIndicationOfLongPress()  // your assertion

    // Release in a second block; the pointer is still "down" between blocks.
    rule.onNodeWithTag(Tag).performTouchInput { up() }
}
```

Pointer state survives across blocks; the recomposer runs between them. This is how `CombinedClickableTest` exercises haptic-feedback-on-long-press paths (e.g. `CombinedClickableTest.kt:446-471`).

### Pattern: Velocity-controlled fling

```kotlin
rule.onNodeWithTag(Tag).performTouchInput {
    swipeWithVelocity(
        start = bottomCenter,
        end = topCenter,
        endVelocity = 4_000f,         // px/second
        durationMillis = 200,
    )
}
```

`swipeWithVelocity` shapes the path so the final velocity is within ~0.1 of the target (TouchInjectionScope.kt:617-637). The duration must be long enough for at least 3 input events (~40 ms minimum); the API throws `IllegalArgumentException` with a fix suggestion if the input is infeasible.

## Mandatory rules

- **MUST** use `performTouchInput { … }`. **MUST NOT** use `performGesture { … }` — it is `@Deprecated` (Actions.kt:337-355). This is non-negotiable.
- **MUST** express coordinates in the node-local system using `center`, `topLeft`, `bottomRight`, `percentOffset(...)`, etc. **MUST NOT** hardcode screen pixel coordinates — the test will be device-shaped.
- **MUST** route post-gesture state assertions through `rule.runOnIdle { … }` (skydoves hot take #5). Reading `state` on the test thread races with the recomposer that processed the gesture.
- **MUST** advance the test clock with `rule.mainClock.advanceTimeBy(...)` between split `performTouchInput` blocks when the test depends on a duration (long-press, double-tap window). **MUST NOT** use `Thread.sleep` — see `../../synchronization/synchronizing-with-idle/SKILL.md`.
- **PREFERRED:** start every drag from `center` and use `moveBy` with relative deltas. Absolute paths (`moveTo(topLeft)`) work but read worse and break under RTL layouts.
- **PREFERRED:** pick `swipe` for monotonic linear motion, `swipeWithVelocity` only when fling behaviour is under test, and `multiTouchSwipe` only for >2 pointers or non-linear curves.

## Verification

- [ ] No occurrence of `performGesture` in the test source set: `git grep -n 'performGesture' src/androidTest src/test` returns nothing.
- [ ] No `Thread.sleep` inside or between `performTouchInput` blocks (skydoves hot take #7).
- [ ] No hardcoded `Offset(<screen-pixel>, <screen-pixel>)` in the test — every position resolves through `center`, `top*`, `bottom*`, `centerLeft`, etc.
- [ ] Assertions following a touch gesture run inside `rule.runOnIdle { … }`.
- [ ] `./gradlew :app:connectedDebugAndroidTest` (or `:app:testDebugUnitTest`) passes for the test under change.

## References

- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Actions.kt` — `performTouchInput` (Actions.kt:399), `performGesture` deprecation (Actions.kt:337-355).
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/TouchInjectionScope.kt` — DSL surface (`down`, `moveTo`, `moveBy`, `up`, `cancel`, `click`, `longClick`, `doubleClick`, `swipe`, `swipeUp` / `Down` / `Left` / `Right`, `swipeWithVelocity`, `pinch`, `multiTouchSwipe`).
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/InjectionScope.kt` — geometry helpers (`center`, `topLeft`, `topCenter`, `bottomRight`, `width`, `height`, `right`, `bottom`, `percentOffset`, `visibleSize`, `advanceEventTime`).
- `compose/foundation/foundation/src/androidDeviceTest/.../DraggableTest.kt` — canonical `swipe(start, end, durationMillis)` (DraggableTest.kt:114-146).
- `compose/foundation/foundation/src/androidDeviceTest/.../TransformableTest.kt` — `pinch(...)` four-point gesture (TransformableTest.kt:128-141).
- `compose/foundation/foundation/src/androidDeviceTest/.../CombinedClickableTest.kt` — `performTouchInput { longClick() }`, split `down(center)` / `up()` blocks for haptic tests.
- `compose/foundation/foundation/src/androidDeviceTest/.../ScrollableAreaTest.kt` — low-level `down(); moveBy()` flow.
- skydoves — compose-performance-skills: https://github.com/skydoves/compose-performance-skills
