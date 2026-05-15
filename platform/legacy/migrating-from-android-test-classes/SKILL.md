---
name: migrating-from-android-test-classes
description: Use this skill to migrate a codebase off the deprecated `android.test.*` testing classes that ship in the Android platform SDK onto AndroidX Test. Covers `InstrumentationTestRunner` → `AndroidJUnitRunner`; `InstrumentationTestCase`/`AndroidTestCase` → `@RunWith(AndroidJUnit4)` + `InstrumentationRegistry`/`ApplicationProvider`; `ActivityInstrumentationTestCase2` → `ActivityScenario`/`ActivityScenarioRule` + Espresso; `ServiceTestCase`/`ProviderTestCase2` → `ServiceTestRule`/`ProviderTestRule`; `MoreAsserts` → Truth/Hamcrest; `ViewAsserts`/`TouchUtils` → Espresso matchers/actions; `android.test.UiThreadTest`/`FlakyTest`/`suitebuilder.annotation.*` → `androidx.test.*`; `android.test.mock.*` → real test contexts or Mockito; and JUnit3 `extends TestCase` → JUnit4 `@Test`. Use when the user sees `@Deprecated` on `android.test.*`, asks how to replace `ActivityInstrumentationTestCase2` / `InstrumentationTestCase` / `MoreAsserts` / `TouchUtils`, or "old Android tests won't run on AndroidJUnitRunner".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - legacy-tests
  - android-test-package
  - InstrumentationTestCase
  - ActivityInstrumentationTestCase2
  - ServiceTestCase
  - MoreAsserts
  - TouchUtils
  - androidx-test-migration
  - deprecated-test-api
---

# Migrating Off android.test.* — From the Platform's Legacy Test Classes to AndroidX Test

The `android.test.*` package shipped in the Android SDK (`InstrumentationTestCase`, `ActivityInstrumentationTestCase2`, `AndroidTestCase`, `ServiceTestCase`, `MoreAsserts`, `TouchUtils`, `android.test.mock.*`, …) is `@Deprecated` — the platform's own Javadoc routes every class to AndroidX Test. Old codebases still carry these, and they actively hold back JUnit4, Espresso, `ActivityScenario`, and the AndroidX runner. This skill is the class-by-class replacement map and the mechanical migration patterns.

## When to use this skill

- The IDE flags `android.test.InstrumentationTestCase` / `AndroidTestCase` / `ActivityInstrumentationTestCase2` / `ServiceTestCase` / `MoreAsserts` / `ViewAsserts` / `TouchUtils` / `android.test.mock.*` as `@Deprecated`.
- A module's `testInstrumentationRunner` is `android.test.InstrumentationTestRunner` and tests behave oddly under `AndroidJUnitRunner` (or won't be discovered).
- The user asks how to replace a specific legacy base class: "`getInstrumentation()` outside a `TestCase`", "`getActivity()` / `setActivityIntent()` without `ActivityInstrumentationTestCase2`", "`getContext()` without `AndroidTestCase`".
- A test extends `junit.framework.TestCase` (JUnit3 style: `setUp()` override, `testFoo()` naming, `assertEquals` from `junit.framework.Assert`) and the team wants JUnit4.
- The user mentions `MoreAsserts.assertContentsInAnyOrder` / `assertEquals(int[], int[])` / `assertContainsRegex`, `ViewAsserts.assertOnScreen`, `TouchUtils.tapView`/`dragQuarterScreenDown`, or `android.test.suitebuilder.annotation.SmallTest`.

## When NOT to use this skill

- The codebase is already on AndroidX Test and the question is about *using* it (the runner, `ActivityScenario`, Espresso, `ServiceTestRule`) — go straight to `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`, `../../../instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md`, `../../../instrumentation/espresso/writing-espresso-tests/SKILL.md`.
- The question is about `kotlin.test` assertions (`assertEquals`, `assertFailsWith`) — use `../../../kotlin/kotlin-test/writing-tests-with-kotlin-test/SKILL.md`. (`kotlin.test` is one valid target for `MoreAsserts` calls.)
- The legacy class in question is a Robolectric shadow or a JUnit3 *non-Android* `TestCase` in pure-JVM code — for the JVM/Robolectric side use `../../../jvm-tests/robolectric/using-robolectric-correctly/SKILL.md` and `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`.
- The user wants to *write a new* test — do not start from `android.test.*` at all; use the AndroidX skills directly.

## Prerequisites

