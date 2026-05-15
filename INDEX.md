# INDEX — Symptom and API lookup for android-testing-skills

This index maps a developer's symptom or API of interest to the skill that resolves it. Skim **By
symptom** first, then **By API**, then **By workflow phase** to understand where each skill fits
in the loop.

Seven sets:

- [`compose/`](compose/) — 25 skills, Jetpack Compose UI testing (incl. preview-driven development, preview screenshot catalogs in CI, stability validation)
- [`fundamentals/`](fundamentals/) — 5 skills, the conceptual foundation
- [`kotlin/`](kotlin/) — 1 skill, Kotlin-native test tooling (the `kotlin.test` multiplatform library)
- [`jvm-tests/`](jvm-tests/) — 6 skills, JVM unit tests
- [`instrumentation/`](instrumentation/) — 6 skills, on-device tests (incl. Gradle Managed Devices)
- [`platform/`](platform/) — 1 skill, platform-shipped test tooling (migrating off the deprecated `android.test.*` classes)
- [`adb/`](adb/) — 10 skills, ADB-driven device E2E

## By symptom

### Fundamentals & strategy

| Symptom | Skill |
|---|---|
| Don't know where to start writing tests for a new feature | [fundamentals/concepts/choosing-what-to-test](fundamentals/concepts/choosing-what-to-test/SKILL.md) |
| Test pyramid feels arbitrary or upside-down (too many big tests, too few small) | [fundamentals/concepts/understanding-the-testing-pyramid](fundamentals/concepts/understanding-the-testing-pyramid/SKILL.md) |
| Confused between fake / mock / stub / spy / dummy | [fundamentals/doubles/picking-test-doubles](fundamentals/doubles/picking-test-doubles/SKILL.md) |
| Tests are flaky and time-dependent — what's the strategy? | [fundamentals/strategies/applying-testing-strategies](fundamentals/strategies/applying-testing-strategies/SKILL.md) |
| `src/sharedTest/` build is broken on AGP 7.2+ | [fundamentals/strategies/organizing-test-source-sets](fundamentals/strategies/organizing-test-source-sets/SKILL.md) |
| Confused about `testImplementation` vs `androidTestImplementation` vs `debugImplementation` | [fundamentals/strategies/organizing-test-source-sets](fundamentals/strategies/organizing-test-source-sets/SKILL.md) |
| Hilt `@HiltAndroidTest` rule ordering / `@UninstallModules` trouble | [fundamentals/strategies/applying-testing-strategies](fundamentals/strategies/applying-testing-strategies/SKILL.md) |

### Kotlin (`kotlin.test`)

| Symptom | Skill |
|---|---|
| `import org.junit.*` unresolved in `commonTest` / want one assertion API across targets | [kotlin/kotlin-test/writing-tests-with-kotlin-test](kotlin/kotlin-test/writing-tests-with-kotlin-test/SKILL.md) |
| `testImplementation(kotlin("test"))` — runs on JUnit but no JUnit was added; JUnit4 vs JUnit5 selection | [kotlin/kotlin-test/writing-tests-with-kotlin-test](kotlin/kotlin-test/writing-tests-with-kotlin-test/SKILL.md) |
| `assertFailsWith` / `assertContentEquals` / `assertIs` / `@BeforeTest` — how/when | [kotlin/kotlin-test/writing-tests-with-kotlin-test](kotlin/kotlin-test/writing-tests-with-kotlin-test/SKILL.md) |
| `kotlin.test.assertEquals` failure shows a bare `AssertionError` with no diff | [kotlin/kotlin-test/writing-tests-with-kotlin-test](kotlin/kotlin-test/writing-tests-with-kotlin-test/SKILL.md) |

### JVM unit tests

