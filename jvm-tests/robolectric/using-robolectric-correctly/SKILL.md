---
name: using-robolectric-correctly
description: Use this skill to run Android-aware unit tests on the JVM with Robolectric — the right runner choice (AndroidJUnit4 vs RobolectricTestRunner), @Config sdk/qualifiers/application, the includeAndroidResources requirement, common shadows (ShadowApplication, ShadowLog, ShadowLooper, ShadowPackageManager, ShadowSystemClock), looper draining (shadowOf(Looper.getMainLooper()).idle()), and when NOT to reach for Robolectric (screenshot tests, RenderThread, accessibility services). Also covers the AGP 7.2+ sharedTest reality check and the migration to KMP androidHostTest/androidDeviceTest. If the user mentions Robolectric, AndroidJUnit4 host test, RobolectricTestRunner, @Config, includeAndroidResources, ShadowLooper.idle, shadowOf, sharedTest broken on AGP 7.2, ParameterizedRobolectricTestRunner, "test passes on JVM fails on device", or "Resources NotFoundException" in unit tests, use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - robolectric
  - jvm-android-tests
  - android-junit4
  - shadows
  - shadow-looper
  - config-annotation
  - shared-test
  - host-test
  - include-android-resources
  - parameterized-robolectric
---

# Using Robolectric Correctly — Android On The JVM, Without The Footguns

Robolectric simulates Android on the JVM by swapping in pure-Java reimplementations of system classes ("shadows"). It is fast (10-100x vs an emulator) but **not** an emulator: no real `RenderThread`, no real Binder, no real GPU. This skill locks down the runner choice, the `@Config` matrix, the looper draining ritual, and the AGP 7.2+ sharedTest gotcha. Compose-on-Robolectric specifics live in `../../../compose/synchronization/controlling-the-test-clock/SKILL.md`.

## When to use this skill

- The class under test imports `android.*` (Context, Resources, View, PackageManager, Looper) and the developer wants to test it on the JVM in seconds, not on an emulator in minutes.
- A unit test fails with `RuntimeException: Method ... not mocked` (the bare-Android-jar default) — the developer needs Robolectric's shadows.
- A unit test fails with `Resources$NotFoundException` — `testOptions.unitTests.includeAndroidResources` is missing.
- The developer asks "AndroidJUnit4 vs RobolectricTestRunner — which?".
- A `Handler.postDelayed` / `Looper` queue does not advance under Robolectric's default `LooperMode.PAUSED`.
- The developer asks how to run the **same** test source on JVM and on a device (`sharedTest` / `androidHostTest`+`androidDeviceTest`).
- Build error after AGP 7.2+: "Source directory ... already added to source set", caused by the classic `sharedTest` `srcDir` sharing pattern.

## When NOT to use this skill

