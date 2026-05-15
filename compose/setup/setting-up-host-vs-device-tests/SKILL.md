---
name: setting-up-host-vs-device-tests
description: Use this skill to choose between host (Robolectric/JVM) and device (instrumentation) tests for Jetpack Compose, and to configure each correctly. Covers the `androidHostTest` (a.k.a. `src/test/`) vs `androidDeviceTest` (a.k.a. `src/androidTest/`) source set split, what each flavor can and cannot drive (RenderThread, screenshots, accessibility), the `@RunWith(AndroidJUnit4::class) @Config(minSdk = 23)` setup for Robolectric, and why `Thread.sleep` is forbidden everywhere except screenshot tests waiting on the RenderThread. Use when the user reports "tests pass locally but fail on CI", asks "Robolectric vs instrumentation", mentions screenshot tests, ripple animations, accessibility checks throwing on Robolectric, or "should this test live in test/ or androidTest/".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - robolectric
  - host-test
  - device-test
  - androidTest
  - instrumentation-test
  - render-thread
  - screenshot-test
  - thread-sleep
  - mainClock-advanceTimeBy
---

# Setting Up Host vs Device Tests — Pick the Right Source Set

Compose tests run unchanged on either Robolectric (JVM, fast, no emulator) or on a real/virtual device (full Android stack, RenderThread, accessibility). The same `runComposeUiTest { setContent { … } }` block compiles in both — only the underlying `Looper` and `Choreographer` differ. This skill encodes which flavor each test should live in, the Robolectric class skeleton, and the one legitimate `Thread.sleep` exception (screenshot tests waiting on the RenderThread).

## When to use this skill

- The user is starting a Compose UI test and asks "test/ or androidTest/?".
- A test passes on a local emulator but fails on CI's Robolectric runner (or vice versa).
- The user reports `Build.FINGERPRINT == "robolectric"` warnings from `enableAccessibilityChecks(...)`.
- A screenshot test produces a black/empty PNG on the host runner.
- A ripple/`pressInteraction` test renders no ripple on Robolectric and the user is debugging why.
- The user wrote `Thread.sleep(1000)` to "wait for an animation" and is asking why it is flaky.
- The user mentions `androidDeviceTest` / `androidHostTest` source sets (used in `androidx` itself).

## When NOT to use this skill

- The dependencies are not yet wired correctly — start with `./configuring-test-dependencies/SKILL.md`.
- The choice is between `createComposeRule()` and `runComposeUiTest { }` — see `./choosing-test-rule-vs-runtest/SKILL.md`.
- The test runs in the right flavor but is flaky on idle/animation — see `../../synchronization/synchronizing-with-idle/SKILL.md` and `../../synchronization/testing-animations-deterministically/SKILL.md`.

## Prerequisites

- `androidx.compose.ui:ui-test`, `ui-test-junit4`, and `ui-test-manifest` on the correct configurations for the chosen flavor — see `./configuring-test-dependencies/SKILL.md`.
- For host tests: `org.robolectric:robolectric` on `testImplementation`, `testOptions { unitTests.isIncludeAndroidResources = true }` in the Android block.
- For device tests: a configured emulator or physical device, `androidx.test.runner.AndroidJUnitRunner` (or a Hilt subclass) as the `testInstrumentationRunner`.
- Working knowledge of `MainTestClock` semantics — see the synchronization skill set.

## Workflow