| Symptom | Skill |
|---|---|
| `@RunWith(JUnit4::class)` — Android stubs not initializing | [jvm-tests/runner/configuring-junit4-on-android](jvm-tests/runner/configuring-junit4-on-android/SKILL.md) |
| Pick between Mockito and MockK | [jvm-tests/mocking/mocking-with-mockito](jvm-tests/mocking/mocking-with-mockito/SKILL.md), [jvm-tests/mocking/mocking-with-mockk](jvm-tests/mocking/mocking-with-mockk/SKILL.md) |
| `Cannot mock/spy final class` from Mockito | [jvm-tests/mocking/mocking-with-mockito](jvm-tests/mocking/mocking-with-mockito/SKILL.md) |
| Suspending function — Mockito stub doesn't work | [jvm-tests/mocking/mocking-with-mockk](jvm-tests/mocking/mocking-with-mockk/SKILL.md), [jvm-tests/coroutines/testing-coroutines-with-runtest](jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md) |
| Need to mock a static / `object` / constructor | [jvm-tests/mocking/mocking-with-mockk](jvm-tests/mocking/mocking-with-mockk/SKILL.md) |
| `runBlockingTest` is deprecated | [jvm-tests/coroutines/testing-coroutines-with-runtest](jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md) |
| `Dispatchers.Main` not initialized in test | [jvm-tests/coroutines/testing-coroutines-with-runtest](jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md) |
| Testing a `Flow` that emits multiple values | [jvm-tests/coroutines/testing-flows-with-turbine](jvm-tests/coroutines/testing-flows-with-turbine/SKILL.md) |
| `SharedFlow.toList()` hangs the test | [jvm-tests/coroutines/testing-flows-with-turbine](jvm-tests/coroutines/testing-flows-with-turbine/SKILL.md) |
| Verify a `(T) -> Unit` callback / listener was invoked (with X, and only that) | [jvm-tests/coroutines/testing-flows-with-turbine](jvm-tests/coroutines/testing-flows-with-turbine/SKILL.md) |
| Robolectric test passes locally, fails in CI on a different SDK | [jvm-tests/robolectric/using-robolectric-correctly](jvm-tests/robolectric/using-robolectric-correctly/SKILL.md) |
| `R.string.foo` not found in Robolectric test | [jvm-tests/robolectric/using-robolectric-correctly](jvm-tests/robolectric/using-robolectric-correctly/SKILL.md) |
| Want a single test class that runs on JVM and on device | [jvm-tests/robolectric/using-robolectric-correctly](jvm-tests/robolectric/using-robolectric-correctly/SKILL.md), [fundamentals/strategies/organizing-test-source-sets](fundamentals/strategies/organizing-test-source-sets/SKILL.md) |

### Instrumentation tests

| Symptom | Skill |
|---|---|
| `@RunWith(AndroidJUnit4::class)` — which import? | [instrumentation/runner/running-instrumented-tests-with-androidjunit4](instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md) |
| `androidTestImplementation("androidx.test:orchestrator")` doesn't activate orchestrator | [instrumentation/runner/running-instrumented-tests-with-androidjunit4](instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md) |
| `ActivityTestRule` deprecation warning | [instrumentation/scenarios/launching-activities-with-activityscenario](instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md) |
| Need to test Activity lifecycle (recreate, moveToState) | [instrumentation/scenarios/launching-activities-with-activityscenario](instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md) |
| Need to test `Activity.RESULT_*` in isolation | [instrumentation/scenarios/launching-activities-with-activityscenario](instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md) |
| Test a Fragment in isolation without an Activity boilerplate | [instrumentation/scenarios/launching-fragments-with-fragmentscenario](instrumentation/scenarios/launching-fragments-with-fragmentscenario/SKILL.md) |
| Fragment test crashes on AppCompat theme inflation | [instrumentation/scenarios/launching-fragments-with-fragmentscenario](instrumentation/scenarios/launching-fragments-with-fragmentscenario/SKILL.md) |
| Espresso click on a button inside an `AlertDialog` does nothing | [instrumentation/espresso/writing-espresso-tests](instrumentation/espresso/writing-espresso-tests/SKILL.md) |
| `NoMatchingViewException` / `AmbiguousViewMatcherException` | [instrumentation/espresso/writing-espresso-tests](instrumentation/espresso/writing-espresso-tests/SKILL.md) |
| `BottomNavigationViewActions` cannot be resolved | [instrumentation/espresso/writing-espresso-tests](instrumentation/espresso/writing-espresso-tests/SKILL.md) |
| Test a flow that crosses into Settings or another app | [instrumentation/uiautomator/cross-app-tests-with-uiautomator](instrumentation/uiautomator/cross-app-tests-with-uiautomator/SKILL.md) |
| `StaleObjectException` from a `UiObject2` reference | [instrumentation/uiautomator/cross-app-tests-with-uiautomator](instrumentation/uiautomator/cross-app-tests-with-uiautomator/SKILL.md) |
| Need a reproducible emulator for instrumented tests in CI (no hand-managed AVD) | [instrumentation/managed-devices/running-tests-on-gradle-managed-devices](instrumentation/managed-devices/running-tests-on-gradle-managed-devices/SKILL.md) |
| `./gradlew allDevicesCheck` / `<device>DebugAndroidTest` — how to define managed devices | [instrumentation/managed-devices/running-tests-on-gradle-managed-devices](instrumentation/managed-devices/running-tests-on-gradle-managed-devices/SKILL.md) |
| CI emulator runs slow/heavy — what's an ATD image? | [instrumentation/managed-devices/running-tests-on-gradle-managed-devices](instrumentation/managed-devices/running-tests-on-gradle-managed-devices/SKILL.md) |

### Legacy `android.test.*` migration

