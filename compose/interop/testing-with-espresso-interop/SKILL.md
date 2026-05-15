---
name: testing-with-espresso-interop
description: Use this skill to mix Compose finders and Espresso `onView` in the same test — for Android `Dialog` windows, IME (soft keyboard) state, `ComposeView` inside an Android View hierarchy, or focus interactions that cross the Compose/View boundary. Covers `createAndroidComposeRule<A>()` setup, the `EspressoLink` bridge that auto-registers Compose's `IdlingResource` with Espresso, the `onRootWithViewInteraction` API, and why `Espresso.onView(...)` MUST run from the test thread (not from `runOnIdle` / `runOnUiThread`). Use when the developer mentions "Espresso onView and Compose at the same time", "ComposeView inside a View", "soft keyboard test", "Dialog focus", "compose interop test", or shows a test that calls Espresso from inside `runOnIdle` and deadlocks.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - espresso
  - interop
  - createAndroidComposeRule
  - EspressoLink
  - IdlingResource
  - onRootWithViewInteraction
  - dialog-test
  - keyboard-test
---

# Testing with Espresso Interop — One Activity, Two Test Frameworks, One Bridge

The Compose `ComposeTestRule` and the Espresso `onView` API operate on the same `Activity` simultaneously. Compose synchronization (idling resources, frame clock awaits) is bridged into Espresso through a single internal `IdlingResource` named `EspressoLink`, so the developer does not register anything manually. The trap is threading: `Espresso.onView` MUST run on the test thread, not from inside `rule.runOnIdle { }` or `rule.runOnUiThread { }`. This skill encodes the canonical interop pattern from `androidx.compose.foundation`'s text-field IME tests.

## When to use this skill

- The test must operate on an Android `Dialog` window (which lives in its own `Window` and may not be in the Compose semantic tree).
- The test must verify IME (soft keyboard) state — only Espresso's `onView(supportsInputMethods()).perform(click())` makes the keyboard appear.
- The screen-under-test is a hybrid `Activity` with an Android View hierarchy that contains a `ComposeView`, or vice versa.
- The developer asks "how do I use Espresso and Compose in the same test".
- A test calls `Espresso.onView(...)` from `runOnIdle { }` and hangs / throws "cannot be run from the main thread".

## When NOT to use this skill

- The test is pure Compose; no Android View interactions exist. Use `../../patterns/structuring-a-compose-test/SKILL.md`.
- The synchronization symptom is "test passes locally, flaky on CI" without any View interop. Use `../../synchronization/synchronizing-with-idle/SKILL.md`.
- The test is for an `androidx.compose.ui.window.Dialog` (Compose dialog) — that DOES surface in the semantics tree via `isDialog()` and does NOT need Espresso. Use `../../finders/composing-semantics-matchers/SKILL.md`.

## Prerequisites

- `androidx.test.espresso:espresso-core` on `androidTestImplementation`.
- `androidx.compose.ui:ui-test-junit4` on `androidTestImplementation` and `androidx.compose.ui:ui-test-manifest` on `debugImplementation` (see `../../setup/configuring-test-dependencies/SKILL.md`).
- An `Activity` that hosts both layers — `FragmentActivity` is the standard pick; a custom `Activity` declared in `src/androidTest/AndroidManifest.xml` also works.
- The test class skeleton from `../../patterns/structuring-a-compose-test/SKILL.md`.

## Workflow

- [ ] **1. Use `createAndroidComposeRule<A>()` (v2) for the host Activity.** Cited from `compose/foundation/foundation/src/androidDeviceTest/kotlin/androidx/compose/foundation/textfield/TextFieldFocusCustomDialogTest.kt:60`:

```kotlin
import androidx.compose.ui.test.junit4.v2.createAndroidComposeRule
import androidx.fragment.app.FragmentActivity
import kotlinx.coroutines.test.StandardTestDispatcher

@get:Rule
val rule = createAndroidComposeRule<FragmentActivity>(StandardTestDispatcher())
```

- [ ] **2. Set Compose content with `rule.setContent { }`, then call `rule.waitForIdle()`.** This drains the Compose recomposer/effect queue so the next Espresso interaction sees a stable view tree.

- [ ] **3. Drive the View layer from the test thread via `Espresso.onView(...)`.** Do NOT wrap this call in `runOnIdle` or `runOnUiThread`.

```kotlin
import androidx.test.espresso.Espresso
import androidx.test.espresso.action.ViewActions
import androidx.test.espresso.matcher.ViewMatchers

rule.waitForIdle()
Espresso.onView(ViewMatchers.supportsInputMethods()).perform(ViewActions.click())
```

