---
name: launching-activities-with-activityscenario
description: Use this skill to launch, drive, and tear down an Activity from an instrumentation test using `ActivityScenario` and the JUnit4 wrapper `ActivityScenarioRule`. Covers `ActivityScenario.launch<A>()`, `launch(intent)`, `launchActivityForResult`, `moveToState(Lifecycle.State.*)`, `recreate()`, `onActivity { }`, the `Closeable` / `use { }` idiom, and the manifest declaration trap. Establishes that `androidx.test.ext.junit.rules.ActivityScenarioRule` is the single canonical FQN — `androidx.test.rule.ActivityScenarioRule` does NOT exist; the package only ships the deprecated `ActivityTestRule`. Use when the user reports `ActivityNotFoundException`, `ActivityTestRule deprecated`, "how do I rotate the activity in a test", "how do I read setResult from a test", "how do I survive recreate", "Cannot run onActivity since Activity is not alive", or "test deadlocks on launch".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-instrumentation-test
  - ActivityScenario
  - ActivityScenarioRule
  - ActivityTestRule-deprecated
  - moveToState
  - recreate
  - onActivity
  - launchActivityForResult
  - Lifecycle-State
  - configuration-change
  - Closeable
---

# Launching Activities with ActivityScenario — The Canonical ActivityTestRule Replacement

`ActivityScenario` is the modern replacement for the deprecated `ActivityTestRule`. It provides a thread-safe, lifecycle-aware handle to drive an `Activity` through `CREATED` / `STARTED` / `RESUMED` / `DESTROYED`, recreate it for configuration-change emulation, and post work onto its UI thread via `onActivity { }`. The JUnit4 wrapper `ActivityScenarioRule` calls `launch` in `before` and `close` in `after`. This skill encodes the canonical FQN trap, the threading rules, and the manifest requirement.

## When to use this skill

- The user is starting a new instrumentation test that needs an Activity host.
- The user is migrating from `ActivityTestRule` (deprecated) and asks for the replacement.
- A test fails with `ActivityNotFoundException` and the Activity is missing from the test APK manifest.
- A `recreate()` test "loses" state and the user is debugging `remember` vs `rememberSaveable`.
- The user posts an action onto the UI thread and the test deadlocks (`onActivity` called from the wrong thread).
- A test calls `setResult(...) + finish()` and wants to read the `Activity.RESULT_*` back.
- The user mentions `Settings.Global.ALWAYS_FINISH_ACTIVITIES` ("Don't keep activities") as a way to test recreation — `ActivityScenario` rejects it.

## When NOT to use this skill

- The host is a Fragment, not an Activity — see `../launching-fragments-with-fragmentscenario/SKILL.md`.
- The host is `androidx.activity.ComponentActivity` for Compose tests — `createComposeRule()` already wraps the scenario; see `../../../compose/setup/configuring-test-dependencies/SKILL.md`.
- The runner / dependency stack is not yet set up — start with `../../runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.
- The user needs to interact with system UI or another app — see `../../uiautomator/cross-app-tests-with-uiautomator/SKILL.md`.
- The user is choosing between host (Robolectric) and device tests — see `../../../compose/setup/setting-up-host-vs-device-tests/SKILL.md`.

## Prerequisites

- `androidTestImplementation("androidx.test:core:1.7.0")` for `androidx.test.core.app.ActivityScenario`.
- `androidTestImplementation("androidx.test.ext:junit:1.3.0")` for `androidx.test.ext.junit.rules.ActivityScenarioRule`.
- `androidTestImplementation("androidx.test:runner:1.7.0")` and `testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"` from the runner skill.
- The test Activity declared in `src/androidTest/AndroidManifest.xml`, `src/debug/AndroidManifest.xml`, or the production manifest.

## Workflow

- [ ] **1. Pick the launch entry point.** All four are static factories on `androidx.test.core.app.ActivityScenario`:

```kotlin
import androidx.test.core.app.ActivityScenario
import androidx.test.core.app.launchActivity                 // Kotlin reified extension

// Plain class-based launch
val s1: ActivityScenario<MyActivity> = ActivityScenario.launch(MyActivity::class.java)

// Reified inline (KTX)
val s2 = launchActivity<MyActivity>()

// With explicit intent (extras / data uri / action)
val intent = Intent(ApplicationProvider.getApplicationContext(), MyActivity::class.java)
    .putExtra("user_id", 42)
val s3 = ActivityScenario.launch<MyActivity>(intent)

// For activities that call setResult(...) + finish() before reaching RESUMED
val s4 = ActivityScenario.launchActivityForResult(MyActivity::class.java)
```

`launch` blocks the caller until the Activity reaches `RESUMED`. `launchActivityForResult` blocks until the Activity completes — read `scenario.result` afterwards. Source: `androidx.test.core.app.ActivityScenario` (per `tasks/research/R2-scenario.md` lines 48-93).

- [ ] **2. Drive lifecycle with `moveToState(Lifecycle.State.*)`.** Allowed targets: `CREATED`, `STARTED`, `RESUMED`, `DESTROYED`. `DESTROYED` is **terminal** — once reached the scenario cannot move back (lines 100-102 of R2):

```kotlin
import androidx.lifecycle.Lifecycle

scenario.moveToState(Lifecycle.State.STARTED)   // pause behind another activity
scenario.moveToState(Lifecycle.State.RESUMED)   // back to foreground
scenario.moveToState(Lifecycle.State.DESTROYED) // terminal — close() is now a no-op
```

`moveToState` cannot be called from the main thread; calling from a `@UiThreadTest` method or from inside a `Looper` handler deadlocks because it `await(...)`s an instrumentation barrier from the test thread (R2 lines 823-828).

- [ ] **3. Use `recreate()` to emulate a configuration change.** This runs `onSaveInstanceState`, destroys the Activity, creates a fresh instance, and runs `onRestoreInstanceState`. State stored via `rememberSaveable` / `SavedStateHandle` / `Bundle` survives. Plain `remember { ... }` and bare `var` fields do **not** (R2 lines 839-850):

```kotlin
scenario.recreate()
scenario.onActivity { activity ->
    // activity is now a fresh instance
    assertThat(activity.viewModel.count).isEqualTo(2)  // survives if hoisted in the ViewModel
}
```

If a test asserts that state survives `recreate()` and finds it gone, the production code probably uses `remember`, not `rememberSaveable`.

- [ ] **4. Read or mutate the Activity inside `onActivity { activity -> }`.** This block runs on the UI thread, blocks the test thread until done, and is the only legal way to touch fields/views directly:

```kotlin
scenario.onActivity { activity ->
    activity.findViewById<EditText>(R.id.email).setText("test@example.com")
    activity.viewModel.refresh()
}
```

The `activity` reference is **scoped to the lambda**. Never store it past the lambda body — `recreate()` invalidates it, and `close()` destroys the instance.

To capture data out of the scope, use a holder (R2 lines 649-658):

```kotlin
var captured: MyActivity? = null
scenario.onActivity { captured = it }
val current: MyActivity = captured!!
```

- [ ] **5. Read `Activity.RESULT_*` via `scenario.result`.** Only valid for scenarios launched with `launchActivityForResult`, and only after the Activity has called `setResult(...) + finish()`:

```kotlin
val scenario = ActivityScenario.launchActivityForResult(LoginActivity::class.java)
scenario.onActivity { it.confirmLogin() }       // calls setResult(RESULT_OK) + finish()
val result: Instrumentation.ActivityResult = scenario.result
assertThat(result.resultCode).isEqualTo(Activity.RESULT_OK)
```

- [ ] **6. Prefer the `ActivityScenarioRule` JUnit4 wrapper for class-scoped activities.** Single canonical FQN — `androidx.test.ext.junit.rules.ActivityScenarioRule`. The package `androidx.test.rule` does NOT contain `ActivityScenarioRule`; it only ships the deprecated `ActivityTestRule`. This is the most common confusion when migrating (R2 line 490):

```kotlin
import androidx.test.ext.junit.rules.ActivityScenarioRule
import org.junit.Rule

@get:Rule
val activityRule = ActivityScenarioRule(LoginActivity::class.java)

@Test
fun loginButtonEnablesAfterEmail() {
    activityRule.scenario.onActivity { it.email = "x@y.z" }
    // ...
}
```

The rule calls `ActivityScenario.launch` in `before` and `scenario.close()` in `after`.