| Symptom | Skill |
|---|---|
| `@Deprecated` on `android.test.InstrumentationTestCase` / `AndroidTestCase` / `ActivityInstrumentationTestCase2` / `ServiceTestCase` / `ProviderTestCase2` | [platform/legacy/migrating-from-android-test-classes](platform/legacy/migrating-from-android-test-classes/SKILL.md) |
| `testInstrumentationRunner` is `android.test.InstrumentationTestRunner`; tests misbehave under `AndroidJUnitRunner` | [platform/legacy/migrating-from-android-test-classes](platform/legacy/migrating-from-android-test-classes/SKILL.md) |
| Replace `MoreAsserts` / `ViewAsserts` / `TouchUtils` / `android.test.mock.MockContext` & friends | [platform/legacy/migrating-from-android-test-classes](platform/legacy/migrating-from-android-test-classes/SKILL.md) |
| `extends junit.framework.TestCase` (JUnit3 shape) → JUnit4 | [platform/legacy/migrating-from-android-test-classes](platform/legacy/migrating-from-android-test-classes/SKILL.md) |

### Compose UI tests

| Symptom | Skill |
|---|---|
| `ActivityNotFoundException: ComponentActivity` at test launch | [compose/setup/configuring-test-dependencies](compose/setup/configuring-test-dependencies/SKILL.md) |
| Lint warning `TestManifestGradleConfiguration` | [compose/setup/configuring-test-dependencies](compose/setup/configuring-test-dependencies/SKILL.md) |
| Mixing `createComposeRule` v1 and v2 | [compose/setup/choosing-test-rule-vs-runtest](compose/setup/choosing-test-rule-vs-runtest/SKILL.md) |
| Compose test passes locally, fails in CI on Robolectric | [compose/setup/setting-up-host-vs-device-tests](compose/setup/setting-up-host-vs-device-tests/SKILL.md) |
| "no node matched" / "0 nodes found" in Compose | [compose/debug/printing-the-semantics-tree](compose/debug/printing-the-semantics-tree/SKILL.md), [compose/finders/finding-nodes-by-tag-text-content](compose/finders/finding-nodes-by-tag-text-content/SKILL.md) |
| Error message hints "...were found in the unmerged tree" | [compose/finders/finding-nodes-by-tag-text-content](compose/finders/finding-nodes-by-tag-text-content/SKILL.md) |
| `assertExists` overuse where the test means "visible" | [compose/assertions/asserting-node-state-and-text](compose/assertions/asserting-node-state-and-text/SKILL.md) |
| `performGesture` deprecated | [compose/actions/injecting-touch-gestures](compose/actions/injecting-touch-gestures/SKILL.md) |
| Compose animation test flaky / `InfiniteAnimationPolicy CancellationException` | [compose/synchronization/testing-animations-deterministically](compose/synchronization/testing-animations-deterministically/SKILL.md) |
| `mainClock.advanceTimeBy(5)` only advances one frame, not 5 ms | [compose/synchronization/controlling-the-test-clock](compose/synchronization/controlling-the-test-clock/SKILL.md) |
| `waitUntil` timeout fires too quickly | [compose/synchronization/synchronizing-with-idle](compose/synchronization/synchronizing-with-idle/SKILL.md) |
| `rememberSaveable` state appears to vanish in Compose test | [compose/patterns/testing-state-restoration](compose/patterns/testing-state-restoration/SKILL.md) |
| Compose + Espresso (Dialog, IME, custom Activity) | [compose/interop/testing-with-espresso-interop](compose/interop/testing-with-espresso-interop/SKILL.md) |
| Need accessibility checks on every Compose action | [compose/debug/enabling-accessibility-checks](compose/debug/enabling-accessibility-checks/SKILL.md) |
| Inheriting an unfamiliar Compose test suite | [compose/audit/auditing-compose-test-suite](compose/audit/auditing-compose-test-suite/SKILL.md) |
| `@Preview` fails to render — needs a `ViewModel` / repository / `Context` | [compose/preview/developing-with-compose-previews](compose/preview/developing-with-compose-previews/SKILL.md) |
| Previews keep going "out of date" / breaking; how to structure them | [compose/preview/developing-with-compose-previews](compose/preview/developing-with-compose-previews/SKILL.md) |
| Want every UI state side by side without running the app (`@PreviewParameter`) | [compose/preview/developing-with-compose-previews](compose/preview/developing-with-compose-previews/SKILL.md) |
| Composable loads a Coil/Glide image and the preview is blank (`LocalInspectionMode`) | [compose/preview/developing-with-compose-previews](compose/preview/developing-with-compose-previews/SKILL.md) |
| Want `@Preview`s screenshotted on a real device and a browsable catalog in CI | [compose/preview/capturing-preview-screenshots-in-ci](compose/preview/capturing-preview-screenshots-in-ci/SKILL.md) |
| `captureAllPreviews` / HotSwan / preview catalog on GitHub Pages | [compose/preview/capturing-preview-screenshots-in-ci](compose/preview/capturing-preview-screenshots-in-ci/SKILL.md) |
| Paparazzi vs Roborazzi vs device-rendered preview screenshots — which? | [compose/preview/capturing-preview-screenshots-in-ci](compose/preview/capturing-preview-screenshots-in-ci/SKILL.md) |
| A composable became unskippable / a parameter went stable → unstable | [compose/stability/validating-compose-stability](compose/stability/validating-compose-stability/SKILL.md) |
| Want CI to fail on Compose stability regressions (`stabilityCheck` / baseline) | [compose/stability/validating-compose-stability](compose/stability/validating-compose-stability/SKILL.md) |

