---
name: injecting-mouse-and-keyboard
description: Use this skill to drive Jetpack Compose UI from tests with non-touch input — performMouseInput (click, rightClick, doubleClick, tripleClick, longClick, animateMoveTo, dragAndDrop, smoothScroll, enter / exit, press / release / scroll), performKeyInput (keyDown, keyUp, isKeyDown, modifier-state vals isCtrlDown / isShiftDown / isAltDown / isMetaDown / isFnDown / isCapsLockOn / isNumLockOn / isScrollLockOn, helpers pressKey, withKeyDown, withKeysDown, withKeyToggled, withKeysToggled), performKeyPress for direct KeyEvent injection, and performMultiModalInput for hover-then-click flows. Covers MouseButton (Primary, Secondary, Tertiary), ScrollWheel (Horizontal, Vertical), and the repeat-key behaviour driven by advanceEventTime. Use when the developer asks "how do I right-click in a Compose test", "test a Ctrl+S shortcut", "send a keyboard shortcut", "simulate hover", "test a tooltip", "scroll the mouse wheel", or "drag and drop with a mouse" on desktop or Compose Multiplatform.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - performMouseInput
  - performKeyInput
  - performMultiModalInput
  - mouse-button
  - scroll-wheel
  - keyboard-shortcut
  - hover
  - compose-multiplatform
---

# Injecting Mouse and Keyboard — performMouseInput, performKeyInput, performMultiModalInput

Mouse and keyboard input are first-class modalities in Compose tests. Each has its own injection scope and entry point, plus a shared `performMultiModalInput` that combines all of them inside one batched gesture (touch, mouse, key, rotary, trackpad, indirect pointer). Coordinates remain node-local — the same rules as touch — and the entire injection state survives across `perform.*Input` blocks on the same node.

## When to use this skill

- The test exercises a desktop or Compose Multiplatform UI that responds to clicks, hover, right-click, scroll wheel, or modifier keys.
- The test must drive a keyboard shortcut (e.g. `Ctrl+S`, `Shift+Tab`, arrow-key navigation).
- The test must verify hover-only behaviour: tooltip pop, color change on `Modifier.hoverable`, mouse-only ripple state.
- The test simulates a drag-and-drop with a mouse pointer.
- The test must combine modalities — for example "hover at (x, y) then click and hold while pressing `Shift`".

## When NOT to use this skill

- The interaction is a finger tap or drag — use `../injecting-touch-gestures/SKILL.md`. Mouse is not interchangeable with touch on Android device tests.
- A simple `performClick()` already routes through the right modality per platform — use `../clicking-and-scrolling/SKILL.md`.
- The test types into a `TextField` — use `../entering-text/SKILL.md`. Key input here is for shortcuts, not text entry.
- The test waits for a hover-driven animation to settle — pause the clock first per `../../synchronization/testing-animations-deterministically/SKILL.md`.

## Prerequisites

- `androidx.compose.ui:ui-test-junit4` (or `ui-test` for `runComposeUiTest`) configured per `../../setup/configuring-test-dependencies/SKILL.md`.
- For mouse hover events on Android: API level that supports hover (typically API 23+) — Robolectric host tests work but check `@Config(minSdk = …)`.
- For key input that targets a focusable node: the node must be focusable (e.g. wrapped in `Modifier.focusable()` or `Modifier.focusRequester(...)`). Otherwise key events route to the focused root.

## Workflow

### Mouse: performMouseInput

1. **Open a mouse scope** — `performMouseInput(block: MouseInjectionScope.() -> Unit)` (Actions.kt:448-461). Like touch, events are batched and flushed when the block returns; recomposition cannot interleave inside the block.

2. **Use node-local coordinates.** `(0, 0)` is the node's top-left, identical to `TouchInjectionScope`. `MouseInjectionScope` extends `InjectionScope` so `center`, `topLeft`, `bottomRight`, `percentOffset(...)` are all available (InjectionScope.kt:34-206).

3. **Pick the right helper.** All in `MouseInjectionScope.kt`:

