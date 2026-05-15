---
name: applying-testing-strategies
description: Use this skill to apply Android-team testing strategies — determinism, hermetic execution, Given-When-Then / Arrange-Act-Assert structure, naming conventions, and Hilt-based dependency replacement. Encodes which guidance actually lives on `/training/testing/fundamentals/strategies` (qualitative pyramid, network-access table) versus `/training/testing/instrumented-tests/stability` (determinism / flake) versus `/training/dependency-injection/hilt-testing` (`@HiltAndroidTest`, `HiltAndroidRule`, `@TestInstallIn`, `@UninstallModules`, `@BindValue`, rule ordering). Use when the user asks "how should I structure tests", "Given When Then or Arrange Act Assert", "how do I name test methods", "Hilt rule order", "@HiltAndroidTest", "@BindValue", "how to swap a binding for tests", "tests pass locally fail on CI", or "make tests deterministic".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - test-strategy
  - determinism
  - flaky-tests
  - given-when-then
  - arrange-act-assert
  - hilt-testing
  - HiltAndroidRule
  - TestInstallIn
  - test-naming
---

# Applying Testing Strategies — Determinism, Structure, and Hilt

A test suite needs three things to scale: it must be *deterministic* (same input → same outcome), *structured* (any reader can locate Given/When/Then), and *replaceable* (a single binding swap reroutes the production graph to fakes). This skill encodes Google's strategy guidance — flagging which page each rule actually lives on — and the Hilt mechanics that wire it into the build.

## When to use this skill

- The user asks "how should I structure my Android tests" / "Given-When-Then or Arrange-Act-Assert" / "how do I name test methods".
- The user asks about `@HiltAndroidTest`, `HiltAndroidRule`, `@TestInstallIn`, `@UninstallModules`, `@BindValue`, or rule ordering.
- The user reports "tests pass locally but fail on CI" — usually a determinism / hermeticity issue.
- The user is starting a new module and needs a strategy doc / convention list.
- The user mentions `RuleChain.outerRule` for combining `HiltAndroidRule` with `ActivityScenarioRule` or `composeTestRule`.

## When NOT to use this skill

- The user is choosing scope (small/medium/big) — use `../../concepts/understanding-the-testing-pyramid/SKILL.md`.
- The user is choosing what behavior to cover — use `../../concepts/choosing-what-to-test/SKILL.md`.
- The user is choosing fakes vs mocks — use `../../doubles/picking-test-doubles/SKILL.md`.
- The user is wiring source sets — use `../organizing-test-source-sets/SKILL.md`.
- The user is debugging a single flaky Compose animation test — use `../../../compose/synchronization/synchronizing-with-idle/SKILL.md`.

## Prerequisites

- Familiarity with the small/medium/big pyramid framing from `../../concepts/understanding-the-testing-pyramid/SKILL.md`.
- A test target where dependencies are injected (Hilt, manual constructor injection, or another DI library).
- For the Hilt section: `dagger.hilt.android.testing.HiltAndroidRule` and `dagger.hilt.android.testing.HiltAndroidTest` available on `androidTestImplementation`.
- For determinism: `kotlinx-coroutines-test`, `Robolectric` (if host tests), and the ability to stub `System.currentTimeMillis()` / `Clock.systemDefaultZone()`.

## Source-of-truth map (which page says what)

Per `tasks/research/R8-android-fundamentals.md`, the strategy concepts live on three different pages. Naming the source matters because PR reviewers cite skills.

| Concept | Page that actually documents it |
|---|---|
| Qualitative pyramid ("many small, few big") | `/training/testing/fundamentals/strategies` |
| Five-level Unit/Component/Feature/Application/RC table | `/training/testing/fundamentals/strategies` |
| Network access per layer | `/training/testing/fundamentals/strategies` |
| Determinism / flake handling | `/training/testing/instrumented-tests/stability` (NOT /strategies) |
| Hermetic test definition | `/training/testing/fundamentals/test-doubles` (in passing) |
| Given-When-Then comment style | NEVER named on any page; only used in code samples |
| Arrange-Act-Assert | NEVER named on any page |
| Test naming | Only the backtick caveat on `/training/testing/instrumented-tests` |
| `@HiltAndroidTest`, `HiltAndroidRule`, `@TestInstallIn`, `@UninstallModules`, `@BindValue` | `/training/dependency-injection/hilt-testing` |