### ADB

| Symptom | Skill |
|---|---|
| `error: device not found` after USB reconnect | [adb/devices/connecting-to-devices](adb/devices/connecting-to-devices/SKILL.md), [adb/architecture/understanding-adb-architecture](adb/architecture/understanding-adb-architecture/SKILL.md) |
| `error: more than one device/emulator` | [adb/devices/connecting-to-devices](adb/devices/connecting-to-devices/SKILL.md) |
| Set up wireless ADB on Android 11+ | [adb/devices/connecting-over-wifi](adb/devices/connecting-over-wifi/SKILL.md) |
| `INSTALL_FAILED_USER_RESTRICTED` / signing-conflict on `adb install` | [adb/apps/installing-and-managing-apps](adb/apps/installing-and-managing-apps/SKILL.md) |
| `am force-stop` doesn't reset the app between tests | [adb/apps/installing-and-managing-apps](adb/apps/installing-and-managing-apps/SKILL.md), [adb/control/injecting-input-and-state](adb/control/injecting-input-and-state/SKILL.md) |
| `am instrument` returns 0 even though tests failed | [adb/tests/running-instrumented-tests-via-adb](adb/tests/running-instrumented-tests-via-adb/SKILL.md), [adb/automation/scripting-adb-for-ci](adb/automation/scripting-adb-for-ci/SKILL.md) |
| `svc wifi disable` is a silent no-op on API 30+ | [adb/control/injecting-input-and-state](adb/control/injecting-input-and-state/SKILL.md) |
| `input text "hello world"` drops everything after the space | [adb/control/injecting-input-and-state](adb/control/injecting-input-and-state/SKILL.md) |
| `screencap` produces a corrupt PNG | [adb/capture/capturing-screenshots-and-screenrecord](adb/capture/capturing-screenshots-and-screenrecord/SKILL.md) |
| `screenrecord --time-limit 600` silently capped at 180 | [adb/capture/capturing-screenshots-and-screenrecord](adb/capture/capturing-screenshots-and-screenrecord/SKILL.md) |
| `adb logcat | grep ...` is slow on a busy device | [adb/observability/extracting-logs-with-logcat](adb/observability/extracting-logs-with-logcat/SKILL.md) |
| Need just the crash from logcat, not noise | [adb/observability/extracting-logs-with-logcat](adb/observability/extracting-logs-with-logcat/SKILL.md) |
| `adb pull /data/data/<pkg>/...` denied | [adb/transfer/extracting-test-artifacts](adb/transfer/extracting-test-artifacts/SKILL.md) |
| Pulling a binary file via `adb shell run-as cat` corrupts it | [adb/transfer/extracting-test-artifacts](adb/transfer/extracting-test-artifacts/SKILL.md) |
| Run instrumented tests across N parallel emulators | [adb/automation/scripting-adb-for-ci](adb/automation/scripting-adb-for-ci/SKILL.md) |
| `adb forward` vs `adb reverse` argument-order confusion | [adb/automation/scripting-adb-for-ci](adb/automation/scripting-adb-for-ci/SKILL.md) |

## By API

