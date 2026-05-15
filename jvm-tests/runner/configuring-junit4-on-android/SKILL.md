---
name: configuring-junit4-on-android
description: Use this skill to stand up a JUnit4-based JVM unit-test suite on an Android module. Covers the canonical Gradle dependency matrix (`junit:junit:4.13.2`, `androidx.test:core:1.7.0`, `androidx.test.ext:junit:1.3.0`, `androidx.test.ext:truth:1.7.0`), the `@RunWith(AndroidJUnit4::class)` runner, `InstrumentationRegistry.getInstrumentation()`, `ApplicationProvider.getApplicationContext()`, the `@SmallTest` / `@MediumTest` / `@LargeTest` size annotations wired into `am instrument -e size`, the androidx-test Truth subjects, and the canonical `MainDispatcherRule` setup pattern. Use when the user mentions `AndroidJUnit4 unresolved`, `which AndroidJUnit4`, `InstrumentationRegistry deprecated`, `getApplicationContext in unit test`, `Truth IntentSubject`, `@SmallTest`, `MainDispatcherRule`, or asks how to wire JUnit4 into an Android module's `src/test/`.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - junit4
  - jvm-unit-tests
  - androidx-test
  - AndroidJUnit4
  - InstrumentationRegistry
  - ApplicationProvider
  - SmallTest
  - MainDispatcherRule
  - test-size-annotations
---

# Configuring JUnit4 on Android — The Right Runner, Registry, and Matrix

Almost every "test suite is misconfigured" report boils down to one of three root causes: the wrong `AndroidJUnit4` is imported, the wrong `InstrumentationRegistry` is imported, or the size-annotation / dispatcher plumbing is missing. This skill encodes the exact Gradle matrix from `androidx.test:core` 1.7.0 onward, the canonical FQCNs, and the minimal `MainDispatcherRule` pattern that androidx itself ships in `testutils-ktx`.

## When to use this skill

- The user reports `Cannot resolve symbol AndroidJUnit4`, `AndroidJUnit4 is deprecated`, or asks "which `AndroidJUnit4` should I import".
- The user reports `IllegalStateException: No instrumentation registered` from `InstrumentationRegistry.getInstrumentation()` in a unit test, or asks why `InstrumentationRegistry` is crossed out.
- The user wants Truth subjects (`IntentSubject`, `BundleSubject`, `LocationSubject`, `ParcelableSubject`) but does not know which artifact ships them.
- The user wants `am instrument -e size small` to filter tests and asks how to wire `@SmallTest` / `@MediumTest` / `@LargeTest`.
- The user is writing a `ViewModel` / coroutine test and needs the canonical `MainDispatcherRule` template.

## When NOT to use this skill

- The user is wiring Mockito mocks — use `../../mocking/mocking-with-mockito/SKILL.md`.
- The user is wiring MockK mocks — use `../../mocking/mocking-with-mockk/SKILL.md`.
- The user is configuring `runTest { … }` or `Turbine` — use `../../coroutines/testing-coroutines-with-runtest/SKILL.md` or `../../coroutines/testing-flows-with-turbine/SKILL.md`.
- The user is configuring Robolectric specifically (SDK matrix, resources, `@Config`) — use `../../robolectric/using-robolectric-correctly/SKILL.md`.
- The user is running tests on a device with `am instrument` — use `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.
- The user is choosing between fakes and mocks — use `../../../fundamentals/doubles/picking-test-doubles/SKILL.md`.

## Prerequisites

- Android Gradle Plugin applied (`com.android.application` or `com.android.library`).
- A `src/test/` source set on disk (Gradle creates it lazily — `mkdir -p src/test/kotlin` if missing).
- Kotlin module (these notes assume Kotlin; the same FQCNs work from Java).
- For coroutine-related tests, `org.jetbrains.kotlinx:kotlinx-coroutines-test:1.10.x` already on the classpath.

## Workflow

- [ ] **1. Pin the canonical Gradle dependency matrix.** These are the released stable versions backing this skill (verified against `~/.gradle/caches/modules-2/files-2.1/androidx.test/...`). Pin them directly, do not `latest.release`:

```kotlin
dependencies {
    // Core JUnit4
    testImplementation("junit:junit:4.13.2")

    // androidx.test foundations (work in src/test/ via Robolectric and in src/androidTest/)
    testImplementation("androidx.test:core:1.7.0")
    testImplementation("androidx.test:core-ktx:1.6.1")
    testImplementation("androidx.test:runner:1.7.0")
    testImplementation("androidx.test:rules:1.7.0")
    testImplementation("androidx.test.ext:junit:1.3.0")
    testImplementation("androidx.test.ext:junit-ktx:1.3.0")
    testImplementation("androidx.test.ext:truth:1.7.0")
    testImplementation("com.google.truth:truth:1.4.4")

    // Coroutines test (for MainDispatcherRule / runTest)
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.10.1")
}
```

For instrumented tests in `src/androidTest/`, mirror the same artifacts on `androidTestImplementation`. The artifacts are dual-classpath compatible.

- [ ] **2. Annotate test classes with the correct `@RunWith`.** There are TWO classes named `AndroidJUnit4` on the classpath. The deprecated one is `androidx.test.runner.AndroidJUnit4` (file `runner-1.7.0/androidx/test/runner/AndroidJUnit4.java` L41-44 — `@Deprecated`). The canonical one is `androidx.test.ext.junit.runners.AndroidJUnit4` from the `androidx.test.ext:junit:1.3.0` artifact (file `junit-1.3.0/androidx/test/ext/junit/runners/AndroidJUnit4.java` L49). It delegates to Robolectric on the JVM (when `org.robolectric.RobolectricTestRunner` is on the classpath) and to `androidx.test.internal.runner.junit4.AndroidJUnit4ClassRunner` on a device.

```kotlin
import androidx.test.ext.junit.runners.AndroidJUnit4   // canonical
import org.junit.runner.RunWith