- The class under test has zero `android.*` imports. Robolectric pays a 1-3 s class-load tax for nothing — use plain JUnit4. See `../../runner/configuring-junit4-on-android/SKILL.md`.
- The test verifies pixel correctness, ripple animations, screenshot diffs, accessibility services, or `RenderThread` timing. Run on an emulator/device. See `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.
- The test is about coroutines + `runTest` and does not need framework Android. Use `../../coroutines/testing-coroutines-with-runtest/SKILL.md`.
- The test is about Compose UI rendering. The host-test environment is set up by `../../../compose/setup/configuring-test-dependencies/SKILL.md` and the clock semantics live in `../../../compose/synchronization/controlling-the-test-clock/SKILL.md`.

## Prerequisites

```kotlin
testImplementation("junit:junit:4.13.2")
testImplementation("org.robolectric:robolectric:4.x")
testImplementation("androidx.test:core:1.7.0")              // ApplicationProvider works on both runtimes
testImplementation("androidx.test.ext:junit:1.3.0")         // androidx.test.ext.junit.runners.AndroidJUnit4
```

(`docs/CORPUS.md` §G.1; R7.)

```gradle
android {
    testOptions {
        unitTests {
            includeAndroidResources = true                  // non-negotiable
        }
    }
}
```

`includeAndroidResources` is required if any test (or any class loaded by any test) touches `R.*`, `Resources`, layouts, themes, or anything from `res/`. Without it, resource lookups throw at test startup (R7).

Robolectric 4.x supports SDKs 21 through the latest stable. The androidx Compose host tests pin a floor of 23 — quote it as a single source of truth:

```kotlin
// androidx/compose/ui/ui-test/src/androidHostTest/.../Constants.kt
internal const val RobolectricMinSdk = 23
```

(R7.)

## Runner choice — AndroidJUnit4 vs RobolectricTestRunner

Two runners work; they read `@Config` identically.

| Runner | When |
|---|---|
| `androidx.test.ext.junit.runners.AndroidJUnit4` | **Default.** A router: delegates to Robolectric on JVM and to the real `AndroidJUnit4ClassRunner` on a device. Linchpin of the sharedTest pattern. |
| `org.robolectric.RobolectricTestRunner` | Only when the test must never run on a device — e.g. uses Robolectric-specific shadow APIs like `shadowOf(Looper.getMainLooper()).idle()` directly, or needs a Robolectric subclass like `ParameterizedRobolectricTestRunner`. |

(R7; `docs/CORPUS.md` §G.6.)

```kotlin
import androidx.test.ext.junit.runners.AndroidJUnit4
@RunWith(AndroidJUnit4::class)
@Config(minSdk = 23)
class FooTest { /* runs under Robolectric on JVM, real runner on device */ }

import org.robolectric.RobolectricTestRunner
@RunWith(RobolectricTestRunner::class)
@Config(sdk = [33])
class LooperBoundTest { /* uses ShadowLooper APIs directly — JVM only */ }
```

CRITICAL: `androidx.test.runner.AndroidJUnit4` (the **runner** package, not `ext.junit.runners`) is `@Deprecated` (`docs/CORPUS.md` §G.2). Always import from `androidx.test.ext.junit.runners`.

## @Config — sdk, qualifiers, application

`@Config` configures the simulated Android environment per class or per method. Method-level overrides class-level; subclasses inherit and may override.

```kotlin
// Single SDK
@Config(sdk = [Build.VERSION_CODES.UPSIDE_DOWN_CAKE])
class FooTest

// SDK matrix — runs once per listed SDK
@Config(sdk = [21, 28, 33])
class BarTest

// Range — runs once at the project's targetSdk if it is >= minSdk
@Config(minSdk = 23)
class BazTest

// Per-method override
class QuxTest {
    @Test @Config(sdk = [Build.VERSION_CODES.TIRAMISU])
    fun onTiramisuOnly() { /* ... */ }

    @Test @Config(qualifiers = "fr-rFR-w820dp")           // locale + screen
    fun frenchLayout() { /* ... */ }

    @Test @Config(qualifiers = "+night")                   // additive: keep, force night
    fun darkColors() { /* ... */ }
}