| API or tool | Skill |
|---|---|
| `kotlin.test` — `@Test`/`@BeforeTest`/`@AfterTest`, `assertEquals`/`assertContentEquals`/`assertIs`/`assertFailsWith`/`expect`/`fail`, `Asserter`/`AsserterContributor`, `kotlin("test")`/`kotlin-test-junit`/`kotlin-test-junit5` | [kotlin/kotlin-test/writing-tests-with-kotlin-test](kotlin/kotlin-test/writing-tests-with-kotlin-test/SKILL.md) |
| `androidx.test:core / runner / rules / ext:junit / ext:truth / monitor` | [jvm-tests/runner/configuring-junit4-on-android](jvm-tests/runner/configuring-junit4-on-android/SKILL.md), [instrumentation/runner/running-instrumented-tests-with-androidjunit4](instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md) |
| `AndroidJUnit4`, `InstrumentationRegistry`, `ApplicationProvider` | [jvm-tests/runner/configuring-junit4-on-android](jvm-tests/runner/configuring-junit4-on-android/SKILL.md), [instrumentation/runner/running-instrumented-tests-with-androidjunit4](instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md) |
| `android.test.*` (deprecated) — `InstrumentationTestCase`/`AndroidTestCase`/`ActivityInstrumentationTestCase2`/`ServiceTestCase`/`ProviderTestCase2`/`InstrumentationTestRunner`/`MoreAsserts`/`ViewAsserts`/`TouchUtils`/`android.test.mock.*` | [platform/legacy/migrating-from-android-test-classes](platform/legacy/migrating-from-android-test-classes/SKILL.md) |
| `ActivityScenario`, `ActivityScenarioRule` | [instrumentation/scenarios/launching-activities-with-activityscenario](instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md) |
| `FragmentScenario`, `launchFragmentInContainer`, `launchFragment` | [instrumentation/scenarios/launching-fragments-with-fragmentscenario](instrumentation/scenarios/launching-fragments-with-fragmentscenario/SKILL.md) |
| `Espresso.onView`, `ViewMatchers`, `ViewActions`, `RootMatchers`, `IdlingResource` | [instrumentation/espresso/writing-espresso-tests](instrumentation/espresso/writing-espresso-tests/SKILL.md) |
| `Intents.init/release/intended/intending`, `IntentsRule` | [instrumentation/espresso/writing-espresso-tests](instrumentation/espresso/writing-espresso-tests/SKILL.md) |
| `RecyclerViewActions`, `DrawerActions`, `PickerActions`, `NavigationViewActions` | [instrumentation/espresso/writing-espresso-tests](instrumentation/espresso/writing-espresso-tests/SKILL.md) |
| `UiDevice`, `BySelector`, `UiObject2`, `Until`, `Configurator` | [instrumentation/uiautomator/cross-app-tests-with-uiautomator](instrumentation/uiautomator/cross-app-tests-with-uiautomator/SKILL.md) |
| `android.testOptions.managedDevices`, `ManagedVirtualDevice`, `localDevices`/`groups`, `allDevicesCheck`, `<device>DebugAndroidTest`, ATD images | [instrumentation/managed-devices/running-tests-on-gradle-managed-devices](instrumentation/managed-devices/running-tests-on-gradle-managed-devices/SKILL.md) |
| `Mockito`, `@Mock`, `whenever`, `argumentCaptor`, `mock-maker-inline` | [jvm-tests/mocking/mocking-with-mockito](jvm-tests/mocking/mocking-with-mockito/SKILL.md) |
| `MockK`, `mockk`, `every`, `coEvery`, `verify`, `coVerify`, `slot`, `mockkStatic`, `mockkObject`, `mockkConstructor` | [jvm-tests/mocking/mocking-with-mockk](jvm-tests/mocking/mocking-with-mockk/SKILL.md) |
| `runTest`, `TestScope`, `StandardTestDispatcher`, `UnconfinedTestDispatcher`, `TestCoroutineScheduler`, `Dispatchers.setMain` | [jvm-tests/coroutines/testing-coroutines-with-runtest](jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md) |
| `Turbine`, `flow.test`, `awaitItem`, `expectMostRecentItem`, `turbineScope`, `testIn`, standalone `Turbine<T>()` (`add`/`close`), `takeItem` | [jvm-tests/coroutines/testing-flows-with-turbine](jvm-tests/coroutines/testing-flows-with-turbine/SKILL.md) |
| `Robolectric`, `RobolectricTestRunner`, `@Config`, `Shadows.shadowOf`, `ShadowLooper` | [jvm-tests/robolectric/using-robolectric-correctly](jvm-tests/robolectric/using-robolectric-correctly/SKILL.md) |
| `Hilt`, `@HiltAndroidTest`, `HiltAndroidRule`, `@TestInstallIn`, `@UninstallModules`, `@BindValue` | [fundamentals/strategies/applying-testing-strategies](fundamentals/strategies/applying-testing-strategies/SKILL.md) |
| `createComposeRule`, `runComposeUiTest`, `ComposeTestRule`, `ComposeUiTest`, `StateRestorationTester` | [compose/setup/choosing-test-rule-vs-runtest](compose/setup/choosing-test-rule-vs-runtest/SKILL.md), [compose/patterns/testing-state-restoration](compose/patterns/testing-state-restoration/SKILL.md) |
| `onNodeWithTag`, `onNodeWithText`, `onNodeWithContentDescription`, `onAllNodes`, `onRoot`, `useUnmergedTree` | [compose/finders/finding-nodes-by-tag-text-content](compose/finders/finding-nodes-by-tag-text-content/SKILL.md) |
| `SemanticsMatcher`, `hasText`, `hasClickAction`, `hasScrollToKeyAction`, `hasParent`, `hasAnyAncestor`, `hasAnyDescendant` | [compose/finders/composing-semantics-matchers](compose/finders/composing-semantics-matchers/SKILL.md) |
| `onParent`, `onChildren`, `onSibling`, `onAncestors`, `filter`, `filterToOne` | [compose/finders/traversing-the-semantics-tree](compose/finders/traversing-the-semantics-tree/SKILL.md) |
| `assertExists`, `assertIsDisplayed`, `assertIsEnabled`, `assertIsOn`, `assertTextEquals`, `assertCountEquals` | [compose/assertions/asserting-node-state-and-text](compose/assertions/asserting-node-state-and-text/SKILL.md) |
| `assertWidthIsEqualTo`, `assertHeightIsAtLeast`, `getUnclippedBoundsInRoot`, `Dp.assertIsEqualTo` | [compose/assertions/asserting-bounds-and-dimensions](compose/assertions/asserting-bounds-and-dimensions/SKILL.md) |
| `performClick`, `performScrollToIndex`, `performScrollToKey`, `performScrollToNode` | [compose/actions/clicking-and-scrolling](compose/actions/clicking-and-scrolling/SKILL.md) |
| `performTouchInput`, `swipe`, `pinch`, `longClick`, `multiTouchSwipe` | [compose/actions/injecting-touch-gestures](compose/actions/injecting-touch-gestures/SKILL.md) |
| `performMouseInput`, `performKeyInput`, `withKeyDown`, `MouseButton`, `ScrollWheel` | [compose/actions/injecting-mouse-and-keyboard](compose/actions/injecting-mouse-and-keyboard/SKILL.md) |
| `performTextInput`, `performTextReplacement`, `performImeAction` | [compose/actions/entering-text](compose/actions/entering-text/SKILL.md) |
| `MainTestClock`, `autoAdvance`, `advanceTimeByFrame`, `advanceTimeBy`, `advanceTimeUntil` | [compose/synchronization/controlling-the-test-clock](compose/synchronization/controlling-the-test-clock/SKILL.md) |
| `mainClock.autoAdvance = false`, `InfiniteAnimationPolicy`, `onAnimationFrame` helper | [compose/synchronization/testing-animations-deterministically](compose/synchronization/testing-animations-deterministically/SKILL.md) |
| `waitForIdle`, `waitUntil`, `runOnIdle`, `runOnUiThread`, `IdlingResource`, `IdlingPolicies.setMasterPolicyTimeout` | [compose/synchronization/synchronizing-with-idle](compose/synchronization/synchronizing-with-idle/SKILL.md) |
| `printToLog`, `printToString`, `fetchSemanticsNode` | [compose/debug/printing-the-semantics-tree](compose/debug/printing-the-semantics-tree/SKILL.md) |
| `enableAccessibilityChecks`, `AccessibilityValidator`, `tryPerformAccessibilityChecks` | [compose/debug/enabling-accessibility-checks](compose/debug/enabling-accessibility-checks/SKILL.md) |
| `@Preview`, `@PreviewParameter`, `PreviewParameterProvider`, `LocalInspectionMode`, multipreview annotation classes | [compose/preview/developing-with-compose-previews](compose/preview/developing-with-compose-previews/SKILL.md) |
| `captureAllPreviews`, `HotSwanPreviewActivity`, `@PreviewScreenshot`, `hotSwanCompiler { preview { … } }` (Compose HotSwan) | [compose/preview/capturing-preview-screenshots-in-ci](compose/preview/capturing-preview-screenshots-in-ci/SKILL.md) |
| `stabilityDump`, `stabilityCheck`, `composeStabilityAnalyzer { stabilityValidation { … } }`, `@IgnoreStabilityReport` | [compose/stability/validating-compose-stability](compose/stability/validating-compose-stability/SKILL.md) |
| `adb start-server`, `adb kill-server`, `adb reconnect`, `adb --version` | [adb/architecture/understanding-adb-architecture](adb/architecture/understanding-adb-architecture/SKILL.md) |
| `adb devices [-l]`, `-s`, `-d`, `-e`, `-t`, `wait-for[-TRANSPORT]-<state>` | [adb/devices/connecting-to-devices](adb/devices/connecting-to-devices/SKILL.md) |
| `adb pair`, `adb connect`, `adb disconnect`, `adb mdns check/services`, `adb tcpip` | [adb/devices/connecting-over-wifi](adb/devices/connecting-over-wifi/SKILL.md) |
| `adb install [-r/-d/-t/-g]`, `pm list / clear / grant / disable / dump-profiles` | [adb/apps/installing-and-managing-apps](adb/apps/installing-and-managing-apps/SKILL.md) |
| `adb shell am instrument -w -r -e ...`, `INSTRUMENTATION_STATUS_CODE`, AndroidJUnitRunner status codes | [adb/tests/running-instrumented-tests-via-adb](adb/tests/running-instrumented-tests-via-adb/SKILL.md) |
| `adb shell input tap/swipe/text/keyevent`, `wm size/density`, `settings put`, `cmd <service>` | [adb/control/injecting-input-and-state](adb/control/injecting-input-and-state/SKILL.md) |
| `adb shell screencap`, `adb shell screenrecord`, `adb exec-out` | [adb/capture/capturing-screenshots-and-screenrecord](adb/capture/capturing-screenshots-and-screenrecord/SKILL.md) |
| `adb logcat -b`, `-v threadtime/json`, `-T/-t`, `--pid`, `-r/-n/-f`, `-G/-g` | [adb/observability/extracting-logs-with-logcat](adb/observability/extracting-logs-with-logcat/SKILL.md) |
| `adb pull/push -z/-Z/--sync/-a`, `adb exec-out run-as`, scoped storage | [adb/transfer/extracting-test-artifacts](adb/transfer/extracting-test-artifacts/SKILL.md) |
| `adb forward`, `adb reverse`, parallel `xargs`, `timeout`, retry, Test Orchestrator | [adb/automation/scripting-adb-for-ci](adb/automation/scripting-adb-for-ci/SKILL.md) |