- [ ] **1. Decide the flavor by capability, not by speed.** The matrix:

  | Capability | Host (Robolectric, `src/test/`) | Device (instrumentation, `src/androidTest/`) |
  |---|---|---|
  | Recomposition + state changes | Works | Works |
  | Layout + measurement | Works | Works |
  | Touch input via `performTouchInput` | Works (synthetic events) | Works (real input pipeline) |
  | `Modifier.indication` ripples | **No** (RenderThread not driven) | Works |
  | Screenshot capture (`captureToImage`) | **No** (no real surface) | Works |
  | `enableAccessibilityChecks(...)` | **Inconclusive** — `ComposeUiTestExt.android.kt:50-53` logs `Log.w("ComposeUiTest", "Accessibility checks are currently not supported by Robolectric")` and still installs the validator, but Robolectric does not faithfully drive accessibility services (b/332778271). Treat passing as inconclusive. | Works (`@RequiresApi(34)`) |
  | Gesture detectors needing real-time clock changes (double-tap, long-press timing) | Works only with manual `mainClock.advanceTimeBy(...)` | Works automatically |
  | Activity lifecycle (real `onPause`/`onResume`) | Approximated | Real |
  | Min SDK | 23 (`internal const val RobolectricMinSdk = 23` from `compose/ui/ui-test/src/androidHostTest/.../Constants.kt`) | The module's `minSdk` |
  | Speed (rough order) | Seconds | Tens of seconds + emulator boot |
  | CI footprint | JVM only | Emulator service or Firebase Test Lab |

  Choose host for logic/recomposition/finder coverage. Choose device for anything that touches the RenderThread, real animations involving `Modifier.indication`, screenshot golden tests, or accessibility validation.

- [ ] **2. Place files in the matching source set.** The androidx convention is:

  ```text
  src/
  ├── androidHostTest/        # = test/ — Robolectric on JVM
  │   └── kotlin/.../FooTest.kt
  ├── androidDeviceTest/      # = androidTest/ — instrumentation
  │   └── kotlin/.../FooScreenshotTest.kt
  └── androidCommonTest/      # helpers shared by both
      └── kotlin/.../FooTestHelpers.kt
  ```

  In a typical app module without KMP source sets, the equivalent is `src/test/` (host) and `src/androidTest/` (device). The **same** `runComposeUiTest { setContent { … } }` body compiles unchanged in both — only the Looper/Choreographer differs at runtime.

- [ ] **3. Configure the host test class skeleton.** Copy the canonical androidx pattern from `compose/ui/ui-test/src/androidHostTest/kotlin/androidx/compose/ui/test/RobolectricComposeTest.kt`:

```kotlin
@RunWith(AndroidJUnit4::class)
@Config(minSdk = 23)
@OptIn(ExperimentalTestApi::class)
class MyHostTest {
    @Before
    fun setup() {
        // capture the master timeout so we can restore it afterwards
        masterTimeout = IdlingPolicies.getMasterIdlingPolicy()
    }

    @After
    fun tearDown() {
        masterTimeout?.let {
            IdlingPolicies.setMasterPolicyTimeout(it.idleTimeout, it.idleTimeoutUnit)
        }
    }

    @Test
    fun stateChange() = runComposeUiTest {
        setContent { ClickCounter() }
        onNodeWithText("Click me").performClick()
        onNodeWithText("Click count", substring = true).assertTextEquals("Click count: 1")
    }

    private var masterTimeout: IdlingPolicy? = null
}
```

  `AndroidJUnit4::class` delegates to Robolectric on the JVM and to `AndroidJUnit4ClassRunner` on a device — making the same class portable. `@RunWith(RobolectricTestRunner::class)` also works but ties the class to host-only.

- [ ] **4. Configure the device test class skeleton.**

```kotlin
@RunWith(AndroidJUnit4::class)
class MyDeviceTest {
    @get:Rule val rule = createComposeRule()       // or v2 import — see ./choosing-test-rule-vs-runtest/

    @Test
    fun stateChange() {
        rule.setContent { ClickCounter() }
        rule.onNodeWithText("Click me").performClick()
        rule.onNodeWithText("Click count", substring = true).assertTextEquals("Click count: 1")
    }
}
```

- [ ] **5. Compose `mainClock.advanceTimeBy(...)` for any animation/gesture-detection test.** On Robolectric, gesture detectors (double-tap, long-press) rely on the test clock advancing — they do NOT receive real wall-clock ticks. The `RobolectricComposeTest.kt` source quotes this directly: gesture detectors require manual `mainClock.advanceTimeBy(...)` because they detect events through clock changes. Same applies to any `animateFloatAsState` driven by the recomposer's frame clock — see `../../synchronization/testing-animations-deterministically/SKILL.md`.