// Override the Application instance
@Config(application = MyTestApplication::class)
class WithFakeApp
```

(R7.)

Constraint: `sdk` and `minSdk`/`maxSdk` cannot coexist on the same annotation. `minSdk` and `maxSdk` may be combined.

CI-time hot take: prefer `@Config(sdk = [<one SDK>])` on the commit gate; reserve multi-SDK matrices for the merge gate. Each SDK in the matrix re-runs the whole class.

## Shadows

Shadows are JVM-friendly fakes for Android system classes. Robolectric ships hundreds; you can write your own with `@Implements`/`@Implementation`. Common ones:

| Shadow | Purpose |
|---|---|
| `ShadowApplication` | Inspect started services / broadcasts / granted permissions. |
| `ShadowLog` | Capture `android.util.Log` output (`ShadowLog.stream = System.out`). |
| `ShadowSystemClock` | Advance / freeze `SystemClock.elapsedRealtime()` / `uptimeMillis()`. |
| `ShadowLooper` | Drive message queues; `idleMainLooper()`, `runUiThreadTasksIncludingDelayedTasks()`. |
| `ShadowPackageManager` | Add/remove packages, set system features, queryable intents. |
| `ShadowAlarmManager` | Inspect scheduled alarms without firing them. |
| `ShadowNotificationManager` | Inspect posted notifications and channels. |
| `ShadowInputManager` | Add/remove `InputDevice`s for input tests. |
| `ShadowContentResolver` | Register fake providers, observe inserts/queries. |

`Shadows.shadowOf(realObject)` returns the shadow instance. Static-import and rely on overload resolution:

```kotlin
import androidx.test.core.app.ApplicationProvider
import org.robolectric.Shadows.shadowOf
import org.robolectric.shadows.ShadowPackageManager

val app: Application = ApplicationProvider.getApplicationContext()
val shadowPm: ShadowPackageManager = shadowOf(app.packageManager)
shadowPm.setSystemFeature(PackageManager.FEATURE_CAMERA_ANY, true)
```

## Looper draining — the most-missed step

Robolectric's main looper is **paused** in the default `LooperMode.PAUSED` runtime mode. Handler messages posted by the SUT do NOT run automatically. Drain explicitly:

```kotlin
import android.os.Looper
import org.robolectric.Shadows.shadowOf
import org.robolectric.shadows.ShadowLooper
import java.time.Duration

// Drain everything currently queued on the main looper.
shadowOf(Looper.getMainLooper()).idle()

// Run delayed tasks too (postDelayed et al).
ShadowLooper.runUiThreadTasksIncludingDelayedTasks()

