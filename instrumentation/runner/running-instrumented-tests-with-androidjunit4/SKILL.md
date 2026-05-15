---
name: running-instrumented-tests-with-androidjunit4
description: Use this skill to stand up an Android instrumentation test source set with the canonical `AndroidJUnit4` runner, the correct `AndroidJUnitRunner` Gradle wiring, and supporting infrastructure (Test Orchestrator, runtime permission grants, size annotations, SDK suppression, hermetic animation defaults). Covers `androidx.test:core:1.7.0` / `:runner:1.7.0` / `:rules:1.7.0` / `androidx.test.ext:junit:1.3.0` coordinates, `androidx.test.platform.app.InstrumentationRegistry`, and the `androidTestUtil` configuration trap for `androidx.test:orchestrator`. Use when the user reports `Test runner not found`, `AndroidJUnit4 deprecated`, `InstrumentationRegistry deprecated`, asks "why is my orchestrator not running", "how do I pass `-e` args to the test", "how do I grant a runtime permission in a test", or "instrumented test won't compile".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-instrumentation-test
  - androidx-test
  - AndroidJUnit4
  - AndroidJUnitRunner
  - test-orchestrator
  - InstrumentationRegistry
  - GrantPermissionRule
  - SdkSuppress
  - SmallTest
  - androidTestUtil
  - animationsDisabled
---

# Running Instrumented Tests with AndroidJUnit4 — Get the Runner Stack Right

Instrumentation tests fail before any assertion runs when the runner stack is misconfigured: a deprecated `AndroidJUnit4` import, the orchestrator on `androidTestImplementation` instead of `androidTestUtil`, the deprecated `androidx.test.InstrumentationRegistry` instead of `androidx.test.platform.app.InstrumentationRegistry`. This skill encodes the exact dependency matrix, the runner Gradle config, and the supporting annotations and rules.

## When to use this skill

- The IDE flags `androidx.test.runner.AndroidJUnit4` as `@Deprecated`, or the test compiles but emits the deprecation warning.
- `am instrument` returns `INSTRUMENTATION_FAILED: <pkg>/androidx.test.runner.AndroidJUnitRunner` or `Test runner not found`.
- Test Orchestrator is configured in Gradle but tests still share a single instrumentation process (orchestrator silently disabled).
- The user passes `-e size small` / `-e package com.foo` via `am instrument` and the test ignores it — `RunnerArgs` is unwired.
- A test needs `ACCESS_FINE_LOCATION` / `RECORD_AUDIO` etc. and the user is hand-rolling permission grants from `adb`.
- The user wants to skip a test below API 26 and is searching for the annotation.

## When NOT to use this skill

- The user is launching an `Activity` or driving lifecycle — see `../../scenarios/launching-activities-with-activityscenario/SKILL.md`.
- The user is interacting with Views — see `../../espresso/writing-espresso-tests/SKILL.md`.
- The user is driving system UI or another app — see `../../uiautomator/cross-app-tests-with-uiautomator/SKILL.md`.
- The user runs JVM (Robolectric / pure JUnit) tests — see `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`.
- The user is choosing what to test in the first place — see `../../../fundamentals/strategies/applying-testing-strategies/SKILL.md`.

## Prerequisites

- Android Gradle Plugin module (`com.android.application` or `com.android.library`).
- A `src/androidTest/` source set on disk (`./gradlew :module:connectedDebugAndroidTest` should resolve).
- An emulator or physical device (or Firebase Test Lab / managed devices) — the runner cannot execute on the JVM.
- JUnit 4 on the test classpath (`junit:junit:4.13.2`); JUnit 5 requires a Vintage engine bridge and is out of scope.

## Workflow

- [ ] **1. Pin the AndroidX Test artifacts on `androidTestImplementation`.** These are the canonical 2026 GA versions per `docs/CORPUS.md` H.1:

```kotlin
dependencies {
    androidTestImplementation("androidx.test:core:1.7.0")
    androidTestImplementation("androidx.test:runner:1.7.0")
    androidTestImplementation("androidx.test:rules:1.7.0")
    androidTestImplementation("androidx.test.ext:junit:1.3.0")
    androidTestImplementation("androidx.test.ext:junit-ktx:1.3.0")
    androidTestImplementation("androidx.test.ext:truth:1.7.0")
}
```

`androidx.test.ext:junit` is the artifact that owns the **non-deprecated** `AndroidJUnit4` runner and the **non-deprecated** `ActivityScenarioRule`. `androidx.test:runner` still ships the legacy `AndroidJUnit4` (deprecated) — never import from there. See `tasks/research/R1-androidx-test-core.md` lines 481-490.

- [ ] **2. Set the `testInstrumentationRunner` in the module's `android.defaultConfig`.** The runner FQCN is `androidx.test.runner.AndroidJUnitRunner` — note this is the runner *class*, distinct from the deprecated `AndroidJUnit4` *test runner annotation*. From `tasks/research/R1-androidx-test-core.md` lines 270-289:

```kotlin
android {
    defaultConfig {
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        // Optional: pass `-e key value` args to every run via Gradle
        testInstrumentationRunnerArguments["clearPackageData"] = "true"
    }
    testOptions {
        animationsDisabled = true                  // hermetic — see Pattern below
    }
}
```

- [ ] **3. Annotate test classes with `@RunWith(AndroidJUnit4::class)` from `androidx.test.ext.junit.runners`.** This is the only correct import:

```kotlin
import androidx.test.ext.junit.runners.AndroidJUnit4
import org.junit.Test
import org.junit.runner.RunWith

@RunWith(AndroidJUnit4::class)
class LoginActivityTest {
    @Test fun loginSucceeds() { /* ... */ }
}
```

The deprecated `androidx.test.runner.AndroidJUnit4` is `@Deprecated` at `runner-1.7.0/androidx/test/runner/AndroidJUnit4.java:43-44`. It still works at runtime but the IDE warns and codemods will replace it.

- [ ] **4. Read instrumentation arguments via `androidx.test.platform.app.InstrumentationRegistry`.** The legacy `androidx.test.InstrumentationRegistry` (no `platform.app`) is `@Deprecated` and `@InlineMe`'d to the canonical class — see `tasks/research/R1-androidx-test-core.md` lines 142-153:

```kotlin
import androidx.test.platform.app.InstrumentationRegistry
import androidx.test.core.app.ApplicationProvider

val instrumentation = InstrumentationRegistry.getInstrumentation()
val args: Bundle = InstrumentationRegistry.getArguments()       // -e key value extras
val targetContext: Context = ApplicationProvider.getApplicationContext()
val testContext: Context = instrumentation.context             // package context of the test APK
```

`InstrumentationRegistry.getArguments()` exposes the `-e` key/value pairs from `am instrument -e key value`, e.g. `-e size small -e numShards 4`. The canonical list of supported keys is in `runner-1.7.0/androidx/test/internal/runner/RunnerArgs.java` lines 54-99 (see `tasks/research/R1-androidx-test-core.md` lines 295-336).

- [ ] **5. Annotate tests with `@SmallTest` / `@MediumTest` / `@LargeTest` from `androidx.test.filters`.** Used together with `am instrument -e size small` to filter by speed envelope. From the package KDocs: `SmallTest` ≤200ms, `MediumTest` ≤1000ms, `LargeTest` >1000ms. Sibling annotations: `@FlakyTest`, `@RequiresDevice`, `@Suppress`.

```kotlin
import androidx.test.filters.SmallTest
import androidx.test.filters.SdkSuppress

@RunWith(AndroidJUnit4::class)
@SmallTest
class LoginViewModelTest { /* ... */ }

@Test
@SdkSuppress(minSdkVersion = 26)
fun usesApi26OnlyApi() { /* ... */ }
```

`@SdkSuppress(minSdkVersion, maxSdkVersion, excludedSdks, codeName)` lives at `runner-1.7.0/androidx/test/filters/SdkSuppress.java:39-58`. Tests outside the API range are skipped (assumption-failure, exit code `-4`), not failed.

- [ ] **6. Add Test Orchestrator on `androidTestUtil` — NOT `androidTestImplementation`.** This is the most common single misconfiguration in the runner stack. Test Orchestrator runs each `@Test` in its own instrumentation process so a crash kills only that one test, with optional `clearPackageData` between tests:

```kotlin
dependencies {
    androidTestUtil("androidx.test:orchestrator:1.6.1")  // installs orchestrator APK on device
    androidTestUtil("androidx.test:services:1.6.0")      // required for clearPackageData
}

android {
    testOptions {
        execution = "ANDROIDX_TEST_ORCHESTRATOR"
    }
    defaultConfig {
        testInstrumentationRunnerArguments["clearPackageData"] = "true"
    }
}
```

`androidTestUtil` is a special AGP configuration that installs helper APKs on the device prior to running tests. Putting orchestrator on `androidTestImplementation` compiles, **silently does nothing**, and the test still runs in a single shared process. See `tasks/research/R1-androidx-test-core.md` lines 683-687.

- [ ] **7. Grant runtime permissions with `GrantPermissionRule.grant(...)` instead of `adb shell pm grant`.** The rule ships in `androidx.test:rules` and grants permissions for the duration of the test class:

```kotlin
import androidx.test.rule.GrantPermissionRule
import org.junit.Rule

@get:Rule
val permissionRule: GrantPermissionRule = GrantPermissionRule.grant(
    android.Manifest.permission.ACCESS_FINE_LOCATION,
    android.Manifest.permission.RECORD_AUDIO,
)
```