- [ ] **6. Identify the one legitimate `Thread.sleep` use case: screenshot tests waiting on the RenderThread.** Compose's `IdlingResource` aggregates the recomposer, snapshot, and frame-clock awaiters, but the RenderThread is outside that aggregation. Ripple animations (`Modifier.indication`) and any draw-time animation owned by the platform render pipeline cannot be waited on through `mainClock.advanceTimeBy` or `waitForIdle`. From `compose/material3/material3/src/androidDeviceTest/.../ToggleButtonScreenshotTest.kt:115-123`:

```kotlin
rule.mainClock.autoAdvance = false
rule.onNode(isToggleable()).performTouchInput { down(center) }

rule.mainClock.advanceTimeByFrame()
rule.waitForIdle()                          // wait for measure
rule.mainClock.advanceTimeBy(milliseconds = 200)

// Ripples are drawn on the RenderThread, not the main (UI) thread, so we can't wait for
// synchronization. Instead just wait until after the ripples are finished animating.
Thread.sleep(300)

assertAgainstGolden("toggleButton_lightTheme_defaultToPressed")
```

  This is the only case. Anywhere else, `Thread.sleep` is a smell — it desyncs from `MainTestClock` and produces flakes that wear the developer down. Use `mainClock.advanceTimeBy(durationMs)` (test clock) for animations or `rule.waitUntil(timeoutMillis = …) { … }` (wall clock) for external state.

- [ ] **7. Enforce: NO `Thread.sleep` in host tests.** Host tests cannot drive the RenderThread anyway, so the screenshot exception does not apply. Replace every `Thread.sleep(N)` in a host test with `mainClock.advanceTimeBy(N)` (animation case) or `rule.waitUntil { … }` (external state).

- [ ] **8. Place screenshot, ripple, and accessibility tests in the device source set.** The host runner cannot satisfy them. Specifically:
  - Screenshots: `captureToImage()` requires a real `Surface`. Host returns no pixel data.
  - Ripples: `Modifier.indication` draws on the RenderThread — no observable state on host.
  - Accessibility: `enableAccessibilityChecks(...)` is `@RequiresApi(34)`. On Robolectric, both extension implementations check `Build.FINGERPRINT.lowercase() == "robolectric"`, emit a `Log.w` warning, AND still install the validator — but Robolectric does not faithfully drive accessibility services, so any pass is inconclusive (b/332778271). Run accessibility checks on a real device for trustworthy results.

- [ ] **9. Place fast logic/recomposition/state tests in the host source set.** Examples that thrive on Robolectric:
  - `StateRestorationTester.emulateSavedInstanceStateRestore()` flows.
  - Pure state-change verification (`assertTextEquals`, `assertIsOn`).
  - LazyList finder/scroll tests that don't depend on velocity-driven physics.
  - `mainClock.advanceTimeBy(durationMs)` driven animation snapshots that don't need rasterization.

## Patterns

### Pattern: WRONG vs RIGHT — screenshot test on host

```kotlin
// WRONG — placed in src/test/ (Robolectric)
@RunWith(AndroidJUnit4::class)
@Config(minSdk = 23)
class ButtonScreenshotTest {
    @Test
    fun pressed() = runComposeUiTest {
        setContent { Button(onClick = {}) { Text("OK") } }
        onNode(hasText("OK")).performTouchInput { down(center) }
        onNode(hasText("OK")).captureToImage().assertAgainstGolden("pressed")
    }
}
// WRONG because: Robolectric has no real Surface, so captureToImage() returns black/empty
// pixels and the ripple from Modifier.indication never draws (RenderThread is not driven).
// The test will produce a false-positive pass or a meaningless golden file.
```