@RunWith(AndroidJUnit4::class)
class MyTest { /* ... */ }
```

- [ ] **3. Use the canonical `InstrumentationRegistry`.** `androidx.test.InstrumentationRegistry` (no `.platform.app.`) is `@Deprecated` since `:monitor` 1.x — every member is `@InlineMe`'d to its replacement. The canonical class is `androidx.test.platform.app.InstrumentationRegistry` (file `monitor-1.8.0/androidx/test/platform/app/InstrumentationRegistry.java` L30-85).

```kotlin
import androidx.test.platform.app.InstrumentationRegistry

val instrumentation = InstrumentationRegistry.getInstrumentation()
val args: Bundle = InstrumentationRegistry.getArguments()  // copy of -e key/value pairs
```

`getInstrumentation()` throws `IllegalStateException("No instrumentation registered! Must run under a registering instrumentation.")` if called outside an instrumentation host or before Robolectric initialises its shadow. Robolectric registers an `Instrumentation` instance into the registry before tests run, so this works in `src/test/` when `@RunWith(AndroidJUnit4::class)` is in effect.

- [ ] **4. Pull the application Context via `ApplicationProvider`, not via the registry.** `ApplicationProvider.getApplicationContext()` is the single entry point for the Application context (file `core-1.7.0/androidx/test/core/app/ApplicationProvider.java` L29-43). Internally it returns `getInstrumentation().getTargetContext().getApplicationContext()` — same Context, more readable callsite, and the unchecked generic lets callers cast to their `Application` subclass.

```kotlin
import androidx.test.core.app.ApplicationProvider

val ctx: Context = ApplicationProvider.getApplicationContext()
val app: MyApp = ApplicationProvider.getApplicationContext()   // unchecked cast
```

- [ ] **5. Tag each test with a size annotation when the suite is large enough to need filtering.** Size is one of three: `@SmallTest` (`<200ms`, no Android stubs), `@MediumTest` (`<1000ms`, Android framework via Robolectric), `@LargeTest` (`>1000ms`, instrumented). Annotations live in `androidx.test.filters.*` (file `runner-1.7.0/androidx/test/filters/SmallTest.java` L44-46). The runner's `RunnerArgs.SIZE` (`runner-1.7.0/androidx/test/internal/runner/RunnerArgs.java` L58) wires `-e size <value>` directly to these annotations.

```kotlin
import androidx.test.filters.SmallTest
import androidx.test.filters.MediumTest

@SmallTest
class FastValidatorTest { /* ... */ }