**MUST NOT** attribute determinism guidance to `/strategies` — it is on `/instrumented-tests/stability`. **MUST NOT** claim Google "recommends Given-When-Then" — the pages do not say that; they only *use* the comment style in samples.

## Determinism

Per `/training/testing/instrumented-tests/stability` (and CORPUS §C "Common gotchas"), four sources of non-determinism dominate:

1. **Time** — `System.currentTimeMillis()`, `Clock.systemDefaultZone()`, `Instant.now()`. Inject a `Clock` or use `kotlinx-coroutines-test`'s virtual time.
2. **Concurrency / coroutines** — direct `Dispatchers.IO`, `runBlocking`, real `delay`. Use `runTest`, inject `CoroutineDispatcher`, replace `Dispatchers.Main` via `Dispatchers.setMain(dispatcher)`. See `../../../jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md`.
3. **Animation / frame timing** — Compose's `MainTestClock`, View animations. Disable via `testOptions.animationsDisabled = true` (Gradle) or `Settings.Global.WINDOW_ANIMATION_SCALE = 0` (ADB). For Compose: `mainClock.autoAdvance = false` per the skydoves directives in `docs/SPEC.md` §5.
4. **External state** — network, real DB, `Random()`, `UUID.randomUUID()`. Inject seeds, use fakes, run `pm clear` before each instrumented test.

Concrete checklist:

- [ ] `Clock` (or equivalent time source) is injected; tests substitute `Clock.fixed(Instant.parse(...), ZoneId.of("UTC"))`.
- [ ] `Dispatchers` are injected via a `DispatcherProvider` interface or replaced with `Dispatchers.setMain(StandardTestDispatcher())` in a `@Before`.
- [ ] `Random` is seeded (`Random(42)`) or wrapped behind a `RandomProvider` that the test substitutes.
- [ ] `testOptions.animationsDisabled = true` is set in `build.gradle.kts`.
- [ ] CI runs `pm clear <pkg>` before every instrumented test (Test Orchestrator does this automatically; otherwise add an `@Before`).

## Hermetic vs integration

The `/strategies` page does NOT formalize "hermetic" as a top-level term. The closest definition is on `/test-doubles`:

> "A hermetic test avoids all external dependencies, such as fetching data from the internet."
> — `developer.android.com/training/testing/fundamentals/test-doubles`

The operational rule is on `/strategies`:

| Layer | Network access |
|---|---|
| Unit | None |
| Component | None |
| Feature | "supports mocked network access" |
| Application | n/a |
| Release Candidate | n/a |

**MUST NOT** present "hermetic" as if it were a dedicated section on `/strategies` — it is mentioned in passing on `/test-doubles` only. **MUST** cite both pages when discussing hermeticity: `/test-doubles` for the term, `/strategies` for the operational rule.

## Given-When-Then / Arrange-Act-Assert

Neither pattern is *named* on any of the four primary pages (`/fundamentals`, `/what-to-test`, `/test-doubles`, `/strategies`). They *are* visibly used in Google's code samples. From `/fundamentals`:

```kotlin
// Given an instance of MyViewModel
val viewModel = MyViewModel(myFakeDataRepository)

// When data is loaded
viewModel.loadData()

// Then it should be exposing data
assertTrue(viewModel.data != null)
```
— `developer.android.com/training/testing/fundamentals` (Local/host-side unit test example)

And from the same page:

```kotlin
// When the Continue button is clicked
onView(withText("Continue")).perform(click())
// Then the Welcome screen is displayed
onView(withText("Welcome")).check(matches(isDisplayed()))
```
— `developer.android.com/training/testing/fundamentals` (Espresso example)

Cite "Google's official samples consistently use Given-When-Then comment blocks" — accurate. Do NOT cite "Google recommends Given-When-Then" — that is fabrication.

Both Given-When-Then and Arrange-Act-Assert are the same shape with different vocabulary:

```
Given (Arrange) — set up the SUT and dependencies
When  (Act)     — invoke the behavior under test
Then  (Assert)  — assert on the resulting state or interaction
```