## By workflow phase

A typical Android testing workflow runs through five phases. Skip phases at your peril.

### 1. Foundations — what to test, how to test it

Build the right mental model before touching any test framework.

- [fundamentals/concepts/understanding-the-testing-pyramid](fundamentals/concepts/understanding-the-testing-pyramid/SKILL.md)
- [fundamentals/concepts/choosing-what-to-test](fundamentals/concepts/choosing-what-to-test/SKILL.md)
- [fundamentals/doubles/picking-test-doubles](fundamentals/doubles/picking-test-doubles/SKILL.md)
- [fundamentals/strategies/applying-testing-strategies](fundamentals/strategies/applying-testing-strategies/SKILL.md)
- [fundamentals/strategies/organizing-test-source-sets](fundamentals/strategies/organizing-test-source-sets/SKILL.md)

### 2. JVM unit tests — fast feedback loop

Run on the JVM. Mock dependencies. Use Robolectric only when Android stubs are unavoidable.

- [kotlin/kotlin-test/writing-tests-with-kotlin-test](kotlin/kotlin-test/writing-tests-with-kotlin-test/SKILL.md) — the `kotlin.test` assertion vocabulary used inside all of the below (and in `commonTest` for multiplatform modules)
- [jvm-tests/runner/configuring-junit4-on-android](jvm-tests/runner/configuring-junit4-on-android/SKILL.md)
- [jvm-tests/mocking/mocking-with-mockito](jvm-tests/mocking/mocking-with-mockito/SKILL.md)
- [jvm-tests/mocking/mocking-with-mockk](jvm-tests/mocking/mocking-with-mockk/SKILL.md)
- [jvm-tests/coroutines/testing-coroutines-with-runtest](jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md)
- [jvm-tests/coroutines/testing-flows-with-turbine](jvm-tests/coroutines/testing-flows-with-turbine/SKILL.md)
- [jvm-tests/robolectric/using-robolectric-correctly](jvm-tests/robolectric/using-robolectric-correctly/SKILL.md)