```kotlin
// RIGHT — placed in src/androidTest/ (device)
@RunWith(AndroidJUnit4::class)
class ButtonScreenshotTest {
    @get:Rule val rule = createComposeRule()

    @Test
    fun pressed() {
        rule.setContent { Button(onClick = {}) { Text("OK") } }
        rule.mainClock.autoAdvance = false
        rule.onNode(hasText("OK")).performTouchInput { down(center) }
        rule.mainClock.advanceTimeByFrame()
        rule.waitForIdle()
        rule.mainClock.advanceTimeBy(milliseconds = 200)
        Thread.sleep(300)                          // legitimate — RenderThread for ripples
        rule.onNode(hasText("OK")).captureToImage().assertAgainstGolden("pressed")
    }
}
```

### Pattern: WRONG vs RIGHT — `Thread.sleep` in a host animation test

```kotlin
// WRONG — host test
@Test
fun fadeIn() = runComposeUiTest {
    val target = mutableStateOf(0f)
    setContent { Box(Modifier.alpha(animateFloatAsState(target.value).value)) }
    target.value = 1f
    Thread.sleep(500)                                // smell
    onNode(isRoot()).captureToImage()                // also broken on host
}
// WRONG because: Thread.sleep desyncs from MainTestClock. The test clock has not advanced,
// the recomposer has not run withFrameNanos, and the animation made zero observable progress.
```

```kotlin
// RIGHT — host test (recomposition only — no pixel capture)
@Test
fun fadeIn() = runComposeUiTest {
    mainClock.autoAdvance = false
    val target = mutableStateOf(0f)
    setContent { Box(Modifier.alpha(animateFloatAsState(target.value).value).testTag("box")) }
    runOnUiThread { target.value = 1f }
    mainClock.advanceTimeByFrame()                   // kick off
    mainClock.advanceTimeBy(durationMillis = 500)    // step forward by test clock
    onNodeWithTag("box").assertIsDisplayed()
}
```

For pixel verification of the fade, move the test to the device source set and use `captureToImage()`.

### Pattern: WRONG vs RIGHT — accessibility checks on host

```kotlin
// WRONG — host
@Test
fun submitIsLabelled() = runComposeUiTest {
    enableAccessibilityChecks()                      // logs warning, runs no checks
    setContent { Submit() }
    onNodeWithTag("submit").performClick()
}
// WRONG because: both ui-test-accessibility and ui-test-junit4-accessibility detect
// Build.FINGERPRINT.lowercase() == "robolectric", emit a Log.w, AND still install the
// validator — but Robolectric does not faithfully drive accessibility services, so the
// result is inconclusive (b/332778271). Run accessibility checks on a real device.
```

```kotlin
// RIGHT — device
@RunWith(AndroidJUnit4::class)
class SubmitA11yTest {
    @get:Rule val rule = createAndroidComposeRule<MainActivity>()

    @Test fun submitIsLabelled() {
        rule.enableAccessibilityChecks()
        rule.setContent { Submit() }
        rule.onNodeWithTag("submit").performClick()  // accessibility checks run automatically
    }
}
```

### Pattern: shared body, two source sets

The same `runComposeUiTest { ... }` body can compile in both source sets when the test only uses the common API. Place the body in `androidCommonTest/` and create two thin wrappers — one in `androidHostTest/` (with `@RunWith(AndroidJUnit4::class) @Config(minSdk = 23)`), one in `androidDeviceTest/` (no `@Config`). This is how androidx's own ui-test module exercises both flavors without code duplication. For an app module without KMP source sets, prefer keeping the body inline in whichever flavor is appropriate.

## Mandatory rules