@MediumTest
class RoomDaoTest { /* ... */ }
```

The corresponding `am instrument` invocation is `adb shell am instrument -w -r -e size small <pkg>/androidx.test.runner.AndroidJUnitRunner`. See `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md` for the on-device counterpart.

- [ ] **6. Reach for Truth subjects from `androidx.test.ext:truth` for Android-domain assertions.** Truth shipped with `androidx.test.ext:truth:1.7.0` adds Android-aware `Subject` types. Each exposes a `static assertThat(T actual)` shortcut and a `static Subject.Factory<T, S> name()` factory:

| Subject | FQCN package | Asserts on |
| --- | --- | --- |
| `IntentSubject` | `androidx.test.ext.truth.content` | `android.content.Intent` (action, data, component, extras, categories, flags) |
| `BundleSubject` / `BaseBundleSubject` / `PersistableBundleSubject` | `androidx.test.ext.truth.os` | `Bundle` / `BaseBundle` / `PersistableBundle` keys, types, values |
| `ParcelableSubject` | `androidx.test.ext.truth.os` | `Parcelable` (round-trips through a `Parcel` to verify the writer/reader) |
| `LocationSubject` | `androidx.test.ext.truth.location` | `android.location.Location` |
| `NotificationSubject` / `NotificationActionSubject` / `PendingIntentSubject` | `androidx.test.ext.truth.app` | `Notification` family |
| `MotionEventSubject` / `PointerCoordsSubject` / `PointerPropertiesSubject` | `androidx.test.ext.truth.view` | `MotionEvent` family |
| `SparseBooleanArraySubject` | `androidx.test.ext.truth.util` | `SparseBooleanArray` |

Idiomatic import-and-call:

```kotlin
import androidx.test.ext.truth.content.IntentSubject.assertThat
import androidx.test.ext.truth.os.BundleSubject

@Test fun intent_routes_to_settings() {
    val intent = Intent(Intent.ACTION_VIEW, Uri.parse("settings://wifi"))
    assertThat(intent).hasAction(Intent.ACTION_VIEW)
    assertThat(intent).hasData(Uri.parse("settings://wifi"))
}
```

- [ ] **7. Install a `MainDispatcherRule` for any test that touches `Dispatchers.Main`.** Compose, `viewModelScope`, and `LiveData` all default to `Dispatchers.Main`, which is unavailable on the JVM unless swapped. The canonical androidx pattern lives at `testutils/testutils-ktx/src/jvmMain/kotlin/androidx/testutils/MainDispatcherRule.jvm.kt` — use it verbatim:

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.StandardTestDispatcher
import kotlinx.coroutines.test.TestDispatcher
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.setMain
import org.junit.rules.TestWatcher
import org.junit.runner.Description

@OptIn(ExperimentalCoroutinesApi::class)
class MainDispatcherRule(
    val testDispatcher: TestDispatcher = StandardTestDispatcher(),
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }

    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

Apply on a test class:

```kotlin
@RunWith(AndroidJUnit4::class)
class MyViewModelTest {
    @get:Rule val mainDispatcherRule = MainDispatcherRule()

    @Test fun loadsUsers() = runTest {
        val vm = MyViewModel(repo = FakeUserRepository())
        vm.load()
        advanceUntilIdle()
        assertThat(vm.users.value).isNotEmpty()
    }
}
```

`StandardTestDispatcher` is preferred — it queues continuations and only advances on `runCurrent()`/`advanceUntilIdle()`, matching `kotlinx.coroutines.test.runTest` semantics. `UnconfinedTestDispatcher` dispatches eagerly and is appropriate only when ordering is irrelevant.

- [ ] **8. Verify the test class actually runs.** Execute the suite once: `./gradlew :<module>:testDebugUnitTest --tests com.example.MyTest`. The Gradle output shows the runner choice — `org.junit.runner.RunWith: AndroidJUnit4` confirms the canonical runner is bound; `Test process exited with status code` near the top confirms Robolectric is wiring up.

## Patterns

### Pattern: WRONG vs RIGHT — choosing the `AndroidJUnit4` runner

```kotlin
// WRONG
import org.junit.runners.JUnit4
import org.junit.runner.RunWith

@RunWith(JUnit4::class)
class MyTest { /* uses InstrumentationRegistry, ApplicationProvider, ... */ }
// WRONG because: stock JUnit4 does not register the Android `Instrumentation` instance
// into InstrumentationRegistry. ApplicationProvider.getApplicationContext() throws
// IllegalStateException("No instrumentation registered!"). Robolectric is also never
// activated even when on the classpath, so all framework calls hit unmocked stubs.
```

```kotlin
// RIGHT
import androidx.test.ext.junit.runners.AndroidJUnit4   // ext.junit.runners, NOT runner.AndroidJUnit4
import org.junit.runner.RunWith