The rule is implemented at `rules-1.7.0/androidx/test/rule/GrantPermissionRule.java`. On API 23+ it uses `UiAutomation.grantRuntimePermission`; on older devices the permission must already be granted in the manifest (the rule is a no-op).

- [ ] **8. Disable animations for hermetic runs.** `testOptions { animationsDisabled = true }` toggles `window_animation_scale`, `transition_animation_scale`, and `animator_duration_scale` to `0` for the duration of the test run, then restores the previous values on teardown. Without this, tap-to-focus, ripple, and Activity transitions race with assertions.

- [ ] **9. (Optional) Order multiple `@Rule`s with `@Rule(order = N)`.** Lower order numbers evaluate **outer first**. Hilt requires `@HiltAndroidRule` at `order = 0`:

```kotlin
@get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
@get:Rule(order = 1) val activityRule = ActivityScenarioRule(LoginActivity::class.java)
```

- [ ] **10. Run.** From the host:

```bash
./gradlew :app:connectedDebugAndroidTest
# or via adb directly:
adb shell am instrument -w -r \
  -e size small \
  -e clearPackageData true \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner
```

`-w` (wait) is **required** for the exit code to be meaningful. Without it, `$?` is `0` even on test failure. See `docs/CORPUS.md` I.5.

## Patterns

### Pattern: WRONG vs RIGHT — `AndroidJUnit4` import

```kotlin
// WRONG
import androidx.test.runner.AndroidJUnit4   // @Deprecated since androidx.test:runner 1.x
@RunWith(AndroidJUnit4::class)
class FooTest
// WRONG because: this class is @Deprecated at runner-1.7.0/.../AndroidJUnit4.java:43-44.
// The IDE flags it; codemods replace it; the doc explicitly redirects to ext.junit.
```

```kotlin
// RIGHT
import androidx.test.ext.junit.runners.AndroidJUnit4
@RunWith(AndroidJUnit4::class)
class FooTest
```

### Pattern: WRONG vs RIGHT — `InstrumentationRegistry` import

```kotlin
// WRONG
import androidx.test.InstrumentationRegistry             // @Deprecated, every member @InlineMe'd
val ctx = InstrumentationRegistry.getTargetContext()     // returns target package context, NOT the application
// WRONG because: monitor-1.8.0/androidx/test/InstrumentationRegistry.java:34 is @Deprecated.
// Use androidx.test.platform.app.InstrumentationRegistry + ApplicationProvider.
```

```kotlin
// RIGHT
import androidx.test.platform.app.InstrumentationRegistry
import androidx.test.core.app.ApplicationProvider
val instrumentation = InstrumentationRegistry.getInstrumentation()
val ctx: Context = ApplicationProvider.getApplicationContext()
```

### Pattern: WRONG vs RIGHT — Test Orchestrator configuration (THE common misconfig)

```kotlin
// WRONG
dependencies {
    androidTestImplementation("androidx.test:orchestrator:1.6.1")
}
android.testOptions.execution = "ANDROIDX_TEST_ORCHESTRATOR"
// WRONG because: orchestrator must be installed as a separate APK on the device.
// `androidTestImplementation` puts it on the test APK classpath instead, where it does
// nothing. Tests still run in one shared process. No error, just silent inactivation.
```

```kotlin
// RIGHT
dependencies {
    androidTestUtil("androidx.test:orchestrator:1.6.1")
    androidTestUtil("androidx.test:services:1.6.0")  // required for clearPackageData
}
android.testOptions.execution = "ANDROIDX_TEST_ORCHESTRATOR"
```

### Pattern: filtering tests at run time via `RunnerArgs`

```bash
# Run a single method
adb shell am instrument -w -r \
  -e class com.example.LoginActivityTest#loginSucceeds \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner

# Filter by size annotation
adb shell am instrument -w -r -e size small  com.example.app.test/androidx.test.runner.AndroidJUnitRunner

# Shard across N runners (CI fan-out)
adb shell am instrument -w -r -e numShards 4 -e shardIndex 0 \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner
```

The full list of supported `-e` keys lives in `runner-1.7.0/androidx/test/internal/runner/RunnerArgs.java:54-99` (`class`, `package`, `notClass`, `size`, `annotation`, `notAnnotation`, `numShards`, `shardIndex`, `clearPackageData`, `coverage`, `debug`, `listener`, etc.). To consume them inside the test, read `InstrumentationRegistry.getArguments()`.

## Mandatory rules