- **MUST NOT** put screenshot / `captureToImage` / ripple / `Modifier.indication`-dependent tests in the host source set. The RenderThread is not driven; the test is meaningless even when it appears to pass.
- **MUST NOT** rely on `enableAccessibilityChecks(...)` results from a host test. Both extensions log a warning under `Build.FINGERPRINT.lowercase() == "robolectric"` and still install the validator, but Robolectric does not faithfully drive accessibility services so the result is inconclusive — run accessibility checks on a real device API 34+.
- **MUST NOT** use `Thread.sleep` in a host test under any circumstance. The screenshot exception does not apply (host has no RenderThread).
- **MUST NOT** use `Thread.sleep` in a device test except when waiting on the RenderThread for ripple/screenshot golden capture. Skydoves hot take #7: `Thread.sleep` is a smell. Anywhere else, replace it with `mainClock.advanceTimeBy(durationMs)` (test clock) or `rule.waitUntil(timeoutMillis) { ... }` (wall clock) — see `../../synchronization/synchronizing-with-idle/SKILL.md`.
- **MUST** annotate Robolectric host tests with `@Config(minSdk = 23)` or higher. Lower SDK levels are not supported by androidx's host test infrastructure (`internal const val RobolectricMinSdk = 23`).
- **MUST** annotate gesture-detection or animation host tests to step the clock manually with `mainClock.advanceTimeBy(...)`. Robolectric does not advance the test clock from real-time signals.
- **MUST** set `testOptions { unitTests.isIncludeAndroidResources = true }` in the module's `android { }` block so Robolectric can read merged resources during host Compose tests.
- **PREFERRED:** start with a host test for fast feedback. Move to a device test only when capability requires it (RenderThread, accessibility, real input timing).
- **PREFERRED:** when a test must work in both flavors, factor the body into `androidCommonTest/` (KMP) or a helper function the two flavors call.

## Verification

- [ ] `./gradlew :<module>:testDebugUnitTest` runs and passes for all host tests; `./gradlew :<module>:connectedDebugAndroidTest` runs and passes for all device tests.
- [ ] No `Thread.sleep` appears in any host test (`grep -r "Thread.sleep" src/test src/androidHostTest`).
- [ ] No `captureToImage`, no `enableAccessibilityChecks`, and no test that depends on `Modifier.indication` ripples appears in the host source set.
- [ ] Every `Thread.sleep` in the device source set has a comment explaining the RenderThread / screenshot rationale.
- [ ] Robolectric host test classes carry `@RunWith(AndroidJUnit4::class)` (or `@RunWith(RobolectricTestRunner::class)`) AND `@Config(minSdk = 23)` (or higher).
- [ ] Animation/gesture host tests set `mainClock.autoAdvance = false` and call `mainClock.advanceTimeBy(durationMs)` explicitly.

## References

- Compose testing overview (Android Developers): https://developer.android.com/develop/ui/compose/testing
- Robolectric — Compose support: http://robolectric.org/
- Compose UI release notes: https://developer.android.com/jetpack/androidx/releases/compose-ui
- Testing animations (Android Developers): https://developer.android.com/develop/ui/compose/animation/testing
- `compose/ui/ui-test/src/androidHostTest/kotlin/androidx/compose/ui/test/RobolectricComposeTest.kt` — canonical host class skeleton, `@RunWith(AndroidJUnit4::class) @Config(minSdk = RobolectricMinSdk)`, gesture-detector clock comment, `IdlingPolicies` setup/teardown.
- `compose/ui/ui-test/src/androidHostTest/kotlin/androidx/compose/ui/test/Constants.kt` — `internal const val RobolectricMinSdk = 23`.
- `compose/material3/material3/src/androidDeviceTest/kotlin/androidx/compose/material3/ToggleButtonScreenshotTest.kt:115-123` — the canonical legitimate `Thread.sleep(300)` waiting on the RenderThread for ripple completion before `assertAgainstGolden`.
- `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/RobolectricIdlingStrategy.android.kt` — Robolectric idling strategy that drives the host idle loop.
- `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeIdlingResource.android.kt` — the recomposer + snapshot + frame-clock aggregator (caps at 100 frames/call); does NOT include the RenderThread, which is why ripple tests need `Thread.sleep`.