Pick one vocabulary per repo (skydoves preference: Given-When-Then, matching Google's samples). Use it as inline comment blocks, not as method names.

## Test naming

The only verbatim naming guidance across the four primary pages is the backtick caveat on `/instrumented-tests`:

> "**Note:** Using backticks to name tests in Kotlin is only supported on devices running API 30 and above."
> — `developer.android.com/training/testing/instrumented-tests`

Implications:

- **Local tests** (`src/test/`) — backtick names are safe (run on JVM, not the Android runtime).
- **Instrumented tests** (`src/androidTest/`) — backtick names require `minSdk >= 30` on the test APK. Below that, use `camelCase` or `snake_case_with_underscores`.

skydoves convention (combine these two facts): a `methodName_state_expected` shape that reads as a sentence.

```
loadUsers_emptyList_emitsEmptyState
applyCoupon_negativeTotal_emitsError
save_orchestratorMarksDirtyOnce
```

## Hilt for testing — the full mechanic

Source: `developer.android.com/training/dependency-injection/hilt-testing` plus CORPUS §F.6.

### Why Hilt for tests at all

> "Hilt isn't necessary for unit tests, since when testing a class that uses constructor injection, you don't need to use Hilt to instantiate that class."
> — `developer.android.com/training/dependency-injection/hilt-testing`

> "For integration tests, Hilt injects dependencies as it would in your production code. Testing with Hilt requires no maintenance because Hilt automatically generates a new set of components for each test."
> — `developer.android.com/training/dependency-injection/hilt-testing`

So: **constructor-inject in unit tests** (skip Hilt), **use Hilt in UI / integration tests**.

### `@HiltAndroidTest` + `HiltAndroidRule` ordered first

```kotlin
@HiltAndroidTest
class HomeScreenTest {
    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) val composeRule = createAndroidComposeRule<MainActivity>()

    @Inject lateinit var userRepository: UserRepository

    @Before fun inject() = hiltRule.inject()
}
```

`HiltAndroidRule` MUST execute **before** any rule that touches the injected graph (Activity launch, Compose setup). The `/hilt-testing` page phrases this as "the `HiltAndroidRule` executes first". Lower `order` values run first in JUnit4 (`@Rule(order = 0)` outranks `order = 1`), so the conventional pattern is `0` then `1` as shown — but the rule is "Hilt first", not specifically "order = 0". Negative numbers also work (`order = -1` paired with `order = 0`), and any monotonically increasing scheme is fine. Per `/hilt-testing`:

> "You must annotate any UI test that uses Hilt with `@HiltAndroidTest`. This annotation is responsible for generating the Hilt components for each test. Also, you need to add the `HiltAndroidRule` to the test class."
> — `developer.android.com/training/dependency-injection/hilt-testing`

> "To inject types into a test, use `@Inject` for field injection. To tell Hilt to populate the `@Inject` fields, call `hiltRule.inject()`."
> — `developer.android.com/training/dependency-injection/hilt-testing`

### `@TestInstallIn` — replace a binding for ALL tests

```kotlin
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [AnalyticsModule::class],
)
abstract class FakeAnalyticsModule {
    @Singleton @Binds
    abstract fun bindAnalyticsService(impl: FakeAnalyticsService): AnalyticsService
}
```

The replacement applies to every `@HiltAndroidTest` in the module. Use this when the same fake fits every test (in-memory DB, no-op analytics).

### `@UninstallModules` — replace in a SINGLE test

```kotlin
@UninstallModules(AnalyticsModule::class)
@HiltAndroidTest
class SettingsActivityTest {
    @Module @InstallIn(SingletonComponent::class)
    abstract class TestModule {
        @Singleton @Binds
        abstract fun bindAnalyticsService(impl: FakeAnalyticsService): AnalyticsService
    }
}
```

Per `/hilt-testing` warnings (must surface in skills):

> "**Warning:** You cannot uninstall modules that are not annotated with `@InstallIn`. Attempting to do so causes a compilation error."

> "**Warning:** `@UninstallModules` can only uninstall `@InstallIn` modules, not `@TestInstallIn` modules."
> — `developer.android.com/training/dependency-injection/hilt-testing`

### `@BindValue` — quick swap for a single test

```kotlin
@UninstallModules(AnalyticsModule::class)
@HiltAndroidTest
class SettingsActivityTest {
    @BindValue @JvmField
    val analyticsService: AnalyticsService = FakeAnalyticsService()
}
```

`@BindValue` is the lightweight form — no test module class. Use when the test wants to seed a single instance and assert on it.

### `@CustomTestApplication` — non-Hilt base class

When the production `Application` extends a non-Hilt base (`MultiDexApplication`, a vendor app class), Hilt cannot generate the test app automatically. Use:

```kotlin
@CustomTestApplication(BaseApplication::class)
interface HiltTestApplication
```

The generated `HiltTestApplication_Application` is then named in `testInstrumentationRunnerArguments` or via a custom `AndroidJUnitRunner`.

### Rule ordering with multiple rules

```kotlin
@HiltAndroidTest
class SettingsActivityTest {
    @get:Rule(order = 0) var hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) var settingsActivityTestRule = SettingsActivityTestRule(...)
}
```
— `developer.android.com/training/dependency-injection/hilt-testing`

Or with `RuleChain` for mid-chain composition:

```kotlin
@get:Rule
val chain: TestRule = RuleChain
    .outerRule(HiltAndroidRule(this))
    .around(InstantTaskExecutorRule())
    .around(MainDispatcherRule())
```

## Patterns

### Pattern: WRONG vs RIGHT — Given-When-Then comment block

```kotlin
// WRONG — no structure; reader cannot find the act vs assert boundary
@Test
fun loadsUser() = runTest {
    val repo = FakeUserRepository().apply { seed(User(UserId("u1"), "Alice")) }
    val vm = UserViewModel(repo)
    vm.load(UserId("u1"))
    val state = vm.state.value
    assertEquals("Alice", state.name)
    assertFalse(state.isLoading)
}
// WRONG because: the test crams setup, action, and assertion together. With three or
// more lines per stage, readers cannot locate failures fast. Google's own samples use
// Given-When-Then comments as visual separators.
```

```kotlin
// RIGHT
@Test
fun loadUser_seededId_emitsLoadedState() = runTest {
    // Given a fake repo seeded with Alice
    val repo = FakeUserRepository().apply { seed(User(UserId("u1"), "Alice")) }
    val vm = UserViewModel(repo)

    // When loading the seeded id
    vm.load(UserId("u1"))

    // Then the state is Loaded with Alice
    assertEquals(UserUiState.Loaded(name = "Alice"), vm.state.value)
}
```

### Pattern: WRONG vs RIGHT — Hilt rule order

```kotlin
// WRONG
@HiltAndroidTest
class HomeScreenTest {
    @get:Rule val composeRule = createAndroidComposeRule<MainActivity>()  // order = 0 by default
    @get:Rule val hiltRule = HiltAndroidRule(this)
}
// WRONG because: with no order, JUnit picks ordering implementation-defined. The
// Activity may launch before Hilt's component is built, throwing at @Inject sites.
// Per /hilt-testing, HiltAndroidRule must run first.
```

```kotlin
// RIGHT
@HiltAndroidTest
class HomeScreenTest {
    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @get:Rule(order = 1) val composeRule = createAndroidComposeRule<MainActivity>()
}
```

### Pattern: WRONG vs RIGHT — non-deterministic time

```kotlin
// WRONG
class DueDateTest {
    @Test fun reportsOverdue() {
        val task = Task(dueAt = Instant.now().minusSeconds(60))
        assertTrue(task.isOverdue())   // depends on real wall clock; flake risk on slow CI
    }
}
```

```kotlin
// RIGHT
class DueDateTest {
    private val fixedNow = Instant.parse("2024-01-01T00:00:00Z")
    private val clock = Clock.fixed(fixedNow, ZoneId.of("UTC"))

    @Test fun reportsOverdue() {
        val task = Task(dueAt = fixedNow.minusSeconds(60), clock = clock)
        assertTrue(task.isOverdue())
    }
}
```

### Pattern: WRONG vs RIGHT — `@BindValue` over `every { }`

```kotlin
// WRONG — mocking a Hilt-injected real instance
@HiltAndroidTest
class SettingsActivityTest {
    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
    @Inject lateinit var analytics: AnalyticsService

    @Test fun tracks() {
        // analytics is the real production instance; tests now mutate prod state
        every { (analytics as RealAnalyticsService).enabled = false }   // does not compile
    }
}
```

```kotlin
// RIGHT — replace at the binding boundary
@UninstallModules(AnalyticsModule::class)
@HiltAndroidTest
class SettingsActivityTest {
    @BindValue @JvmField
    val analytics: AnalyticsService = FakeAnalyticsService()

    @get:Rule(order = 0) val hiltRule = HiltAndroidRule(this)
}
```

## Mandatory rules

- **MUST** order `HiltAndroidRule` to execute **first** when combined with any rule that touches the injected graph (Activity, Compose, Fragment, Coroutine). The conventional shape is `@Rule(order = 0)` on Hilt and `@Rule(order = 1)` on the next rule, but any monotonic ordering that puts Hilt first is fine — `/hilt-testing` says "the `HiltAndroidRule` executes first", not specifically `order = 0`.
- **MUST** call `hiltRule.inject()` in `@Before` for any test class with `@Inject` fields.
- **MUST NOT** annotate a `@TestInstallIn`-replaced module with `@UninstallModules` — Hilt fails compilation per the page's warning box.
- **MUST NOT** attribute determinism guidance to `/training/testing/fundamentals/strategies`. Use `/training/testing/instrumented-tests/stability`.
- **MUST NOT** claim "Google recommends Given-When-Then". The pages use it in samples; they do not prescribe it.
- **MUST** inject a `Clock` (or equivalent time source), `DispatcherProvider`, and `Random` seed for any code whose output depends on them.
- **MUST** name local tests with backticks freely; for instrumented tests that may run on devices below API 30, use `camelCase_state_expected` (per `/instrumented-tests`: "Using backticks to name tests in Kotlin is only supported on devices running API 30 and above" — the gate is the **runtime device API**, not just the module's `minSdk`).
- **PREFERRED:** Given-When-Then over Arrange-Act-Assert, mirroring the vocabulary in Google's own code samples.
- **PREFERRED:** `@BindValue` for single-test binding swaps, `@TestInstallIn` for module-wide swaps.
- **PREFERRED:** disable animations via `testOptions.animationsDisabled = true` rather than per-test ADB shell calls.

## Verification

- [ ] Every `@HiltAndroidTest` class declares `HiltAndroidRule` with an `order` that places it first relative to any other JUnit rules in the class (`@Rule(order = 0)` is the conventional choice).
- [ ] Every `@HiltAndroidTest` class with `@Inject` fields calls `hiltRule.inject()` in `@Before`.
- [ ] No production code references `Fake*` classes; all fakes live under `src/androidTest/` or `src/test/` (or a dedicated `core-testing` module).
- [ ] No test method directly reads `Instant.now()`, `System.currentTimeMillis()`, or `Random()` without a seed/injection.
- [ ] `testOptions.animationsDisabled = true` set in module `build.gradle.kts`.
- [ ] CI runs identically to local — no test relies on network, real time, or ambient device state.
- [ ] All tests in the module follow one structural convention (Given-When-Then OR Arrange-Act-Assert; not both).

## References

- `developer.android.com/training/testing/fundamentals/strategies` — qualitative pyramid, network-access table per layer.
- `developer.android.com/training/testing/fundamentals/test-doubles` — hermetic-test definition (in passing).
- `developer.android.com/training/testing/instrumented-tests/stability` — determinism / flake guidance (NOT on /strategies).
- `developer.android.com/training/testing/instrumented-tests` — backtick naming caveat ("Using backticks ... only supported on devices running API 30 and above").
- `developer.android.com/training/dependency-injection/hilt-testing` — `@HiltAndroidTest`, `HiltAndroidRule`, `@TestInstallIn`, `@UninstallModules`, `@BindValue`, `@CustomTestApplication`, the warning boxes, the rule-ordering example.
- `tasks/research/R8-android-fundamentals.md` — verbatim Hilt quotes and the source-of-truth map per page.
- CORPUS §G.3 — `MainDispatcherRule` JUnit4 wrapper at `testutils/testutils-ktx/src/jvmMain/kotlin/androidx/testutils/MainDispatcherRule.jvm.kt`.
- Sibling skills: `../../concepts/understanding-the-testing-pyramid/SKILL.md`, `../../concepts/choosing-what-to-test/SKILL.md`, `../../doubles/picking-test-doubles/SKILL.md`, `../organizing-test-source-sets/SKILL.md`.
- Cross-category: `../../../jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md`, `../../../jvm-tests/coroutines/testing-flows-with-turbine/SKILL.md`, `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`, `../../../jvm-tests/robolectric/using-robolectric-correctly/SKILL.md`, `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`, `../../../instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md`, `../../../adb/observability/extracting-logs-with-logcat/SKILL.md`.