Cited from `TextFieldFocusCustomDialogTest.kt:117-119`. The full IME test:

```kotlin
@Test
fun keyboardShown_forFieldInAndroidDialog_…() {
    val focusRequester = FocusRequester()
    val keyboardHelper = KeyboardHelper(rule)

    rule.setContent {
        wrapContent {
            keyboardHelper.initialize()
            LaunchedEffect(Unit) { focusRequester.requestFocus() }
            BasicTextField(
                value = "",
                onValueChange = {},
                modifier = Modifier.focusRequester(focusRequester),
            )
        }
    }

    rule.waitForIdle()
    // The dialog's window must be focused for the IME to actually show.
    Espresso.onView(ViewMatchers.supportsInputMethods()).perform(ViewActions.click())

    keyboardHelper.waitForKeyboardVisibility(visible = true)
}
```

- [ ] **4. Mix Compose and Espresso assertions freely on the test thread.** The two APIs run against the same Activity.

```kotlin
rule.onNodeWithTag("submit").performClick()
Espresso.onView(ViewMatchers.withId(R.id.legacy_toast)).check(matches(ViewMatchers.isDisplayed()))
```

- [ ] **5. (Optional) Scope Compose finders to a sub-tree resolved by Espresso.** When the Activity contains multiple Compose hosts and a single Espresso `ViewInteraction` identifies which one to interact with, use `onRootWithViewInteraction(viewInteraction)`. This is supported only on `AndroidComposeTestRule`; non-Android test rules throw `"This implementation of ComposeTestRule does not support onRootWithViewInteraction."`.

```kotlin
val nestedComposeView = Espresso.onView(ViewMatchers.withId(R.id.nested_compose))
rule.onRootWithViewInteraction(nestedComposeView).onNodeWithTag("submit").performClick()
```

- [ ] **6. Trust the `EspressoLink` bridge.** Compose's `IdlingResource`s — recomposer, snapshot, frame clock — are aggregated by `ComposeIdlingResource` and surfaced to Espresso through a single `IdlingResource` named `"Compose-Espresso link"`. Cited from `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/EspressoLink.android.kt:34-84`. The framework registers and unregisters it inside `withStrategy { }` (lines 58-73). The developer does NOT need `IdlingRegistry.getInstance().register(...)` for Compose state.

- [ ] **7. Conversely, Espresso idling resources are visible to Compose.** Anything registered via `IdlingRegistry.getInstance().register(…)` is awaited by `rule.waitForIdle()` because the rule polls Espresso through the same bridge.

## Patterns

### Pattern: `Espresso.onView` from `runOnIdle` deadlocks

```kotlin
// WRONG
@Test
fun submit() {
    rule.setContent { /* Compose UI with a button that opens a Dialog */ }
    rule.onNodeWithTag("open").performClick()
    rule.runOnIdle {
        Espresso.onView(ViewMatchers.withId(R.id.confirm)).perform(ViewActions.click())
    }
}
// WRONG because: runOnIdle posts to the UI thread. Espresso.onView calls Espresso.onIdle()
// internally, and EspressoLink.runUntilIdle (EspressoLink.android.kt:75-83) explicitly
// throws on UI-thread invocations:
//   "Functions that involve synchronization (Assertions, Actions, Synchronization;
//    e.g. assertIsSelected(), doClick(), runOnIdle()) cannot be run from the main thread.
//    Did you nest such a function inside runOnIdle {}, runOnUiThread {} or setContent {}?"
```

```kotlin
// RIGHT
@Test
fun submit() {
    rule.setContent { /* Compose UI with a button that opens a Dialog */ }
    rule.onNodeWithTag("open").performClick()
    rule.waitForIdle()
    Espresso.onView(ViewMatchers.withId(R.id.confirm)).perform(ViewActions.click())
}
```

### Pattern: dialog focus via Espresso, content via Compose

```kotlin
// WRONG — assuming Compose's IME helper alone is enough
@Test
fun dialogKeyboard() {
    rule.setContent { CustomDialog { BasicTextField(/* … */) } }
    keyboardHelper.waitForKeyboardVisibility(visible = true)   // never visible
}
// WRONG because: Android Dialogs live in a separate Window. Even if the TextField requests
// focus, the soft keyboard does not appear until the dialog window itself takes input
// focus. Only Espresso can drive that — `onView(supportsInputMethods()).perform(click())`.
```

```kotlin
// RIGHT
@Test
fun dialogKeyboard() {
    rule.setContent { CustomDialog { BasicTextField(/* … */) } }
    rule.waitForIdle()
    Espresso.onView(ViewMatchers.supportsInputMethods()).perform(ViewActions.click())
    keyboardHelper.waitForKeyboardVisibility(visible = true)
}
```