@RunWith(AndroidJUnit4::class)
class MyTest { /* ... */ }
```

The deprecated alias `androidx.test.runner.AndroidJUnit4` still compiles but emits an unsuppressed deprecation warning. New code MUST import from `androidx.test.ext.junit.runners`.

### Pattern: WRONG vs RIGHT — `InstrumentationRegistry` import

```kotlin
// WRONG
import androidx.test.InstrumentationRegistry        // @Deprecated since :monitor

val ctx = InstrumentationRegistry.getContext()      // also deprecated — not the Application context
// WRONG because: every member of androidx.test.InstrumentationRegistry is @Deprecated
// and @InlineMe'd to its replacement. getContext() returns the instrumentation package
// context, NOT the Application context — common source of "resource not found" failures.
```

```kotlin
// RIGHT
import androidx.test.platform.app.InstrumentationRegistry
import androidx.test.core.app.ApplicationProvider

val ctx: Context = ApplicationProvider.getApplicationContext()    // application context
val args: Bundle = InstrumentationRegistry.getArguments()          // -e key/value pairs
val instr = InstrumentationRegistry.getInstrumentation()
```

### Pattern: WRONG vs RIGHT — accessing Compose `Dispatchers.Main` from a unit test

```kotlin
// WRONG
class MyViewModelTest {
    @Test fun loadsUsers() = runBlocking {
        val vm = MyViewModel(repo = FakeUserRepository())
        vm.load()
        // Throws IllegalStateException: "Module with the Main dispatcher is missing"
    }
}
// WRONG because: kotlinx-coroutines-android (the Main dispatcher impl for Android) is
// NOT on the unit-test classpath. Without Dispatchers.setMain(...), any viewModelScope or
// LaunchedEffect that touches the Main dispatcher fails immediately.
```

```kotlin
// RIGHT
class MyViewModelTest {
    @get:Rule val mainDispatcherRule = MainDispatcherRule()

    @Test fun loadsUsers() = runTest {
        val vm = MyViewModel(repo = FakeUserRepository())
        vm.load()
        advanceUntilIdle()
        assertThat(vm.users.value).hasSize(2)
    }
}
```

### Pattern: filtering by size from the command line

The `@SmallTest` / `@MediumTest` / `@LargeTest` annotations bind directly to `RunnerArgs.SIZE` (`runner-1.7.0/androidx/test/internal/runner/RunnerArgs.java` L58). For instrumented tests:

```bash
adb shell am instrument -w -r -e size small \
    com.example.test/androidx.test.runner.AndroidJUnitRunner
```

For Robolectric-on-JVM tests, Gradle property filtering is the standard route:

```bash
./gradlew :app:testDebugUnitTest --tests "com.example.*" \
    -PandroidTestSize=small
```

Combined with `-e annotation com.example.SmokeTest` (the `RunnerArgs.ANNOTATION` arg, L60) you get composable filtering.

## Mandatory rules

- **MUST** import `AndroidJUnit4` from `androidx.test.ext.junit.runners`. **MUST NOT** import `androidx.test.runner.AndroidJUnit4` — it is `@Deprecated`.
- **MUST** import `InstrumentationRegistry` from `androidx.test.platform.app`. **MUST NOT** import `androidx.test.InstrumentationRegistry` — it is `@Deprecated`.
- **MUST** use `ApplicationProvider.getApplicationContext()` for the Application context. **MUST NOT** call `InstrumentationRegistry.getInstrumentation().getTargetContext()` directly — `ApplicationProvider` is the canonical entry point and handles the unchecked cast.
- **MUST** install a `MainDispatcherRule` (or equivalent `Dispatchers.setMain`/`resetMain` plumbing) before any test that touches `viewModelScope`, `LaunchedEffect`, or any `Dispatchers.Main` consumer. Without it, the test crashes with `IllegalStateException: Module with the Main dispatcher is missing`.
- **MUST** prefer `StandardTestDispatcher` over `UnconfinedTestDispatcher` in `MainDispatcherRule`. `StandardTestDispatcher` matches `runTest` semantics; eager dispatch is a footgun for ordering.
- **MUST** annotate every test class or method with exactly ONE size annotation (`@SmallTest` / `@MediumTest` / `@LargeTest`) when the suite is filtered by size. Multiple size annotations on the same target produce undefined runner behaviour.
- **MUST** pin Robolectric independently — it does not ship in the androidx-test BOM. Use `org.robolectric:robolectric:4.x` matching the `compileSdk` of the module.
- **MUST NOT** use `runBlockingTest { }` in new code — it is `@Deprecated(level = DeprecationLevel.ERROR)` since `kotlinx-coroutines-test` 1.7. Use `runTest { }` instead.
- **MUST NOT** use `ActivityTestRule<A>` — it is `@Deprecated`. Use `androidx.test.ext.junit.rules.ActivityScenarioRule<A>` instead (`junit-1.3.0/androidx/test/ext/junit/rules/ActivityScenarioRule.java` L56). For instrumented variants, see `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.
- **PREFERRED:** declare `testOptions.unitTests.isIncludeAndroidResources = true` in the module's `android { … }` block. Robolectric needs merged resources to inflate any layout or read `R.string.*`.