| Helper | Default | What it does |
|---|---|---|
| `click(position = center, button = MouseButton.Primary)` | primary click at center | press → 60 ms wait → release |
| `rightClick(position = center)` | secondary click at center | shorthand for `click(position, MouseButton.Secondary)` |
| `doubleClick(position = center, button = Primary)` | midway between min/max double-tap window | `click(); advanceEventTime(delay); click()` |
| `tripleClick(position = center, button = Primary)` | same delays as double | three sequential clicks |
| `longClick(position = center, button = Primary)` | `longPressTimeoutMillis + 100` ms hold | press, hold, release |
| `animateMoveTo(position, durationMillis = 300)` | 300 ms | streams move events along a linear path |
| `animateMoveBy(delta, durationMillis = 300)` | relative form | — |
| `animateMoveAlong(curve, durationMillis = 300)` | arbitrary curve | — |
| `dragAndDrop(start, end, button = Primary, durationMillis = 300)` | primary drag | `updatePointerTo(start); press; animateMoveTo(end); release` |
| `smoothScroll(scrollAmount, durationMillis = 300, scrollWheel = Vertical)` | vertical | streams scroll events |

4. **Drop to low-level events** when needed:
   - `press(button: MouseButton = Primary)` / `release(button: MouseButton = Primary)`.
   - `moveTo(position, delayMillis = eventPeriodMillis)` / `moveBy(delta, delayMillis = eventPeriodMillis)`.
   - `updatePointerTo(position)` / `updatePointerBy(delta)` — adjust position without sending an event.
   - `enter(position)` / `exit(position)` — explicit hover-enter / hover-exit.
   - `scroll(delta, scrollWheel = Vertical)` — single wheel tick.
   - `cancel(delayMillis)` — emit an `ACTION_CANCEL`.
   - `currentPosition: Offset` — last known mouse position.

5. **Reference `MouseButton` and `ScrollWheel`** from `Mouse.kt`:
   - `MouseButton.Primary`, `MouseButton.Secondary`, `MouseButton.Tertiary` — `expect value class MouseButton`.
   - `ScrollWheel.Vertical`, `ScrollWheel.Horizontal` — `value class ScrollWheel`.

### Keyboard: performKeyInput

1. **Open a key scope** — `performKeyInput(block: KeyInjectionScope.() -> Unit)` (Actions.kt:532-545).

2. **Send key events.** Core surface (KeyInjectionScope.kt:55-107):
   - `keyDown(key: Key)` / `keyUp(key: Key)` — primitives. Throw `IllegalStateException` if state is invalid (already-down, already-up).
   - `isKeyDown(key: Key): Boolean` — query injection state.
   - Modifier-state vals: `isCtrlDown`, `isAltDown`, `isShiftDown`, `isMetaDown`, `isFnDown` (KeyInjectionScope.kt:236-269), `isCapsLockOn`, `isNumLockOn`, `isScrollLockOn` (KeyInjectionScope.kt:64-81). These reflect the **injected** state, not the host machine.

3. **Use the helpers** (KeyInjectionScope.kt:146-229):
   - `pressKey(key, pressDurationMillis = 50L)` — `keyDown(key); advanceEventTime(50); keyUp(key)`.
   - `withKeyDown(key) { … }` — runs `block` while `key` is held; auto-releases in `finally`. The held key MUST NOT be used inside `block`.
   - `withKeysDown(listOf(key1, key2)) { … }` — same but multiple keys held simultaneously.
   - `withKeyToggled(key) { … }` / `withKeysToggled(keys) { … }` — `pressKey` before and after `block`. Useful for `CapsLock`, `NumLock`, `ScrollLock`.

4. **Repeat-key behavior.** Holding a key down and advancing the **event time** (via `advanceEventTime` from `InjectionScope`) produces repeat events: the first repeat fires after 500 ms, then every 50 ms (KeyInjectionScope.kt:43-50). This is **NOT** triggered by `MainTestClock.advanceTimeBy` — that one advances the test clock, not the injection event time.

5. **Direct KeyEvent injection** — `SemanticsNodeInteraction.performKeyPress(KeyEvent): Boolean` from `KeyInputHelpers.kt:27`. Returns `true` if the event was consumed. Use it when you already have a `KeyEvent` (e.g. constructed via `KeyEvent(NativeKeyEvent(...))` for fine-grained source/scancode control); otherwise prefer the DSL.

### Multi-modal: performMultiModalInput

1. **Open a multi-modal scope** — `performMultiModalInput(block: MultiModalInjectionScope.() -> Unit)` (Actions.kt:582-594).

2. **Dispatch into sub-scopes** — `touch { … }`, `mouse { … }`, `key { … }`, `rotary { … }`, `trackpad { … }`, `indirectPointer(...)` (MultiModalInjectionScope.kt:51-90). All sub-scopes share the same injection state, so a finger left "down" in `touch` is still down when `mouse` runs next.

3. **Pick this entry point** when a single test step combines modalities — e.g. hover-while-shift-held — instead of stitching together two separate `perform.*Input` calls.

## Patterns

### Pattern: Right-click menu