### Pattern: `createComposeRule()` cannot reach an Activity

```kotlin
// WRONG
@get:Rule val rule = createComposeRule(StandardTestDispatcher())     // ComponentActivity host
@Test fun customActivityFlow() {
    Espresso.onView(ViewMatchers.withId(R.id.my_activity_view))      // does not exist
        .perform(ViewActions.click())
}
// WRONG because: createComposeRule launches the empty ComponentActivity from ui-test-manifest.
// To interact with a custom Activity's view hierarchy, launch that Activity via
// createAndroidComposeRule<MyActivity>().
```

```kotlin
// RIGHT
@get:Rule val rule = createAndroidComposeRule<MyActivity>(StandardTestDispatcher())
```

## Mandatory rules

- **MUST** call `Espresso.onView(...)` from the test thread, AFTER `rule.waitForIdle()`. **MUST NOT** wrap it in `rule.runOnIdle { }` or `rule.runOnUiThread { }` — Espresso's idle wait throws on the UI thread (`EspressoLink.android.kt:75-83`).
- **MUST** use `createAndroidComposeRule<A>(StandardTestDispatcher())` for any test that needs to address View IDs of the host Activity. `createComposeRule()` only launches the bare `ComponentActivity` from `ui-test-manifest`.
- **MUST NOT** manually register a Compose `IdlingResource` with `IdlingRegistry.getInstance()` — `EspressoLink` (a single bridge resource named `"Compose-Espresso link"`) does this for the framework. Cited at `EspressoLink.android.kt:34-84`.
- **PREFERRED:** scope Compose interactions to a Compose subtree via `rule.onRootWithViewInteraction(viewInteraction)` when an Activity hosts multiple `ComposeView`s. Calling it on a non-`AndroidComposeTestRule` implementation throws `IllegalStateException` (the source uses `error("This implementation of ComposeTestRule does not support onRootWithViewInteraction.")` at `ComposeTestRuleExt.android.kt:38-42`).
- **PREFERRED:** for soft-keyboard tests, follow `rule.setContent { … }` → `rule.waitForIdle()` → `Espresso.onView(supportsInputMethods()).perform(click())` → `keyboardHelper.waitForKeyboardVisibility(visible = true)` (`TextFieldFocusCustomDialogTest.kt:117-128`).
- **MUST NOT** use this skill for `androidx.compose.ui.window.Dialog` (Compose dialog) — those surface as semantic nodes matched by `isDialog()`. See `../../finders/composing-semantics-matchers/SKILL.md`.

## Verification

- [ ] The rule is `createAndroidComposeRule<HostActivity>(StandardTestDispatcher())` — not `createComposeRule()`.
- [ ] Every `Espresso.onView(...)` call lives at test-method scope, not inside `runOnIdle { }` / `runOnUiThread { }` / `setContent { }`.
- [ ] `rule.waitForIdle()` is called between the Compose action and the next `Espresso.onView(...)` (or vice versa).
- [ ] No manual `IdlingRegistry.getInstance().register(composeIdlingResource)` calls appear — the framework does this through `EspressoLink`.
- [ ] When multiple `ComposeView`s exist in one Activity, finders are scoped via `rule.onRootWithViewInteraction(...)`.
- [ ] No `Thread.sleep` is used to "let Espresso settle" — `rule.waitForIdle()` already polls Espresso through the bridge. See `../../synchronization/synchronizing-with-idle/SKILL.md`.

## References

- Espresso onView API: https://developer.android.com/training/testing/espresso/basics
- Compose-Espresso interop guide: https://developer.android.com/develop/ui/compose/testing#interop-uiautomator-espresso
- IME interop canonical test: `compose/foundation/foundation/src/androidDeviceTest/kotlin/androidx/compose/foundation/textfield/TextFieldFocusCustomDialogTest.kt:57-128`
- `EspressoLink` bridge implementation: `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/EspressoLink.android.kt:34-84`
- `onRootWithViewInteraction` definition: `compose/ui/ui-test-junit4/src/androidMain/kotlin/androidx/compose/ui/test/junit4/AndroidComposeTestRule.android.kt`
- `createAndroidComposeRule` v2: `compose/ui/ui-test-junit4/src/androidMain/kotlin/androidx/compose/ui/test/junit4/v2/AndroidComposeTestRule.android.kt`
- KeyboardHelper utility: `compose/foundation/foundation/src/androidDeviceTest/kotlin/androidx/compose/foundation/text/KeyboardHelper.kt`