- AndroidX Test on the test classpath — see `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md` for the exact coordinate matrix (`androidx.test:core`, `:runner`, `:rules`, `androidx.test.ext:junit`). For host/JVM tests, `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`.
- JUnit4 on the classpath (`junit:junit:4.13.2` or via `kotlin("test")`). The legacy classes are JUnit3-shaped (`extends TestCase`); the migration target is JUnit4 (`@Test` + `@RunWith`).
- Espresso (`androidx.test.espresso:espresso-core`) for anything that was using `TouchUtils` / `ViewAsserts` / `ActivityInstrumentationTestCase2`'s UI poking.

## The replacement map

| Legacy `android.test.*` (deprecated) | Use instead | Notes |
|---|---|---|
| `InstrumentationTestRunner` (the `testInstrumentationRunner`) | `androidx.test.runner.AndroidJUnitRunner` | Change `android.defaultConfig.testInstrumentationRunner`. The legacy runner does not run JUnit4 or Espresso correctly. |
| `InstrumentationTestCase` | `@RunWith(AndroidJUnit4::class)` + `InstrumentationRegistry.getInstrumentation()` | `getInstrumentation()` → `androidx.test.platform.app.InstrumentationRegistry.getInstrumentation()`. `injectInstrumentation(...)` is gone. `sendKeys`/`runTestOnUiThread` → `getInstrumentation().runOnMainSync { … }` or `@UiThreadTest`. |
| `AndroidTestCase` | `@RunWith(AndroidJUnit4::class)` + `ApplicationProvider.getApplicationContext()` | `getContext()` → `ApplicationProvider.getApplicationContext<Context>()`. `getTestContext()` (the test APK's context) → `InstrumentationRegistry.getInstrumentation().context`. |
| `ApplicationTestCase<T>` | `@RunWith(AndroidJUnit4::class)`, drive `Application` via `ApplicationProvider.getApplicationContext()` | No 1:1 base class — the `Application` is just a context now; assert on it directly. |
| `ActivityInstrumentationTestCase2<T>`, `ActivityUnitTestCase<T>`, `SingleLaunchActivityTestCase<T>` | `ActivityScenario<T>` / `@get:Rule ActivityScenarioRule<T>` (+ Espresso for UI) | `getActivity()` → `scenario.onActivity { activity -> … }`. `setActivityIntent(intent)` → `ActivityScenario.launch<T>(intent)`. `getInstrumentation().waitForIdleSync()` → Espresso's automatic sync. See `../../../instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md`. |
| (Fragment cases, if any home-grown) | `FragmentScenario` / `launchFragmentInContainer` | `../../../instrumentation/scenarios/launching-fragments-with-fragmentscenario/SKILL.md`. |
| `ServiceTestCase<T>` | `androidx.test.rule.ServiceTestRule` | `startService(intent)` / `bindService(intent)` move onto the rule (`@get:Rule val serviceRule = ServiceTestRule()`); `getService()` → the `IBinder` from `serviceRule.bindService(intent)`. |
| `ProviderTestCase2<T>` | `androidx.test.rule.provider.ProviderTestRule` | `ProviderTestCase2` is the JUnit3-shaped form; `ProviderTestRule.Builder(MyProvider::class.java, AUTHORITY).build()` is the JUnit4 way and still gives an isolated `ContentResolver`. |
| `LoaderTestCase` | (Loaders are themselves deprecated) | Migrate the Loader to a `ViewModel` + coroutines/`Flow` first, then test that with `runTest` / Turbine. |
| `MoreAsserts` (`assertContentsInAnyOrder`, `assertEquals(int[], int[])`, `assertEmpty`, `assertNotEqual`, `assertContainsRegex`, `checkEqualsAndHashCodeMethods`) | Google Truth (`assertThat(list).containsExactly(...)`, `.isEmpty()`, `.containsMatch(regex)`), Hamcrest, or `kotlin.test.assertContentEquals` for ordered arrays | The platform Javadoc says "use Hamcrest"; Truth reads better. `androidx.test.ext:truth` ships Android-specific Truth subjects. |
| `ViewAsserts` (`assertOnScreen`, `assertHorizontalCenterAligned`, `assertBottomAligned`, …) | Espresso `ViewMatchers` + `ViewAssertions` (`matches(isCompletelyDisplayed())`, layout-relation custom matchers) | See `../../../instrumentation/espresso/writing-espresso-tests/SKILL.md`. |
| `TouchUtils` (`tapView`, `clickView`, `longClickView`, `dragQuarterScreenDown`, `scrollToBottom`, `dragViewToTop`) | Espresso `ViewActions` (`click()`, `longClick()`, `swipeUp()`/`swipeDown()`, `scrollTo()`, `GeneralSwipeAction`) | The legacy methods need a live `Instrumentation` + Activity reference; Espresso actions run inside `onView(...).perform(...)`. |
| `android.test.UiThreadTest` | `androidx.test.annotation.UiThreadTest` | Just change the import; or replace with `getInstrumentation().runOnMainSync { … }` / `scenario.onActivity { … }` (which is already on the main thread). |
| `android.test.FlakyTest` | `androidx.test.filters.FlakyTest` | Import change. |
| `android.test.suitebuilder.annotation.SmallTest` / `MediumTest` / `LargeTest` / `Suppress` | `androidx.test.filters.SmallTest` / `MediumTest` / `LargeTest` / `Suppress` | Import change; the runner reads `-e size small` against the AndroidX annotations. |
| `android.test.InstrumentationTestSuite`, `android.test.suitebuilder.TestSuiteBuilder` | JUnit4 `@RunWith(Suite::class)` + `@Suite.SuiteClasses([...])`, or just let `AndroidJUnitRunner` discover by `@SmallTest`/package | Manual suite building is rarely needed once the AndroidX runner is in place. |
| `android.test.mock.MockContext` | A real context (`ApplicationProvider.getApplicationContext()`), or Robolectric's context on the JVM, or `mock(Context::class.java)` (Mockito/MockK) for tight isolation | `MockContext` throws `UnsupportedOperationException` from almost every method — it was a hand-stubbing base, which a mocking framework does better. |
| `android.test.mock.MockContentResolver` + `MockContentProvider` | `ProviderTestRule` (isolated resolver), or a real `ContentResolver` from a Robolectric/instrumented context | If you genuinely need a stub resolver, Mockito a `ContentResolver`. |
| `android.test.mock.MockPackageManager` | Mockito/MockK `mock(PackageManager::class.java)` | The platform Javadoc explicitly says "use a mocking framework like Mockito". |
| `android.test.mock.MockResources` / `MockCursor` / `MockDialogInterface` / `MockApplication` / `MockService` | Mockito/MockK mocks, or real instances from a test context | Same story — these were stub bases, mocking frameworks supersede them. |
| `android.test.AndroidTestRunner`, `android.test.PerformanceTestCase`, `android.test.RepetitiveTest` | `AndroidJUnitRunner`; `androidx.benchmark` (Microbenchmark/Macrobenchmark); a custom JUnit4 `@Rule` or just a loop | `PerformanceTestCase` predates `androidx.benchmark` — real perf testing belongs there. |
| `extends junit.framework.TestCase` + `assertEquals` from `junit.framework.Assert` (JUnit3 shape) | `@RunWith(AndroidJUnit4::class)` + `@Test` methods + `org.junit.Assert.*` / `kotlin.test.*` | Method names no longer need a `test` prefix; `setUp`/`tearDown` overrides become `@Before`/`@After` (or `@BeforeTest`/`@AfterTest`). |

> The platform's own `@deprecated` Javadoc often points at the *old* "Android Testing Support Library" / `android.support.test.*` names (e.g. `ActivityTestRule`). Those became `androidx.test.*`, and several have since been re-deprecated again — `ActivityTestRule` → `ActivityScenario`. Migrate to the **current** target in the table above, not to whatever the platform Javadoc literally says.

## Workflow

- [ ] **1. Flip the runner first.** Set `android.defaultConfig.testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"` and add the AndroidX Test deps. Until this is done, JUnit4/Espresso/`ActivityScenario` cannot run. (`../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.)

- [ ] **2. Convert one test class at a time, JUnit3 → JUnit4.** Drop `extends …TestCase`, add `@RunWith(AndroidJUnit4::class)`, annotate test methods `@Test`, convert `setUp()`/`tearDown()` overrides to `@Before`/`@After` (no `super` call), and rename `testFoo` → `foo` if you like. Replace each legacy accessor with its AndroidX equivalent from the table.

- [ ] **3. Replace base-class superpowers with explicit AndroidX entry points.** `getContext()` → `ApplicationProvider.getApplicationContext()`; `getInstrumentation()` → `InstrumentationRegistry.getInstrumentation()`; `getActivity()` → `ActivityScenario` + `onActivity { }`; `getService()` → `ServiceTestRule.bindService(...)`.

- [ ] **4. Swap the assertion helpers.** `MoreAsserts.*` → Truth/`kotlin.test`; `ViewAsserts.*` → Espresso `ViewAssertions`; `TouchUtils.*` → Espresso `ViewActions`.

- [ ] **5. Swap the mock contexts.** Anything extending `android.test.mock.MockContext`/`MockResources`/`MockPackageManager` → a real test context where possible, otherwise a Mockito/MockK mock. (`../../../jvm-tests/mocking/mocking-with-mockito/SKILL.md`.)

- [ ] **6. Delete the legacy imports and confirm nothing references `android.test.*` or `android.test.mock.*` any more.** A clean module compiles with zero `import android.test.` lines.

## Patterns

### Pattern: `ActivityInstrumentationTestCase2` → `ActivityScenario`

```kotlin
// WRONG — deprecated base class
class LoginActivityTest : ActivityInstrumentationTestCase2<LoginActivity>(LoginActivity::class.java) {
    fun testTitleShown() {
        val activity = activity                       // getActivity()
        assertEquals("Sign in", activity.title)
    }
}
// WRONG because: android.test.ActivityInstrumentationTestCase2 is @Deprecated; it ties the test to
// the legacy InstrumentationTestRunner, blocks JUnit4/Espresso, and getActivity() leaks the Activity
// off the main thread.
```

```kotlin
// RIGHT — ActivityScenario + JUnit4
@RunWith(AndroidJUnit4::class)
class LoginActivityTest {
    @get:Rule val scenario = ActivityScenarioRule(LoginActivity::class.java)

    @Test fun titleShown() {
        scenario.scenario.onActivity { activity ->
            assertEquals("Sign in", activity.title)
        }
        onView(withId(R.id.title)).check(matches(withText("Sign in")))   // or assert via Espresso
    }
}
```

### Pattern: `AndroidTestCase` → `@RunWith(AndroidJUnit4)` + `ApplicationProvider`

```kotlin
// WRONG
class FormatterTest : AndroidTestCase() {
    fun testCurrency() {
        val s = CurrencyFormatter(context).format(1099)   // getContext()
        assertEquals("$10.99", s)
    }
}
// WRONG because: android.test.AndroidTestCase is @Deprecated; getContext() and the TestCase shape
// are the legacy world.
```

```kotlin
// RIGHT
@RunWith(AndroidJUnit4::class)
class FormatterTest {
    private val context: Context = ApplicationProvider.getApplicationContext()

    @Test fun currency() {
        assertEquals("$10.99", CurrencyFormatter(context).format(1099))
    }
}
```

### Pattern: `MoreAsserts` / `TouchUtils` calls

```kotlin
// WRONG
MoreAsserts.assertContentsInAnyOrder(ids, 3L, 1L, 2L)         // android.test.MoreAsserts (deprecated)
TouchUtils.tapView(this, listView)                            // android.test.TouchUtils (deprecated)
ViewAsserts.assertOnScreen(rootView, listView)                // android.test.ViewAsserts (deprecated)
```

```kotlin
// RIGHT
assertThat(service.ids()).containsExactly(1L, 2L, 3L)         // Google Truth (order-insensitive)
onView(withId(R.id.list)).perform(click())                    // Espresso ViewActions
onView(withId(R.id.list)).check(matches(isCompletelyDisplayed()))   // Espresso ViewAssertions
```

### Pattern: `android.test.mock.MockContext` subclass → real context or Mockito

```kotlin
// WRONG
class FakeContext : MockContext() {                            // android.test.mock.MockContext (deprecated)
    override fun getPackageName() = "com.example"
    // every other method throws UnsupportedOperationException
}
```

```kotlin
// RIGHT — a real test context if the SUT just reads from it
val context: Context = ApplicationProvider.getApplicationContext()

// RIGHT — a mock if you must control specific calls in isolation
val context = mock<Context> { on { packageName } doReturn "com.example" }
```

## Mandatory rules

- **MUST NOT** add new tests using any `android.test.*` or `android.test.mock.*` class — all are `@Deprecated`. New tests use AndroidX Test (`AndroidJUnit4`, `ActivityScenario`, `ServiceTestRule`, `ProviderTestRule`, Espresso) and `org.junit`/`kotlin.test` assertions.
- **MUST** switch `testInstrumentationRunner` to `androidx.test.runner.AndroidJUnitRunner` before (or as the first step of) migrating any test class — the legacy `InstrumentationTestRunner` does not run JUnit4/Espresso.
- **MUST** replace `getActivity()`/`setActivityIntent()` with `ActivityScenario` (`launch`, `onActivity { }`), not with `ActivityTestRule` — `ActivityTestRule` is itself deprecated.
- **MUST** replace `android.test.mock.MockContext`/`MockResources`/`MockPackageManager`/`MockContentResolver` subclasses with a real test context where the SUT only reads from it, otherwise a Mockito/MockK mock — never re-derive a new hand-stub from `MockContext`.
- **MUST** convert JUnit3 shape (`extends TestCase`, `testFoo()`, `setUp()`/`tearDown()` overrides, `junit.framework.Assert`) to JUnit4 (`@RunWith(AndroidJUnit4)`, `@Test`, `@Before`/`@After`, `org.junit.Assert`/`kotlin.test`) — do not leave a class half-migrated.
- **MUST NOT** trust the platform `@deprecated` Javadoc's literal target (it often names the obsolete `android.support.test.*` API); migrate to the current AndroidX target in this skill's table.
- **PREFERRED:** migrate one test class fully per change, with the runner flip landing first; a half-migrated module (some classes on `android.test.*`, some on AndroidX) is hard to reason about.
- **PREFERRED:** Google Truth (`androidx.test.ext:truth` for the Android subjects) over Hamcrest as the `MoreAsserts` replacement — better failure messages.

## Verification

- [ ] `git grep -n 'import android.test\.' -- '*.java' '*.kt'` is empty (or limited to a tracked, justified holdout list).
- [ ] No module's `testInstrumentationRunner` is `android.test.InstrumentationTestRunner`; all are `androidx.test.runner.AndroidJUnitRunner`.
- [ ] No test class `extends ActivityInstrumentationTestCase2` / `AndroidTestCase` / `InstrumentationTestCase` / `ServiceTestCase` / `ProviderTestCase2` / `junit.framework.TestCase`.
- [ ] No `MoreAsserts.` / `ViewAsserts.` / `TouchUtils.` references remain.
- [ ] No subclass of `android.test.mock.MockContext` / `MockResources` / `MockPackageManager` / `MockContentResolver` remains.
- [ ] Migrated instrumentation tests run green under `./gradlew :module:connectedDebugAndroidTest`; migrated host tests under `./gradlew :module:testDebugUnitTest`.

## References

- developer.android.com/training/testing — AndroidX Test overview; the migration destination for everything in `android.test.*`.
- developer.android.com/reference/android/test/package-summary — the `android.test` package reference, every class marked deprecated with its replacement pointer.
- `frameworks/base/test-base/src/android/test/AndroidTestCase.java`, `InstrumentationTestCase.java`, `UiThreadTest.java`, `FlakyTest.java` (AOSP) — `@Deprecated` classes whose Javadoc routes to `InstrumentationRegistry` / `androidx.test.annotation.UiThreadTest` / `androidx.test.filters.FlakyTest`.
- `frameworks/base/test-runner/src/android/test/ActivityInstrumentationTestCase2.java`, `ServiceTestCase.java`, `ProviderTestCase2.java`, `InstrumentationTestRunner.java`, `MoreAsserts.java`, `ViewAsserts.java`, `TouchUtils.java` (AOSP) — the deprecated runner/base classes; Javadocs name `ActivityTestRule`/`ServiceTestRule`/`AndroidJUnitRunner`/Hamcrest/Espresso as replacements.
- `frameworks/base/test-mock/src/android/test/mock/MockContext.java`, `MockPackageManager.java`, `MockContentResolver.java`, `MockResources.java`, `MockCursor.java` (AOSP) — the stub-context bases; `MockPackageManager`'s Javadoc explicitly says "use a mocking framework like Mockito".
- Cross-set: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md` — the `AndroidJUnitRunner` / AndroidX Test coordinate matrix to land first.
- Cross-set: `../../../instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md` — `ActivityScenario` / `ActivityScenarioRule`, the replacement for `ActivityInstrumentationTestCase2`.
- Cross-set: `../../../instrumentation/scenarios/launching-fragments-with-fragmentscenario/SKILL.md` — `FragmentScenario` for any home-grown fragment-test bases.
- Cross-set: `../../../instrumentation/espresso/writing-espresso-tests/SKILL.md` — Espresso `ViewMatchers`/`ViewActions`/`ViewAssertions`, the replacement for `ViewAsserts`/`TouchUtils`.
- Cross-set: `../../../jvm-tests/mocking/mocking-with-mockito/SKILL.md` — mocking `Context`/`PackageManager`/`Resources`, the replacement for `android.test.mock.*`.
- Cross-set: `../../../jvm-tests/robolectric/using-robolectric-correctly/SKILL.md` — real Android stubs on the JVM, an alternative to a mock context.
- Cross-set: `../../../kotlin/kotlin-test/writing-tests-with-kotlin-test/SKILL.md` — `kotlin.test` assertions (`assertEquals`, `assertContentEquals`, `assertFailsWith`), one valid replacement for `MoreAsserts` and `junit.framework.Assert`.
- Cross-set: `../../../fundamentals/strategies/organizing-test-source-sets/SKILL.md` — `src/test/` vs `src/androidTest/`, which the migrated classes need to be sorted into.