```kotlin
// RIGHT
rule.onNodeWithTag(RowTag).performMouseInput {
    rightClick(center)
}
rule.onNodeWithText("Delete").assertIsDisplayed()
```

`rightClick` is shorthand for `click(position, MouseButton.Secondary)` (MouseInjectionScope.kt:356-357). The label says "right" for familiarity, but it actually triggers the secondary button — correct on left-handed mice as well.

### Pattern: Hover-driven tooltip

From `CombinedClickableTest.kt:3415-3422`:

```kotlin
rule.onNodeWithTag("myClickable").performMouseInput { enter(center) }

rule.runOnIdle {
    assertThat(interactions).hasSize(1)
    assertThat(interactions.first()).isInstanceOf(HoverInteraction.Enter::class.java)
}

rule.onNodeWithTag("myClickable").performMouseInput { exit(Offset(-1f, -1f)) }
```

`enter(position)` emits a hover-enter; `exit(position)` emits a hover-exit. Pass an off-node `Offset(-1f, -1f)` to `exit` to mimic the cursor leaving the surface.

### Pattern: Ctrl+S shortcut — verbose vs idiomatic

```kotlin
// WRONG
rule.onNodeWithTag(EditorTag).performKeyInput {
    keyDown(Key.CtrlLeft)
    pressKey(Key.S)
    keyUp(Key.CtrlLeft)
}
// WRONG because: a thrown assertion or early-return inside the block leaks Key.CtrlLeft
// in the "down" state, polluting subsequent tests. The release is not in a finally.
```

```kotlin
// RIGHT
rule.onNodeWithTag(EditorTag).performKeyInput {
    withKeyDown(Key.CtrlLeft) {
        pressKey(Key.S)
    }
}
```

`withKeyDown` releases the key in a `finally` block (KeyInjectionScope.kt:164-171). For multi-modifier shortcuts use `withKeysDown(listOf(Key.CtrlLeft, Key.ShiftLeft)) { pressKey(Key.S) }`.

### Pattern: Repeat-key autorepeat

```kotlin
rule.onNodeWithTag(InputTag).performKeyInput {
    keyDown(Key.DirectionDown)
    advanceEventTime(800)   // 500 ms initial delay + 6 repeats at 50 ms
    keyUp(Key.DirectionDown)
}
```

`advanceEventTime` is the input-dispatcher clock — it triggers repeat events at the documented cadence (first repeat at 500 ms, then every 50 ms — KeyInjectionScope.kt:43-50). **MUST NOT** substitute `rule.mainClock.advanceTimeBy(…)`: the main test clock advances Compose's frame clock, not the input dispatcher's event time, and no repeat events will be enqueued.

### Pattern: Drag-and-drop with mouse

```kotlin
rule.onNodeWithTag(SourceTag).performMouseInput {
    dragAndDrop(
        start = center,
        end = Offset(center.x + 200f, center.y),
        durationMillis = 250,
    )
}
```

`dragAndDrop` updates the pointer to `start`, presses, animates a move to `end`, then releases (MouseInjectionScope.kt:523-533). Default button is `MouseButton.Primary`; pass `button = MouseButton.Tertiary` for middle-click drag.

### Pattern: Smooth scroll

```kotlin
rule.onNodeWithTag(ScrollAreaTag).performMouseInput {
    smoothScroll(
        scrollAmount = -10f,                  // negative scrollAmount = scroll back; new content appears at the top (MouseInjectionScope.kt:537-542)
        durationMillis = 200,
        scrollWheel = ScrollWheel.Vertical,
    )
}
```

Positive `scrollAmount` reveals content from the bottom of a vertical column or from the end of a horizontal row (MouseInjectionScope.kt:537-542). For a single-tick wheel event, drop to `scroll(delta, scrollWheel)`.

### Pattern: Multi-modal — hover then click while Shift held

`MultiModalInjectionScope` exposes `mouse { }`, `key { }`, `touch { }` etc. as sub-scopes; modifier-key state from one `key { withKeyDown(...) { } }` invocation persists across the rest of the surrounding `performMultiModalInput`, so a subsequent `mouse { }` block is dispatched while Shift is still down. The `withKeyDown` block itself has `KeyInjectionScope` as its receiver — calling `mouse { }` from inside it does not compile.

```kotlin
rule.onNodeWithTag(NodeTag).performMultiModalInput {
    mouse { enter(center) }
    key { keyDown(Key.ShiftLeft) }
    mouse { click() }                 // dispatched with Shift still held
    key { keyUp(Key.ShiftLeft) }
}
```