### 3. Instrumentation — integration and UI on a real device

Run on an emulator or physical device. Cover the View / Fragment / Activity stack and cross-app flows.

- [instrumentation/runner/running-instrumented-tests-with-androidjunit4](instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md)
- [instrumentation/scenarios/launching-activities-with-activityscenario](instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md)
- [instrumentation/scenarios/launching-fragments-with-fragmentscenario](instrumentation/scenarios/launching-fragments-with-fragmentscenario/SKILL.md)
- [instrumentation/espresso/writing-espresso-tests](instrumentation/espresso/writing-espresso-tests/SKILL.md)
- [instrumentation/uiautomator/cross-app-tests-with-uiautomator](instrumentation/uiautomator/cross-app-tests-with-uiautomator/SKILL.md)
- [instrumentation/managed-devices/running-tests-on-gradle-managed-devices](instrumentation/managed-devices/running-tests-on-gradle-managed-devices/SKILL.md) — run the above on emulators Gradle provisions/boots/tears down; reproducible in CI
- [platform/legacy/migrating-from-android-test-classes](platform/legacy/migrating-from-android-test-classes/SKILL.md) — if the codebase still has `android.test.*` (`ActivityInstrumentationTestCase2`, `MoreAsserts`, …), migrate it onto the above first

