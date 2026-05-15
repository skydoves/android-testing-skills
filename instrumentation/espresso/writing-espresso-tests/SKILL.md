---
name: writing-espresso-tests
description: Use this skill to write Espresso 3.7.0 tests against Android Views — `onView(matcher).perform(action).check(matches(...))`, `onData(...)` for `AdapterView`, `RecyclerViewActions`, `Intents.intended/intending`, and `IdlingResource` registration. Covers the full `ViewMatchers` / `ViewActions` / `ViewAssertions` catalog, the `RootMatchers.DEFAULT` exclusion of dialogs and popups (the `inRoot(isDialog())` requirement), the `BottomNavigationViewActions` doesn't-exist gotcha in 3.7.0 contrib, and the `PickerActions.setDate(year, month, day)` 1-12 month convention. Use when the user reports `NoMatchingViewException`, `AmbiguousViewMatcherException`, `PerformException`, asks "how do I click in a dialog", "Espresso typeText doesn't fire", "intent stub for camera", or "Espresso vs UiAutomator".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - espresso
  - onView
  - ViewMatchers
  - ViewActions
  - ViewAssertions
  - RootMatchers
  - IdlingResource
  - IdlingPolicies
  - RecyclerViewActions
  - Intents
  - PickerActions
  - NoMatchingViewException
---

# Writing Espresso Tests — The Idle-Then-Act Model for Android Views

Espresso is the canonical instrumentation-time UI test framework for Android **Views** (not Compose — for Compose, use the `compose/` skill set; for cross-Compose-View interop see `../../../compose/interop/testing-with-espresso-interop/SKILL.md`). Each `onView(...).perform(...).check(...)` call internally pumps the main looper to idle, finds exactly one matching view, runs the action, and pumps to idle again before checking. This skill encodes the full matcher/action/assertion catalog, the `RootMatchers.DEFAULT` trap, and the per-version gotchas.

## When to use this skill

- The user is testing an Activity, Fragment, or Dialog whose UI is built with Android Views (XML layouts, AppCompat, Material View components).
- A test fails with `NoMatchingViewException`, `AmbiguousViewMatcherException`, or `PerformException`.
- A click on an `AlertDialog` button silently does nothing — `RootMatchers.DEFAULT` excludes dialogs.
- The user wants to stub `Intent.startActivityForResult` for camera, contacts, or share flows.
- The user wants to register an `IdlingResource` for a long-running task that Espresso can't see.
- The user is debugging a `RecyclerView` interaction.
- The user mentions `BottomNavigationViewActions` and is surprised it isn't on the classpath in Espresso 3.7.0 contrib.

## When NOT to use this skill

- The UI is Jetpack Compose — use `compose/` skills (`../../../compose/finders/finding-nodes-by-tag-text-content/SKILL.md`, etc.). Mixed Compose+View hosts: `../../../compose/interop/testing-with-espresso-interop/SKILL.md`.
- The test crosses app boundaries (Settings, dialer, system UI) — `InjectEventSecurityException` from Espresso. Use `../../uiautomator/cross-app-tests-with-uiautomator/SKILL.md`.
- The runner stack is not yet set up — start with `../../runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.
- The Activity has not been launched — pair with `../../scenarios/launching-activities-with-activityscenario/SKILL.md`.

## Prerequisites

- Runner stack from `../../runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.
- Espresso artifacts (3.7.0 GA per `docs/CORPUS.md` H.1):

```kotlin
dependencies {
    androidTestImplementation("androidx.test.espresso:espresso-core:3.7.0")
    androidTestImplementation("androidx.test.espresso:espresso-contrib:3.7.0")     // RecyclerViewActions, PickerActions
    androidTestImplementation("androidx.test.espresso:espresso-intents:3.7.0")     // Intents.intended/intending
    androidTestImplementation("androidx.test.espresso:espresso-accessibility:3.7.0")
    androidTestImplementation("androidx.test.espresso:espresso-web:3.7.0")         // WebView
    implementation("androidx.test.espresso:espresso-idling-resource:3.7.0")        // for production code that exposes IdlingResources
}

android.testOptions.animationsDisabled = true   // hermetic
```