// Advance virtual time by N ms; only handlers due before that time fire.
shadowOf(Looper.getMainLooper()).idleFor(Duration.ofMillis(500))
```

(R7.)

This complements (and is independent from) Compose's `MainTestClock.advanceTimeBy(...)`. The Compose clock and the platform looper are separate. For Compose-on-Robolectric, `runComposeUiTest { … waitForIdle() }` drives the looper for you via `RobolectricIdlingStrategy` — outside Compose, drain manually.

## sharedTest / androidHostTest / androidDeviceTest

The classic sharedTest pattern (per the 2021 Robolectric blog post) puts one source directory under both `test` and `androidTest`:

```gradle
// LEGACY — broken on AGP 7.2+
android {
    sourceSets {
        String dir = 'src/sharedTest/'
        test       { java.srcDirs += dir + 'java' }
        androidTest { java.srcDirs += dir + 'java' }
    }
}
```

CRITICAL: starting with **AGP 7.2+, you cannot put the same source directory into multiple source sets** (Robolectric GitHub issue #7432; R7). The build fails with "Source directory ... already added to source set".

Modern alternatives (PREFERRED order):

1. **AndroidX-style KMP source sets** — `commonTest` -> `{androidHostTest, androidDeviceTest}` via the `androidXMultiplatform` plugin. Compose internally migrated to this:
   ```gradle
   sourceSets {
       androidCommonTest  { dependsOn(commonTest) }
       androidDeviceTest  { dependsOn(androidCommonTest) }   // instrumented APK
       androidHostTest    { dependsOn(androidCommonTest) }   // Robolectric on JVM
   }
   ```
2. **Separate Gradle module** that both `test` and `androidTest` depend on as `testImplementation` / `androidTestImplementation`.
3. **Symlink or copy** sources at configuration time (least preferred — fragile).

Both `androidHostTest` and `androidDeviceTest` can use `@RunWith(AndroidJUnit4::class)` unchanged because `AndroidJUnit4` is a router. Source-set wiring is `../../../fundamentals/strategies/organizing-test-source-sets/SKILL.md`.

## When NOT to reach for Robolectric

| Scenario | Where it belongs |
|---|---|
| Pure Kotlin, no `android.*` imports | Plain JUnit4 in `src/test/` (no Robolectric). |
| Pixel-correct rendering, screenshot tests | Emulator/device — `RenderThread` is real there, not on Robolectric. |
| Accessibility service behavior | Emulator/device. Compose's `enableAccessibilityChecks` installs the validator on Robolectric AND logs a warning (`Build.FINGERPRINT.lowercase() == "robolectric"`); Robolectric does not faithfully drive the accessibility services so any pass is inconclusive (b/332778271). |
| `Choreographer` fidelity, animation timing close to production | Emulator/device. |
| GPU/Skia behavior | Emulator/device. `@GraphicsMode(GraphicsMode.Mode.NATIVE)` opt-in is a stub of the real pipeline. |

## Patterns

### Pattern: WRONG — RobolectricTestRunner for a test that should also run on device

```kotlin
// WRONG
@RunWith(RobolectricTestRunner::class)
@Config(minSdk = 23)
class LoginViewModelTest {
    @Test fun loginFlow() { /* ... */ }
}
// WRONG because: this class CAN run unchanged on a device too — there are no
// Robolectric-only APIs in the body. Pinning to RobolectricTestRunner locks the
// test to the JVM and forfeits the sharedTest pattern. The same class moved to
// src/androidTest/ (or androidDeviceTest) would not run.
```

```kotlin
// RIGHT
@RunWith(AndroidJUnit4::class)
@Config(minSdk = RobolectricMinSdk)                      // pin via single source of truth
class LoginViewModelTest {
    @Test fun loginFlow() { /* ... */ }
}
```

### Pattern: WRONG — assert before draining the looper

```kotlin
// WRONG
@RunWith(AndroidJUnit4::class)
class HandlerTest {
    @Test fun postedRuns() {
        var fired = false
        Handler(Looper.getMainLooper()).post { fired = true }
        assertTrue(fired)                                 // FAIL: runnable is queued, not run
    }
}
// WRONG because: Robolectric's default LooperMode.PAUSED does not auto-run posted
// runnables. The post is queued; assertTrue runs before the queue is drained.
```

```kotlin
// RIGHT
@RunWith(AndroidJUnit4::class)
class HandlerTest {
    @Test fun postedRuns() {
        var fired = false
        Handler(Looper.getMainLooper()).post { fired = true }
        shadowOf(Looper.getMainLooper()).idle()           // drain queue
        assertTrue(fired)
    }
}
```

### Pattern: WRONG — missing includeAndroidResources

```gradle
// WRONG
android {
    testOptions {
        unitTests { /* includeAndroidResources missing */ }
    }
}
```

Symptom: `android.content.res.Resources$NotFoundException: String resource ID #0x7f0c0001` at test startup. The fix:

```gradle
// RIGHT
android {
    testOptions {
        unitTests {
            includeAndroidResources = true                // Groovy
            // isIncludeAndroidResources = true           // Kotlin DSL
        }
    }
}
```

### Pattern: WRONG — sharedTest srcDir sharing on AGP 7.2+

```gradle
// WRONG
android {
    sourceSets {
        test       { java.srcDirs += 'src/sharedTest/java' }
        androidTest { java.srcDirs += 'src/sharedTest/java' }
    }
}
// WRONG because: AGP 7.2+ rejects sharing a single source directory across
// source sets (Robolectric issue #7432). Build fails with "Source directory
// already added to source set".
```

```gradle
// RIGHT — AndroidX-style KMP source sets
sourceSets {
    androidCommonTest  { dependsOn(commonTest) }
    androidDeviceTest  { dependsOn(androidCommonTest) }
    androidHostTest    { dependsOn(androidCommonTest) }
}
```

### Pattern: ParameterizedRobolectricTestRunner — when matrix is the point