## Verification

- [ ] `./gradlew :<module>:testDebugUnitTest` compiles and reports `Tests run: N, Failures: 0` for the new suite.
- [ ] `grep -r "import androidx.test.runner.AndroidJUnit4" src/test src/androidTest` returns NO matches (only `androidx.test.ext.junit.runners.AndroidJUnit4` is allowed).
- [ ] `grep -r "import androidx.test.InstrumentationRegistry" src/test src/androidTest` returns NO matches.
- [ ] Any test that uses `viewModelScope` / `Dispatchers.Main` declares `@get:Rule val mainDispatcherRule = MainDispatcherRule()` and uses `runTest { … }`.
- [ ] At least one test class is annotated with `@SmallTest`, `@MediumTest`, or `@LargeTest` if the suite is meant to be filtered.
- [ ] `./gradlew :<module>:testDebugUnitTest --info` output shows `org.robolectric.RobolectricTestRunner` (host) or `AndroidJUnit4ClassRunner` (device) in the runner stack — confirming the `AndroidJUnit4` delegate selection.

## References

- AndroidX Test releases: https://developer.android.com/jetpack/androidx/releases/test
- Local tests guide: https://developer.android.com/training/testing/local-tests
- Instrumented tests guide: https://developer.android.com/training/testing/instrumented-tests
- AndroidJUnitRunner reference: https://developer.android.com/training/testing/junit-runner
- Truth subject reference (upstream tree): https://github.com/android/android-test/tree/main/ext/truth/java/androidx/test/ext/truth
- `runner-1.7.0/androidx/test/runner/AndroidJUnit4.java` (L41-44) — `@Deprecated` alias.
- `junit-1.3.0/androidx/test/ext/junit/runners/AndroidJUnit4.java` (L49-70) — canonical runner with Robolectric/Android delegate selection.
- `monitor-1.8.0/androidx/test/platform/app/InstrumentationRegistry.java` (L30-85) — canonical registry; `getInstrumentation`, `getArguments`, `registerInstance`.
- `monitor-1.8.0/androidx/test/InstrumentationRegistry.java` (L34) — `@Deprecated` legacy registry.
- `core-1.7.0/androidx/test/core/app/ApplicationProvider.java` (L29-43) — single-method facade returning the Application context.
- `runner-1.7.0/androidx/test/filters/SmallTest.java` (L44-46), `MediumTest.java`, `LargeTest.java` — size annotation definitions.
- `runner-1.7.0/androidx/test/internal/runner/RunnerArgs.java` (L54-99) — every `-e <key> <value>` argument the runner accepts, including `size`, `annotation`, `class`, `package`, `numShards`/`shardIndex`.
- `testutils/testutils-ktx/src/jvmMain/kotlin/androidx/testutils/MainDispatcherRule.jvm.kt` — canonical androidx `MainDispatcherRule` source, identical pattern shipped here.
- Cross-set: `../../../kotlin/kotlin-test/writing-tests-with-kotlin-test/SKILL.md` — the framework-agnostic `kotlin.test` assertions (`assertEquals`, `assertFailsWith`, `@BeforeTest`) used inside these JUnit4 tests; also the only assertion API available in `commonTest`.