- **MUST** use `androidx.test.ext.junit.runners.AndroidJUnit4` for `@RunWith(...)`. The `androidx.test.runner.AndroidJUnit4` form is `@Deprecated`.
- **MUST** use `androidx.test.platform.app.InstrumentationRegistry`. The bare `androidx.test.InstrumentationRegistry` is `@Deprecated`.
- **MUST** put `androidx.test:orchestrator` on `androidTestUtil`, never on `androidTestImplementation`. Putting it on the wrong configuration fails silently.
- **MUST** set `testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"` (the runner *class* — that one is not deprecated, only the test-runner *annotation* is).
- **MUST** pass `-w` to `am instrument` so the exit code reflects test result. Without it, `$?` is meaningless.
- **MUST NOT** call `Thread.sleep` to wait for instrumentation lifecycle (`onCreate`, `onStart`). Use `ActivityScenario.moveToState` (see `../../scenarios/launching-activities-with-activityscenario/SKILL.md`) or an `IdlingResource`.
- **MUST NOT** grant permissions via raw `adb shell pm grant` from inside the test — use `GrantPermissionRule.grant(...)` so the grant scopes to the test class and reverts cleanly.
- **MUST NOT** use `@Rule public ActivityTestRule` — it is `@Deprecated`. Use `@get:Rule val rule = ActivityScenarioRule(...)`.
- **PREFERRED:** annotate every instrumentation test class with `@SmallTest` / `@MediumTest` / `@LargeTest` so CI shards can filter by size.
- **PREFERRED:** keep `testOptions.animationsDisabled = true` even when individual tests don't seem to need it; ripple/transition flake silently corrupts unrelated tests.

## Verification

- [ ] `./gradlew :<module>:connectedDebugAndroidTest` runs without `Test runner not found` or `INSTRUMENTATION_FAILED`.
- [ ] No source file imports `androidx.test.runner.AndroidJUnit4` or `androidx.test.InstrumentationRegistry` (legacy paths).
- [ ] `grep -r "androidTestImplementation.*orchestrator" build.gradle*` returns nothing — orchestrator is on `androidTestUtil`.
- [ ] `adb shell am instrument -w -r -e size small <pkg>.test/androidx.test.runner.AndroidJUnitRunner` returns a non-zero exit code on test failure.
- [ ] If orchestrator is enabled, `adb logcat | grep AndroidTestOrchestrator` shows per-test process spawning.
- [ ] `testOptions.animationsDisabled = true` is set in the module's `android` block.
- [ ] Any runtime permission required by tests has a `@get:Rule val rule = GrantPermissionRule.grant(...)` (not a manual `adb shell pm grant`).

## References

- Android Developers — AndroidJUnitRunner overview: https://developer.android.com/training/testing/instrumented-tests/androidx-test-libraries/runner
- Android Developers — Test Orchestrator: https://developer.android.com/training/testing/instrumented-tests/androidx-test-libraries/runner#use-android
- AndroidX Test release notes: https://developer.android.com/jetpack/androidx/releases/test
- `runner-1.7.0/androidx/test/runner/AndroidJUnit4.java` lines 41-44 — `@Deprecated public final class AndroidJUnit4 extends Runner`.
- `junit-1.3.0/androidx/test/ext/junit/runners/AndroidJUnit4.java` line 49 — current non-deprecated runner; delegates to Robolectric on JVM and `AndroidJUnit4ClassRunner` on device.
- `runner-1.7.0/androidx/test/runner/AndroidJUnitRunner.java` lines 270-372 — the runner class `am instrument` invokes; lifecycle and orchestrator wait logic.
- `runner-1.7.0/androidx/test/internal/runner/RunnerArgs.java` lines 54-99 — canonical list of every `-e` key the runner accepts.
- `monitor-1.8.0/androidx/test/InstrumentationRegistry.java` line 34 — `@Deprecated` legacy registry; every member `@InlineMe`'d to the platform.app form.
- `monitor-1.8.0/androidx/test/platform/app/InstrumentationRegistry.java` — current canonical registry.
- `rules-1.7.0/androidx/test/rule/GrantPermissionRule.java` lines 1-100 — `static GrantPermissionRule grant(String...)`.
- `runner-1.7.0/androidx/test/filters/SdkSuppress.java` lines 39-58 — `minSdkVersion`/`maxSdkVersion`/`excludedSdks`/`codeName`.
- `tasks/research/R1-androidx-test-core.md` — full runner / test-core deep dive (lines 269-740 for runner; 481-490 for the dual-`AndroidJUnit4` confusion).
- `docs/CORPUS.md` Section H — instrumentation library coordinates and deprecation table.
- Sibling skill: `../../managed-devices/running-tests-on-gradle-managed-devices/SKILL.md` — running these tests on emulators Gradle provisions/boots/tears down (reproducible in CI).
- Cross-set: `../../../adb/tests/running-instrumented-tests-via-adb/SKILL.md` — invoking the runner directly via `adb shell am instrument -w -r`.
- Cross-set: `../../../platform/legacy/migrating-from-android-test-classes/SKILL.md` — if the module still uses the legacy `android.test.*` classes / `InstrumentationTestRunner`, migrate it onto this runner stack first.