- An `Activity` (or Fragment) launched via `ActivityScenario` / `ActivityScenarioRule` / `FragmentScenario`.

## Workflow

- [ ] **1. Find views with `onView(matcher)`.** The matcher is a Hamcrest `Matcher<View>`. Identity-first matchers (most preferred): `withId(R.id.button)`, `withTagKey(int)`, `withResourceName(String)`. Text/desc fallback: `withText`, `withContentDescription`, `withHint`. Source: `androidx.test.espresso.matcher.ViewMatchers` (lines 79-1000+, R3 lines 180-280):

```kotlin
import androidx.test.espresso.Espresso.onView
import androidx.test.espresso.matcher.ViewMatchers.*
import org.hamcrest.Matchers.allOf

onView(withId(R.id.submit_button))
onView(allOf(withText("Submit"), isEnabled()))
onView(withContentDescription("Open menu"))
onView(allOf(withId(R.id.subtitle), hasDescendant(withText("Hello"))))
```

- [ ] **2. Compose state-aware matchers.** Common state matchers from `ViewMatchers`: `isDisplayed`, `isCompletelyDisplayed`, `isDisplayingAtLeast(percent)`, `isEnabled`/`isNotEnabled`, `isFocused`, `isChecked`/`isNotChecked`, `isSelected`, `withEffectiveVisibility(Visibility.VISIBLE|INVISIBLE|GONE)`, `withAlpha(float)`. Hierarchy matchers: `withParent`, `hasDescendant`, `withChild`, `isRoot`, `withClassName`, `isAssignableFrom(Class)`. Layout-relative: `hasSibling`, `isDescendantOfA`. Source: R3 lines 180-280.

- [ ] **3. Act with `ViewActions` via `.perform(action)`.** From `androidx.test.espresso.action.ViewActions`:

```kotlin
import androidx.test.espresso.action.ViewActions.*

onView(withId(R.id.email)).perform(typeText("test@example.com"), closeSoftKeyboard())
onView(withId(R.id.submit_button)).perform(click())
onView(withId(R.id.input)).perform(replaceText("new"), pressImeActionButton())
onView(withId(R.id.list)).perform(swipeUp())
onView(withId(R.id.scroller)).perform(scrollTo())   // requires single-direction ScrollView
```

Click family: `click`, `longClick`, `doubleClick`. Text: `typeText`, `replaceText`, `clearText`, `pressKey`, `pressImeActionButton`, `closeSoftKeyboard`. Gesture: `swipeUp`, `swipeDown`, `swipeLeft`, `swipeRight`, `scrollTo`. Source: `androidx.test.espresso.action.ViewActions`.

- [ ] **4. Assert with `ViewAssertions` via `.check(...)`.** From `androidx.test.espresso.assertion.ViewAssertions`:

```kotlin
import androidx.test.espresso.assertion.ViewAssertions.*
import androidx.test.espresso.matcher.ViewMatchers.*

onView(withId(R.id.title)).check(matches(withText("Welcome")))
onView(withId(R.id.banner)).check(matches(isDisplayed()))
onView(withId(R.id.banner)).check(doesNotExist())
onView(withId(R.id.list)).check(selectedDescendantsMatch(withId(R.id.row), isCompletelyDisplayed()))
```

`matches(matcher)` passes when the resolved view matches; `doesNotExist()` passes when the matcher resolved zero views (NOT when one was found and is invisible — use `matches(not(isDisplayed()))` for that). Source: R3 lines 290-311.