A single `performMultiModalInput` keeps every modality in one batched flush; the modifier-key state, pointer position, and pressed buttons are shared across sub-scopes (MultiModalInjectionScope.kt). Use it when interleaving matters; otherwise two separate `performMouseInput` / `performKeyInput` calls are clearer.

### Pattern: Direct KeyEvent injection for precise control

```kotlin
import androidx.compose.ui.input.key.KeyEvent
import androidx.compose.ui.test.performKeyPress

val event = KeyEvent(NativeKeyEvent(NativeKeyEvent.ACTION_DOWN, NativeKeyEvent.KEYCODE_TAB))
rule.onRoot().performKeyPress(event)
```

`performKeyPress` returns whether the event was consumed (`KeyInputHelpers.kt:27`). It bypasses the DSL — useful for replaying a captured `KeyEvent` or testing IME-derived keystrokes.

## Mandatory rules

- **MUST** wrap modifier-key shortcuts in `withKeyDown` / `withKeysDown` so the auto-release `finally` cleans up after a thrown assertion. Manual `keyDown` / `keyUp` pairs leak modifier state across tests.
- **MUST** use `advanceEventTime` (the `InjectionScope` clock) — **NOT** `MainTestClock.advanceTimeBy` — when expecting key autorepeat events. Repeats are produced by the input dispatcher's event-time stream (KeyInjectionScope.kt:43-50).
- **MUST** keep coordinates node-local using `center`, `topLeft`, `bottomRight`, `percentOffset(...)`. Hardcoded screen coordinates make the test device-shaped — see `../injecting-touch-gestures/SKILL.md`.
- **MUST** funnel post-action assertions through `rule.runOnIdle { … }` (skydoves hot take #5).
- **MUST NOT** use `performMouseInput` to test a finger tap on Android device tests — `performClick()` resolves to a touch tap on Android and a mouse click on desktop, automatically.
- **MUST NOT** use `Thread.sleep` to wait between key or mouse events. Use `advanceEventTime` inside the block (event-time delay) or `mainClock.advanceTimeBy` outside it (frame clock delay).
- **PREFERRED:** select `performMultiModalInput` only when modalities truly interleave; otherwise separate `performMouseInput` and `performKeyInput` calls read better.

## Verification

- [ ] Every modifier-key combo uses `withKeyDown` / `withKeysDown` rather than raw `keyDown`/`keyUp` pairs.
- [ ] No `Thread.sleep` between or inside `performMouseInput` / `performKeyInput` blocks.
- [ ] No hardcoded screen coordinates — every `Offset(...)` is derived from `center`, `top*`, `bottom*`, `percentOffset(...)`, or relative deltas.
- [ ] Tests asserting key autorepeat behaviour use `advanceEventTime`, not `mainClock.advanceTimeBy`.
- [ ] Assertions following an action run inside `rule.runOnIdle { … }`.
- [ ] `./gradlew :app:connectedDebugAndroidTest` (or the equivalent host / desktop task) passes for the test under change.

## References

- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
- Compose Multiplatform testing: https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-test.html
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Actions.kt` — `performMouseInput` (Actions.kt:448), `performKeyInput` (Actions.kt:532), `performMultiModalInput` (Actions.kt:582), `performTrackpadInput`, `performRotaryScrollInput`.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/MouseInjectionScope.kt` — `press`, `release`, `moveTo`, `moveBy`, `enter`, `exit`, `scroll`, `cancel`, plus extensions `click`, `rightClick`, `doubleClick`, `tripleClick`, `longClick`, `animateMoveTo`, `animateMoveBy`, `animateMoveAlong`, `dragAndDrop`, `smoothScroll`.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/KeyInjectionScope.kt` — `keyDown`, `keyUp`, `isKeyDown`, modifier-state vals, `pressKey`, `withKeyDown`, `withKeysDown`, `withKeyToggled`, `withKeysToggled`, repeat-key contract.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/KeyInputHelpers.kt` — `performKeyPress(KeyEvent): Boolean` (KeyInputHelpers.kt:27).
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Mouse.kt` — `MouseButton.Primary` / `Secondary` / `Tertiary`, `ScrollWheel.Horizontal` / `Vertical`.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/MultiModalInjectionScope.kt` — `touch`, `mouse`, `key`, `rotary`, `trackpad`, `indirectPointer`.
- `compose/foundation/foundation/src/androidDeviceTest/.../CombinedClickableTest.kt` — `performMouseInput { enter(center) }`, `performMouseInput { exit(Offset(-1f, -1f)) }` (CombinedClickableTest.kt:3415-3422).
- skydoves — compose-performance-skills: https://github.com/skydoves/compose-performance-skills