```kotlin
// androidx/compose/ui/ui-test/.../ViewVisibilityRobolectricTest.kt (paraphrased)
@RunWith(ParameterizedRobolectricTestRunner::class)
@Config(minSdk = RobolectricMinSdk)
class ViewVisibilityRobolectricTest(private val visibility: Int) {
    companion object {
        @JvmStatic
        @Parameters(name = "visibility={0}")
        fun params() = listOf(View.VISIBLE, View.INVISIBLE, View.GONE)
    }
    @Test fun behaves() { /* uses `visibility` */ }
}
```

This is the one place to pin `RobolectricTestRunner` (its parameterized subclass) — there is no `AndroidJUnit4` parameterized variant.

### Pattern: production code branching on Robolectric — read with caution

```kotlin
// androidx/compose/ui/ui-test/.../RobolectricIdlingStrategy.android.kt
internal val HasRobolectricFingerprint
    get() = Build.FINGERPRINT.lowercase() == "robolectric"
```

This idiom is widespread inside androidx (Compose accessibility, idling, graphics layers). Library code under test SHOULD avoid copying it unless the branch is genuinely test-only — misdetection silently changes behavior.

## Mandatory rules

- **MUST** import `AndroidJUnit4` from `androidx.test.ext.junit.runners`. **MUST NOT** import from `androidx.test.runner` — that one is `@Deprecated` (`docs/CORPUS.md` §G.2).
- **MUST** prefer `@RunWith(AndroidJUnit4::class)` over `@RunWith(RobolectricTestRunner::class)`. The latter only when the test uses Robolectric-specific APIs (e.g. `shadowOf(Looper.getMainLooper()).idle()`) directly, or needs `ParameterizedRobolectricTestRunner`.
- **MUST** set `testOptions.unitTests.includeAndroidResources = true` for any project whose unit tests touch `R.*` / `Resources` / layouts / themes.
- **MUST** drain the looper with `shadowOf(Looper.getMainLooper()).idle()` (or `ShadowLooper.runUiThreadTasksIncludingDelayedTasks()`) before asserting anything that depends on a `Handler.post*` callback. Default `LooperMode.PAUSED` does NOT auto-run.
- **MUST** pin a class-level `@Config(minSdk = …)` (or `sdk = […]`) — do not depend on the project's default SDK varying in CI.
- **MUST NOT** use the legacy `srcDirs` sharing pattern `test.java.srcDirs += 'src/sharedTest/java'` on AGP 7.2+. Migrate to `androidHostTest` + `androidDeviceTest` (KMP source sets) or a separate Gradle module.
- **MUST NOT** add Robolectric to a test class that has zero `android.*` imports. The class-load tax is 1-3 s for nothing.
- **MUST NOT** use Robolectric for screenshot tests, ripple animations, accessibility services, or `RenderThread` timing — use an emulator/device.
- **PREFERRED:** quote `RobolectricMinSdk = 23` from `androidx/compose/ui/ui-test/src/androidHostTest/.../Constants.kt` as the single source of truth for the floor SDK in `androidHostTest` modules.
- **PREFERRED:** group related tests into the same class to amortize the per-class class-load tax. Avoid sprinkling `@RunWith(RobolectricTestRunner::class)` across many small classes.
- **PREFERRED:** single-SDK `@Config(sdk = [33])` on the commit gate; multi-SDK matrices on the merge gate.

## Verification