- [ ] **5. Recognize the implicit idle.** Every `onView(...).perform(...)` and `.check(...)` call internally:
  1. Submits the action on the main thread via the dagger-injected `mainThreadExecutor`.
  2. Calls `uiController.loopMainThreadUntilIdle()` until: the main `MessageQueue` is empty, all `IdlingResource`s registered with `IdlingRegistry` report `isIdleNow == true`, all registered non-main `Looper`s are idle, and the AsyncTask thread pool is idle.
  3. Resolves the view (`viewMatcher` against the current `Root`'s hierarchy). 0 → `NoMatchingViewException`; 2+ → `AmbiguousViewMatcherException`.
  4. Checks `viewAction.getConstraints().matches(view)` → throws `PerformException` if false.
  5. Calls `viewAction.perform(uiController, view)` (or `viewAssertion.check(view, null)`).
  6. Loops the main thread to idle again before returning.

This is the **idle-then-act** model. Tests that work locally but fail on CI are usually missing an `IdlingResource` for a non-Looper-driven async path (custom `ExecutorService`, OkHttp's dispatcher, etc.). Source: R3 lines 165-178, 811-820.

- [ ] **6. For dialogs and popups, override the root with `inRoot(...)`.** `RootMatchers.DEFAULT` excludes dialogs, popups, and toasts (it matches "the focused activity window"). For an `AlertDialog` button:

```kotlin
import androidx.test.espresso.matcher.RootMatchers.*

onView(withText("Submit")).inRoot(isDialog()).perform(click())
onView(withText("Copy")).inRoot(isPlatformPopup()).perform(click())   // overflow menu, popup window
onView(withText("Saved")).inRoot(isPlatformPopup()).check(matches(isDisplayed()))   // toast
```

`RootMatchers.DEFAULT` is `allOf(hasWindowLayoutParams(), allOf(anyOf(allOf(isDialog(), withDecorView(hasWindowFocus())), isSubwindowOfCurrentActivity()), isFocusable()))` per R3 line 311 — that's why a generic dialog button click silently misses.

- [ ] **7. For `RecyclerView`, use `RecyclerViewActions` from contrib.** `onView(...).perform(...)` against a `RecyclerView`'s row directly does NOT work because the row may be off-screen (no view to find):

```kotlin
import androidx.test.espresso.contrib.RecyclerViewActions.*

onView(withId(R.id.list))
    .perform(scrollToPosition<RecyclerView.ViewHolder>(20))

onView(withId(R.id.list))
    .perform(actionOnItemAtPosition<RecyclerView.ViewHolder>(3, click()))

onView(withId(R.id.list))
    .perform(scrollTo<RecyclerView.ViewHolder>(hasDescendant(withText("Pikachu"))))

onView(withId(R.id.list))
    .perform(actionOnItem<RecyclerView.ViewHolder>(hasDescendant(withText("Pikachu")), click()))
```

All `RecyclerViewActions` require `isAssignableFrom(RecyclerView.class) AND isDisplayed()` constraint (R3 line 428).

- [ ] **8. For `AdapterView` (ListView, Spinner), use `onData(...)` not `onView(...)`.**

```kotlin
import androidx.test.espresso.Espresso.onData
import org.hamcrest.Matchers.allOf

onData(allOf(`is`(instanceOf(String::class.java)), `is`("Item 5")))
    .inAdapterView(withId(R.id.list))
    .perform(click())
```

`onData` scrolls the adapter to bring the item on-screen before resolving its view.

- [ ] **9. Stub and verify intents with `Intents`.** Init in `@Before`, release in `@After`. The `IntentsRule` (3.5+) handles both:

```kotlin
import androidx.test.espresso.intent.rule.IntentsRule
import androidx.test.espresso.intent.Intents.*
import androidx.test.espresso.intent.matcher.IntentMatchers.*

@get:Rule val intentsRule = IntentsRule()

@Test
fun openCameraStub() {
    intending(hasAction(MediaStore.ACTION_IMAGE_CAPTURE))
        .respondWith(Instrumentation.ActivityResult(Activity.RESULT_OK, Intent()))

    onView(withId(R.id.take_photo)).perform(click())

    intended(allOf(
        hasAction(MediaStore.ACTION_IMAGE_CAPTURE),
        hasFlag(Intent.FLAG_GRANT_READ_URI_PERMISSION),
    ))
}
```

`IntentsTestRule` is `@Deprecated` per `docs/CORPUS.md` H.7 — use `IntentsRule` or call `Intents.init()` / `Intents.release()` manually.

- [ ] **10. Register `IdlingResource` for non-Looper async work.** Espresso watches the main looper and the `AsyncTask` pool by default; for OkHttp, custom `ExecutorService`, or sensor pipelines you must surface idleness:

```kotlin
class CountingIdlingResource(private val name: String) : IdlingResource {
    private val counter = AtomicInteger(0)
    private var callback: IdlingResource.ResourceCallback? = null

    override fun getName() = name
    override fun isIdleNow() = counter.get() == 0
    override fun registerIdleTransitionCallback(cb: IdlingResource.ResourceCallback) { callback = cb }

    fun increment() { counter.incrementAndGet() }
    fun decrement() { if (counter.decrementAndGet() == 0) callback?.onTransitionToIdle() }
}

@Before fun setUp() = IdlingRegistry.getInstance().register(myIdler)
@After fun tearDown() = IdlingRegistry.getInstance().unregister(myIdler)

// Master policy default is 60 seconds; setIdlingResourceTimeout default is 26 seconds.
// For debugger sessions, opt out of timeouts entirely with
//   IdlingPolicies.setMasterPolicyTimeoutWhenDebuggerAttached(false)
// — there is no "-1 = infinite" magic value.
IdlingPolicies.setMasterPolicyTimeout(60, TimeUnit.SECONDS)
IdlingPolicies.setIdlingResourceTimeout(45, TimeUnit.SECONDS)
```

Wrap network calls / async work in `increment()` / `decrement()` from production code (the `:espresso-idling-resource` artifact is on `implementation`, not `androidTestImplementation`).

- [ ] **11. Read errors precisely.**

  - `NoMatchingViewException` — the matcher resolved zero views in the current root. Confirm the view is currently displayed; if it is in a dialog, add `.inRoot(isDialog())`.
  - `AmbiguousViewMatcherException` — 2+ views matched. Tighten with `allOf(...)`.
  - `PerformException` — the action's constraint failed (e.g. `click()` requires `isDisplayingAtLeast(90)`). The exception message includes the constraint description.
  - `InjectEventSecurityException` — an event would cross to a foreign window. The test reached system UI. Use UiAutomator instead.

## Patterns

### Pattern: WRONG vs RIGHT — clicking a dialog button

```kotlin
// WRONG
onView(withText("Submit")).perform(click())
// WRONG because: RootMatchers.DEFAULT (line 51 of RootMatchers.java) excludes dialogs.
// The matcher does not see the dialog window. Result: NoMatchingViewException ("No views in
// hierarchy found matching: with text Submit"), even though the dialog is clearly on screen.
```

```kotlin
// RIGHT
import androidx.test.espresso.matcher.RootMatchers.isDialog
onView(withText("Submit")).inRoot(isDialog()).perform(click())
```

Same applies to popup menus (`isPlatformPopup()`) and toasts.

### Pattern: WRONG vs RIGHT — `BottomNavigationView` interaction (Espresso 3.7.0 contrib)

```kotlin
// WRONG
import androidx.test.espresso.contrib.BottomNavigationViewActions
onView(withId(R.id.bottom_nav)).perform(BottomNavigationViewActions.selectItem(R.id.tab_profile))
// WRONG because: BottomNavigationViewActions does NOT exist in espresso-contrib 3.7.0.
// The class was historically documented but never shipped in 3.7.0's contrib jar (R3 lines 91-93).
// Compile fails with "unresolved reference".
```

```kotlin
// RIGHT — click the underlying menu item view directly
import com.google.android.material.bottomnavigation.BottomNavigationView
import org.hamcrest.Matchers.allOf

onView(allOf(
    withId(R.id.tab_profile),
    isDescendantOfA(isAssignableFrom(BottomNavigationView::class.java)),
)).perform(click())
```

### Pattern: WRONG vs RIGHT — `PickerActions.setDate` month convention

```kotlin
// WRONG
import androidx.test.espresso.contrib.PickerActions
onView(withClassName(equalTo(DatePicker::class.java.name)))
    .perform(PickerActions.setDate(2026, 4, 15))   // intends "May 15, 2026"
// WRONG because: PickerActions.setDate uses 1-12 month convention (it subtracts 1 internally
// before calling DatePicker.updateDate which uses 0-11). Passing 4 sets April, not May.
// Source: R3 line 460, PickerActions.java line 49.
```

```kotlin
// RIGHT — May = 5
onView(withClassName(equalTo(DatePicker::class.java.name)))
    .perform(PickerActions.setDate(2026, 5, 15))
```

### Pattern: WRONG vs RIGHT — `Thread.sleep` to wait for an animation

```kotlin
// WRONG
onView(withId(R.id.fab)).perform(click())
Thread.sleep(500)            // wait for the FAB exit animation
onView(withId(R.id.fab)).check(doesNotExist())
// WRONG because: Thread.sleep is a smell. Espresso's idle pump already waits for animations
// driven through the main looper; if it doesn't, register an IdlingResource for the offending
// async path. See `docs/CORPUS.md` D.7 — "Thread.sleep is a smell".
```

```kotlin
// RIGHT — disable animations OR register an IdlingResource
android.testOptions.animationsDisabled = true     // for property animators driven via Choreographer
// And/or, for non-Looper waits:
IdlingPolicies.setMasterPolicyTimeout(45, TimeUnit.SECONDS)
IdlingRegistry.getInstance().register(myAnimationIdler)
```

### Pattern: scrolling a `RecyclerView` to an off-screen row before clicking it

```kotlin
import androidx.test.espresso.contrib.RecyclerViewActions.*
import androidx.test.espresso.matcher.ViewMatchers.hasDescendant

onView(withId(R.id.list))
    .perform(scrollTo<RecyclerView.ViewHolder>(hasDescendant(withText("Snorlax"))))
onView(withId(R.id.list))
    .perform(actionOnItem<RecyclerView.ViewHolder>(hasDescendant(withText("Snorlax")), click()))
```

`scrollTo` and `actionOnItem` use a HOLDER matcher (matches the underlying `RecyclerView.ViewHolder`'s itemView), not a `Matcher<View>` against the screen.

## Mandatory rules

- **MUST** add `.inRoot(isDialog())` for any matcher targeting an `AlertDialog` content view, `.inRoot(isPlatformPopup())` for popup menus and toasts. `RootMatchers.DEFAULT` excludes them.
- **MUST** prefer `withId` / `withTagKey` / `withResourceName` over `withText`. Text matchers are i18n-fragile and churn with copy edits.
- **MUST NOT** rely on `Thread.sleep` to wait for animations or async work. Use `IdlingResource` + `IdlingPolicies.setMasterPolicyTimeout`, and `testOptions.animationsDisabled = true`.
- **MUST NOT** import `androidx.test.espresso.contrib.BottomNavigationViewActions` — the class is not in `espresso-contrib:3.7.0`. Use the `isDescendantOfA(isAssignableFrom(BottomNavigationView::class.java))` workaround.
- **MUST** pass 1-12 (not 0-11) for the month parameter of `PickerActions.setDate(year, month, day)`. The action subtracts 1 internally before calling `DatePicker.updateDate`.
- **MUST** call `Intents.init()` / `Intents.release()` (or use `IntentsRule`) — `intended` / `intending` only work between those calls.
- **MUST** put `:espresso-idling-resource:3.7.0` on `implementation` (production), not `androidTestImplementation`. The production code is what increments / decrements the resource.
- **MUST NOT** use `IntentsTestRule` — it is `@Deprecated`. Use `IntentsRule` (3.5+) or manual `init`/`release`.
- **PREFERRED:** disable animations globally via `testOptions.animationsDisabled = true`; do not toggle `Settings.Global` flags by hand.
- **PREFERRED:** read every `NoMatchingViewException` message — it includes a hierarchy dump. The view you wanted may be present but in a different `Root`.

## Verification

- [ ] `grep -r "Thread.sleep" src/androidTest/` returns nothing (or only screenshot-test files where the RenderThread is the wait target).
- [ ] `grep -r "BottomNavigationViewActions" src/androidTest/` returns nothing.
- [ ] `grep -r "IntentsTestRule" src/androidTest/` returns nothing — use `IntentsRule` or manual init/release.
- [ ] Every `inRoot(isDialog())` matches an `AlertDialog`-class API in production; otherwise the test passes against the wrong root.
- [ ] `./gradlew :<module>:connectedDebugAndroidTest` runs without `NoMatchingViewException` or `PerformException` from the catalog above.
- [ ] `testOptions.animationsDisabled = true` is set in the module's `android` block.
- [ ] All registered `IdlingResource`s call `IdlingRegistry.getInstance().unregister(...)` in `@After` (or use a JUnit rule).

## References

- Android Developers — Espresso overview: https://developer.android.com/training/testing/espresso
- Android Developers — Espresso cheat sheet (matchers, actions, assertions): https://developer.android.com/training/testing/espresso/cheat-sheet
- Android Developers — IdlingResource guide: https://developer.android.com/training/testing/espresso/idling-resource
- AndroidX Test (Espresso) release notes: https://developer.android.com/jetpack/androidx/releases/test
- `androidx/test/espresso/matcher/ViewMatchers.java` (lines 79-1000+) — full matcher catalog. Identity (`withId`, `withTagKey`, `withResourceName`); text (`withText`, `withSubstring`, `withHint`); state (`isDisplayed`, `isEnabled`, `isChecked`, `withEffectiveVisibility`); hierarchy (`withParent`, `hasDescendant`, `withChild`, `isRoot`, `isDescendantOfA`).
- `androidx/test/espresso/matcher/RootMatchers.java` line 51 — `DEFAULT = allOf(hasWindowLayoutParams(), allOf(anyOf(allOf(isDialog(), withDecorView(hasWindowFocus())), isSubwindowOfCurrentActivity()), isFocusable()))`. The trap.
- `androidx/test/espresso/action/ViewActions.java` — `click`, `longClick`, `doubleClick`, `typeText`, `replaceText`, `clearText`, `pressKey`, `swipeUp/Down/Left/Right`, `scrollTo`, `pressImeActionButton`, `closeSoftKeyboard`.
- `androidx/test/espresso/assertion/ViewAssertions.java` lines 51-118 — `matches`, `doesNotExist`, `selectedDescendantsMatch`.
- `androidx/test/espresso/contrib/RecyclerViewActions.java` lines 200-249 — `scrollToPosition`, `scrollTo`, `actionOnItemAtPosition`, `actionOnItem`. Constraint: `isAssignableFrom(RecyclerView) AND isDisplayed`.
- `androidx/test/espresso/contrib/PickerActions.java` lines 36-100 — `setDate(year, monthOfYear, dayOfMonth)` with monthOfYear 1-12 (line 49 KDoc). `setTime(hours, minutes)`.
- `androidx/test/espresso/IdlingPolicies.java` lines 96-112 — `setMasterPolicyTimeout`, `setIdlingResourceTimeout`, `setMasterPolicyTimeoutWhenDebuggerAttached`.
- `androidx/test/espresso/intent/Intents.java` — `init`, `release`, `intended`, `intending`. `androidx.test.espresso.intent.rule.IntentsRule` is the modern JUnit4 wrapper.
- `tasks/research/R3-espresso.md` — full Espresso 3.7.0 deep-dive. Lines 91-93 (BottomNavigationViewActions absent), 311 (RootMatchers.DEFAULT), 456-460 (PickerActions month convention), 811-820 (idle-then-act).
- `docs/CORPUS.md` Section H.5 — Espresso essentials and `RootMatchers.DEFAULT` exclusions.