- [ ] **7. For `@Rule(order = N)` ordering with Hilt, place Hilt outermost.** Lower numbers evaluate outer first (closest to the test method runs last):

```kotlin
@get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
@get:Rule(order = 1) val activityRule = ActivityScenarioRule(LoginActivity::class.java)
```

- [ ] **8. Declare every test Activity in a manifest the test APK will see.** The `ui-test-manifest` artifact only contributes `androidx.activity.ComponentActivity`; custom test activities must be declared by the developer in `src/androidTest/AndroidManifest.xml` or `src/debug/AndroidManifest.xml`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application>
        <activity android:name=".MyTestHostActivity" android:exported="true" />
    </application>
</manifest>
```

Without this, `ActivityScenario.launch` throws `ActivityNotFoundException` regardless of what is in production. See `../../../compose/setup/configuring-test-dependencies/SKILL.md` for the parallel Compose story.

- [ ] **9. Close the scenario.** When using `ActivityScenarioRule`, JUnit calls `close()` automatically. When launching ad hoc, use Kotlin `use { }` (the scenario implements `java.io.Closeable`) per R2 lines 138-150:

```kotlin
ActivityScenario.launch<MyActivity>().use { scenario ->
    scenario.onActivity { /* ... */ }
}
```

`close()` drives the activity to `DESTROYED` regardless of current state and is **idempotent** — safe to call after `moveToState(DESTROYED)` (no-op).

## Patterns

### Pattern: WRONG vs RIGHT — `ActivityTestRule` (deprecated) vs `ActivityScenarioRule`

```kotlin
// WRONG
import androidx.test.rule.ActivityTestRule
@Rule
public ActivityTestRule<MyActivity> rule = new ActivityTestRule<>(MyActivity.class);
// WRONG because: ActivityTestRule is @Deprecated at rules-1.7.0/.../ActivityTestRule.java:85.
// Its launch is racy with main-thread state, and lifecycle control is ad hoc.
```

```kotlin
// RIGHT
import androidx.test.ext.junit.rules.ActivityScenarioRule
@get:Rule
val rule = ActivityScenarioRule(MyActivity::class.java)
```

### Pattern: WRONG vs RIGHT — `ActivityScenarioRule` import path (the FQN trap)

```kotlin
// WRONG
import androidx.test.rule.ActivityScenarioRule        // class does not exist!
// WRONG because: androidx.test:rules ships only the deprecated ActivityTestRule, not
// ActivityScenarioRule. The IDE auto-completes neither — the import is unresolved.
```

```kotlin
// RIGHT
import androidx.test.ext.junit.rules.ActivityScenarioRule
// Lives in androidx.test.ext:junit. This is the SINGLE canonical FQN.
```

### Pattern: WRONG vs RIGHT — calling `launch` / `moveToState` from the main thread

```kotlin
// WRONG
@Test @UiThreadTest
fun launchOnUiThread() {
    val scenario = ActivityScenario.launch(MyActivity::class.java)  // deadlocks
}
// WRONG because: launch / moveToState / onActivity / close all await an instrumentation
// barrier on the test thread. From the UI thread the barrier never fires (R2 lines 823-828).
```

```kotlin
// RIGHT
@Test
fun launchFromTestThread() {
    val scenario = ActivityScenario.launch(MyActivity::class.java)
    scenario.onActivity { activity ->
        // UI-thread work goes here, NOT a recursive moveToState/launch
    }
}
```

### Pattern: WRONG vs RIGHT — capturing the activity reference past the lambda

```kotlin
// WRONG
var leaked: MyActivity? = null
scenario.onActivity { leaked = it }
scenario.recreate()
leaked!!.someField   // stale reference; underlying instance is gone
```

```kotlin
// RIGHT
scenario.onActivity { activity ->
    activity.someField = "new value"
}
// or: re-acquire after recreate()
scenario.recreate()
scenario.onActivity { activity ->
    assertThat(activity.someField).isEqualTo("new value (restored from saved state)")
}
```

### Pattern: WRONG vs RIGHT — relying on "Don't keep activities" for recreate testing

```kotlin
// WRONG
// Toggle Settings.Global.ALWAYS_FINISH_ACTIVITIES to force recreation between activities.
// WRONG because: ActivityScenario does not honor "Don't keep activities" — its barrier
// model expects deterministic transitions. Use scenario.recreate() instead.
```

```kotlin
// RIGHT
scenario.recreate()  // deterministic onSaveInstanceState → destroy → create → onRestoreInstanceState
```

## Mandatory rules

- **MUST** import `ActivityScenarioRule` from `androidx.test.ext.junit.rules`. The package `androidx.test.rule` does NOT contain it.
- **MUST NOT** use `ActivityTestRule` for new tests — it is `@Deprecated`. The lint and codemods replace it.
- **MUST NOT** call `ActivityScenario.launch`, `moveToState`, or `close` from the main thread. They block on an instrumentation barrier that only fires off the UI thread.
- **MUST NOT** store the `activity` reference returned by `onActivity` past the lambda body. `recreate()` and `close()` invalidate it.
- **MUST NOT** rely on `Settings.Global.ALWAYS_FINISH_ACTIVITIES` to force recreation — `ActivityScenario` rejects it. Use `scenario.recreate()`.
- **MUST** declare custom test activities in `src/androidTest/AndroidManifest.xml` or `src/debug/AndroidManifest.xml`. The `ui-test-manifest` artifact only contributes `androidx.activity.ComponentActivity`.
- **MUST** call `moveToState(Lifecycle.State.DESTROYED)` or `close()` (or use `ActivityScenarioRule`, which does it automatically) — leaks confuse subsequent tests.
- **PREFERRED:** use `ActivityScenarioRule` for class-scoped Activity tests; use `ActivityScenario.launch(...).use { }` for tests that need fine-grained launch timing.
- **PREFERRED:** order `@get:Rule(order = 0)` for Hilt and `order = 1` (or higher) for `ActivityScenarioRule` so Hilt installs before the Activity is launched.

## Verification

- [ ] `grep -r "androidx.test.rule.ActivityScenarioRule" src/` returns nothing — only `androidx.test.ext.junit.rules.ActivityScenarioRule` is imported.
- [ ] `grep -r "ActivityTestRule" src/` returns nothing for new code.
- [ ] Every custom test Activity is declared in `src/androidTest/AndroidManifest.xml` or `src/debug/AndroidManifest.xml`.
- [ ] Tests that call `scenario.recreate()` use `rememberSaveable` / `SavedStateHandle` / `onSaveInstanceState`-backed state for anything that should survive.
- [ ] No `@UiThreadTest` method calls `launch` / `moveToState` / `close` directly.
- [ ] `./gradlew :<module>:connectedDebugAndroidTest` runs without `ActivityNotFoundException` or `Cannot run onActivity since Activity is not alive`.
- [ ] Tests that assert on `Activity.RESULT_*` use `launchActivityForResult` (not plain `launch`).

## References

- Android Developers — Test your app's activities (ActivityScenario): https://developer.android.com/guide/components/activities/testing
- Android Developers — AndroidX Test guide: https://developer.android.com/training/testing/instrumented-tests/androidx-test-libraries/rules
- AndroidX Test release notes: https://developer.android.com/jetpack/androidx/releases/test
- `androidx.test.core.app.ActivityScenario` — `launch`, `launchActivityForResult`, `moveToState`, `recreate`, `onActivity`, `state`, `result`, `close`. See `tasks/research/R2-scenario.md` lines 48-160.
- `junit-1.3.0/androidx/test/ext/junit/rules/ActivityScenarioRule.java` — the canonical JUnit4 wrapper.
- `rules-1.7.0/androidx/test/rule/ActivityTestRule.java` lines 85-86 — `@Deprecated`; the deprecation Javadoc explicitly redirects to `ActivityScenario`.
- `tasks/research/R2-scenario.md` — full ActivityScenario + FragmentScenario report. Lines 813-919 enumerate every common pitfall (main-thread launch, DESTROYED terminal, `recreate()` + `remember`, etc.).
- `docs/CORPUS.md` Section H.3 — ActivityScenario API surface and JUnit4 wrapper FQN.
- `tasks/research/R1-androidx-test-core.md` lines 477-535 — the dual-`ActivityScenarioRule` confusion and the canonical FQN.