### 4. Compose UI tests — semantics-driven UI assertions

The richest set in this repo. Setup, finders, assertions, actions, sync, patterns, interop, debug, audit.

- Setup: [compose/setup/configuring-test-dependencies](compose/setup/configuring-test-dependencies/SKILL.md), [compose/setup/choosing-test-rule-vs-runtest](compose/setup/choosing-test-rule-vs-runtest/SKILL.md), [compose/setup/setting-up-host-vs-device-tests](compose/setup/setting-up-host-vs-device-tests/SKILL.md)
- Find: [compose/finders/finding-nodes-by-tag-text-content](compose/finders/finding-nodes-by-tag-text-content/SKILL.md), [compose/finders/composing-semantics-matchers](compose/finders/composing-semantics-matchers/SKILL.md), [compose/finders/traversing-the-semantics-tree](compose/finders/traversing-the-semantics-tree/SKILL.md)
- Assert: [compose/assertions/asserting-node-state-and-text](compose/assertions/asserting-node-state-and-text/SKILL.md), [compose/assertions/asserting-bounds-and-dimensions](compose/assertions/asserting-bounds-and-dimensions/SKILL.md)
- Act: [compose/actions/clicking-and-scrolling](compose/actions/clicking-and-scrolling/SKILL.md), [compose/actions/injecting-touch-gestures](compose/actions/injecting-touch-gestures/SKILL.md), [compose/actions/injecting-mouse-and-keyboard](compose/actions/injecting-mouse-and-keyboard/SKILL.md), [compose/actions/entering-text](compose/actions/entering-text/SKILL.md)
- Synchronize: [compose/synchronization/controlling-the-test-clock](compose/synchronization/controlling-the-test-clock/SKILL.md), [compose/synchronization/testing-animations-deterministically](compose/synchronization/testing-animations-deterministically/SKILL.md), [compose/synchronization/synchronizing-with-idle](compose/synchronization/synchronizing-with-idle/SKILL.md)
- Patterns: [compose/patterns/structuring-a-compose-test](compose/patterns/structuring-a-compose-test/SKILL.md), [compose/patterns/testing-lazy-lists](compose/patterns/testing-lazy-lists/SKILL.md), [compose/patterns/testing-state-restoration](compose/patterns/testing-state-restoration/SKILL.md)
- Interop: [compose/interop/testing-with-espresso-interop](compose/interop/testing-with-espresso-interop/SKILL.md)
- Debug: [compose/debug/printing-the-semantics-tree](compose/debug/printing-the-semantics-tree/SKILL.md), [compose/debug/enabling-accessibility-checks](compose/debug/enabling-accessibility-checks/SKILL.md)
- Audit: [compose/audit/auditing-compose-test-suite](compose/audit/auditing-compose-test-suite/SKILL.md)
- Preview: [compose/preview/developing-with-compose-previews](compose/preview/developing-with-compose-previews/SKILL.md), [compose/preview/capturing-preview-screenshots-in-ci](compose/preview/capturing-preview-screenshots-in-ci/SKILL.md)
- Stability gate: [compose/stability/validating-compose-stability](compose/stability/validating-compose-stability/SKILL.md)

### 5. ADB — driving the device, capturing artifacts, automating CI

Below the test framework. Used for E2E flows, multi-device matrices, and CI orchestration.

- Architecture: [adb/architecture/understanding-adb-architecture](adb/architecture/understanding-adb-architecture/SKILL.md)
- Connect: [adb/devices/connecting-to-devices](adb/devices/connecting-to-devices/SKILL.md), [adb/devices/connecting-over-wifi](adb/devices/connecting-over-wifi/SKILL.md)
- Manage apps: [adb/apps/installing-and-managing-apps](adb/apps/installing-and-managing-apps/SKILL.md)
- Run tests: [adb/tests/running-instrumented-tests-via-adb](adb/tests/running-instrumented-tests-via-adb/SKILL.md)
- Drive UI: [adb/control/injecting-input-and-state](adb/control/injecting-input-and-state/SKILL.md)
- Capture: [adb/capture/capturing-screenshots-and-screenrecord](adb/capture/capturing-screenshots-and-screenrecord/SKILL.md), [adb/observability/extracting-logs-with-logcat](adb/observability/extracting-logs-with-logcat/SKILL.md), [adb/transfer/extracting-test-artifacts](adb/transfer/extracting-test-artifacts/SKILL.md)
- Automate: [adb/automation/scripting-adb-for-ci](adb/automation/scripting-adb-for-ci/SKILL.md)