- [ ] No import of `androidx.test.runner.AndroidJUnit4` (use `androidx.test.ext.junit.runners.AndroidJUnit4`). `grep -rn 'androidx.test.runner.AndroidJUnit4' src/test src/androidTest` returns empty.
- [ ] `testOptions.unitTests.includeAndroidResources = true` is set in the module's `build.gradle(.kts)`.
- [ ] No `@RunWith(RobolectricTestRunner::class)` exists on a test class that does not actually use Robolectric-specific APIs.
- [ ] Every test that posts to a `Handler` / triggers `Looper` work calls `shadowOf(Looper.getMainLooper()).idle()` (or `runUiThreadTasksIncludingDelayedTasks`) before asserting.
- [ ] Every Robolectric test class declares `@Config(minSdk = …)` or `@Config(sdk = […])`.
- [ ] Source-set wiring uses `androidHostTest` + `androidDeviceTest` (or a shared module), not legacy `srcDirs += sharedTest`.
- [ ] No screenshot/ripple/accessibility-service test runs under Robolectric — those live in `src/androidTest/` or `androidDeviceTest`.
- [ ] `./gradlew :module:test` passes; `./gradlew :module:connectedDebugAndroidTest` passes for any test in `androidCommonTest`.

## References

- Robolectric — getting started: http://robolectric.org/getting-started/
- Robolectric — configuring: http://robolectric.org/configuring/
- Robolectric — sharedTest blog post: http://robolectric.org/blog/2021/10/06/sharedTest/
- Android Developers — Build local unit tests: https://developer.android.com/training/testing/local-tests
- Android Developers — Robolectric: https://developer.android.com/training/testing/local-tests/robolectric
- Robolectric GitHub issue #7432 — AGP 7.2 sharedTest breakage: https://github.com/robolectric/robolectric/issues/7432
- Research: `tasks/research/R7-robolectric.md`.
- `docs/CORPUS.md` §G.1 / §G.6 — Gradle coordinates and runner choice.
- `androidx/compose/ui/ui-test/src/androidHostTest/.../Constants.kt` — `RobolectricMinSdk = 23` floor.
- `androidx/compose/ui/ui-test/.../RobolectricComposeTest.kt` — canonical `@RunWith(AndroidJUnit4::class) @Config(minSdk = RobolectricMinSdk)` cross-runtime test.
- `androidx/compose/ui/ui-test/.../PrefetchNotHangingMainThreadTest.kt` — `@RunWith(RobolectricTestRunner::class)` with explicit `shadowOf(Looper.getMainLooper()).idle()`.
- `androidx/compose/ui/ui-test/.../ViewVisibilityRobolectricTest.kt` — `ParameterizedRobolectricTestRunner` example.
- `androidx/compose/ui/ui-test/.../RobolectricIdlingStrategy.android.kt` — `Build.FINGERPRINT.lowercase() == "robolectric"` detection idiom.
- `androidx/compose/ui/ui/.../MediaQueryIntegrationTest.kt` — `ShadowPackageManager` / `ShadowInputManager` in an `@RunWith(AndroidJUnit4::class)` test.
- Sibling: `../../coroutines/testing-coroutines-with-runtest/SKILL.md` — `runTest` + `MainDispatcherRule` (independent of Robolectric).
- Sibling: `../../coroutines/testing-flows-with-turbine/SKILL.md` — Flow assertions; Turbine works under Robolectric unchanged.
- Sibling: `../../runner/configuring-junit4-on-android/SKILL.md` — JUnit4 plumbing surrounding Robolectric.
- Sibling: `../../mocking/mocking-with-mockito/SKILL.md` — mocking Android types under Robolectric.
- Sibling: `../../mocking/mocking-with-mockk/SKILL.md` — `mockkStatic(Build::class)` for fingerprint branches.
- Cross-set: `../../../fundamentals/strategies/applying-testing-strategies/SKILL.md` — Robolectric is "medium" tests in Google's three-size framing.
- Cross-set: `../../../fundamentals/strategies/organizing-test-source-sets/SKILL.md` — `src/test/` vs `src/androidTest/` vs `androidHostTest`/`androidDeviceTest`.
- Cross-set: `../../../compose/synchronization/controlling-the-test-clock/SKILL.md` — Compose's `MainTestClock` is separate from `ShadowLooper`.
- Cross-set: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md` — when the same test source needs to run on a device.
