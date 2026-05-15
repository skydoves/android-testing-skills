---
name: mocking-with-mockito
description: Use this skill to wire Mockito (the dominant Android mocking framework, exclusively used by androidx itself) into a JVM unit-test suite. Covers `org.mockito:mockito-core:5.x` plus the `org.mockito.kotlin:mockito-kotlin:5.x` Kotlin DSL — `mock<T>()`, `whenever`, `argumentCaptor<T>()`, `any()` / `anyOrNull()` / `eq()`, `MockitoJUnit.rule()` vs `MockitoAnnotations.openMocks(this)` lifecycle, the inline mock-maker for Kotlin final classes (`mock-maker-inline` plugin file), `@SdkSuppress(minSdkVersion = 28)` for instrumented final-class mocks, and the suspend-stubbing escape hatch via `runBlocking`. Use when the user mentions `whenever`, `Cannot mock final class`, `InvalidUseOfMatchersException`, `@Mock null`, `MockitoJUnitRunner`, `argumentCaptor`, `mock-maker-inline`, or asks how to mock with mockito-kotlin.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - mockito
  - mockito-kotlin
  - jvm-unit-tests
  - mock-maker-inline
  - argumentCaptor
  - whenever
  - MockitoJUnit-rule
  - final-class-mocking
  - openMocks
---

# Mocking with Mockito — The androidx-Native Mocking Stack

Mockito is the mocking framework androidx itself uses — 400+ test files import `org.mockito` across the AOSP checkout, while `io.mockk` returns zero hits. New Android code that wants to "match what Google does" picks Mockito + the `mockito-kotlin` Kotlin DSL. This skill documents the exact dependency matrix, the three setup styles, the Kotlin DSL surface, and the inline-mock-maker plumbing required for Kotlin final classes.

## When to use this skill

- The user is starting a new test suite in a Kotlin Android module and asks "what mocking library should I use".
- The user reports `Cannot mock/spy class … final class` from Mockito 4.x and needs the `mock-maker-inline` configuration.
- The user reports `InvalidUseOfMatchersException` from mixing raw values with `any()` / `eq()`.
- The user is wrestling with the `when` keyword clash in Kotlin (`` `when`(mock.foo()) ``) and wants the `whenever` DSL.
- The user wants `argumentCaptor<T>()` with `firstValue` / `lastValue` / `allValues` instead of `ArgumentCaptor.forClass(...)`.
- The user is migrating from `MockitoAnnotations.initMocks(this)` to `openMocks(this)` or the `MockitoJUnit.rule()` approach.
- The user is mocking a `final` class on an instrumented test on API < 28.

## When NOT to use this skill

- The user wants `coEvery` / `coVerify` for suspend functions, deep singleton mocking via `mockkObject`, or static mocking via `mockkStatic` — use `../mocking-with-mockk/SKILL.md`.
- The user is choosing between fakes and mocks at the design level — use `../../../fundamentals/doubles/picking-test-doubles/SKILL.md`. Per /test-doubles, fakes are preferred over mocks for behaviour-heavy collaborators.
- The runner / Gradle matrix isn't set up yet — start with `../../runner/configuring-junit4-on-android/SKILL.md`.
- The user wants `runTest`-aware testing for coroutines — see `../../coroutines/testing-coroutines-with-runtest/SKILL.md`.

## Prerequisites

- The base test wiring from `../../runner/configuring-junit4-on-android/SKILL.md` is already in place (`@RunWith(AndroidJUnit4::class)`, `androidx.test.ext:junit:1.3.0`, JUnit 4.13.2).
- A Kotlin module on JDK 11+ (Mockito 5.x raised the JDK floor to Java 11).
- A `MainDispatcherRule` is installed if any code-under-test touches `Dispatchers.Main` — see the runner skill.

## Workflow

- [ ] **1. Add the Mockito + mockito-kotlin coordinates.** Mockito 5.x is the recommended baseline because the inline mock-maker is enabled by default — no plugin file or `mockito-inline` swap needed for Kotlin final classes:

```kotlin
dependencies {
    testImplementation("org.mockito:mockito-core:5.14.2")
    testImplementation("org.mockito.kotlin:mockito-kotlin:5.4.0")

    // Instrumented final-class mocks (API 28+ only — see step 7)
    androidTestImplementation("org.mockito:mockito-android:5.14.2")
    androidTestImplementation("org.mockito.kotlin:mockito-kotlin:5.4.0")
}
```

`mockito-kotlin` is NOT a fork — it is a thin DSL helper layer that delegates straight to `mockito-core`. Both must be on the classpath.

- [ ] **2. Pick a mock-lifecycle style.** There are three idiomatic options. Prefer the rule for new code; reserve `openMocks` for cases where the test class can't claim `@RunWith`.

```kotlin
// (a) JUnit4 rule — preferred. No runner ownership, plays nicely with @RunWith(AndroidJUnit4::class).
@get:Rule val mockitoRule: MockitoRule = MockitoJUnit.rule()

// (b) Manual openMocks — works everywhere.
private lateinit var closeable: AutoCloseable
@Before fun setUp() { closeable = MockitoAnnotations.openMocks(this) }
@After  fun tearDown() { closeable.close() }

// (c) JUnit5 extension (only if the project is on JUnit 5 via the Vintage engine).
@ExtendWith(MockitoExtension::class)
```

The deprecated `MockitoAnnotations.initMocks(this)` returns `void` and leaks inline-mock resources. Always use `openMocks(this)` and close the returned `AutoCloseable` in `@After`.

- [ ] **3. Declare mocks and stub them with the mockito-kotlin DSL.** The Kotlin-friendly entry points live under `org.mockito.kotlin.*`:

```kotlin
import org.mockito.kotlin.any
import org.mockito.kotlin.argumentCaptor
import org.mockito.kotlin.atLeastOnce
import org.mockito.kotlin.doReturn
import org.mockito.kotlin.eq
import org.mockito.kotlin.mock
import org.mockito.kotlin.verify
import org.mockito.kotlin.whenever

class UserViewModelTest {
    @get:Rule val mockitoRule = MockitoJUnit.rule()
    @get:Rule val mainDispatcherRule = MainDispatcherRule()

    private val repo: UserRepository = mock()
    private val analytics: Analytics = mock {
        on { isEnabled() } doReturn true
    }

    @Test fun loadsUserAndLogs() = runTest {
        whenever(repo.findById(7L)) doReturn User("Jane")

        val vm = UserViewModel(repo, analytics)
        vm.load(7L)
        advanceUntilIdle()

        val captor = argumentCaptor<AnalyticsEvent>()
        verify(analytics, atLeastOnce()).log(captor.capture())
        assertThat(captor.lastValue.name).isEqualTo("user_loaded")
        verify(repo).findById(eq(7L))
    }
}
```

Highlights:
- `mock<T>()` is reified — no `T::class.java` ceremony.
- `mock { on { … } doReturn … }` is a builder block — collapses a chain of `whenever` calls when stubbing at construction time.
- `whenever(...)` replaces raw Mockito's `` `when`(...) `` (which requires backticks because `when` is a Kotlin keyword).
- `argumentCaptor<T>()` returns a captor with `firstValue` / `secondValue` / `lastValue` / `allValues` properties — no `getValue()` ceremony.
- `any()`, `anyOrNull()`, `eq()` from `org.mockito.kotlin.*` are null-safe in Kotlin (raw Mockito's `any()` returns Java `null`, which crashes a Kotlin non-nullable parameter).

- [ ] **4. Honour the any/eq matcher mixing rule.** If ANY argument uses a matcher (`any()`, `eq()`, `argThat { … }`), ALL arguments must be matchers. Mixing raw values with `any()` raises `InvalidUseOfMatchersException: Misplaced or misused argument matcher`. Wrap raw values in `eq(...)`:

```kotlin
verify(repo).save(any(), eq("identifier"))   // RIGHT — all args are matchers
verify(repo).save(any(), "identifier")        // WRONG — InvalidUseOfMatchersException
```

- [ ] **5. Stub suspend functions from any suspend context.** Mockito does not have first-class suspend support. The compiler accepts `whenever(mock.suspendFn())` only when the call site is itself suspending. **`runTest { … }` already provides that suspend context — wrapping a stub in `runBlocking { … }` inside `runTest { … }` blocks the test dispatcher and can deadlock under `MainDispatcherRule`. Pick one or the other, not both.** This is the chief ergonomic friction with Mockito; if the codebase is suspend-heavy, weigh `../mocking-with-mockk/SKILL.md` instead.

```kotlin
@Test fun loadsUser() = runTest {
    whenever(repo.fetchUser(1)) doReturn User("Jane")   // RIGHT — runTest IS a suspend context
    val vm = UserViewModel(repo)
    vm.load(1)
    advanceUntilIdle()
    verify(repo).fetchUser(1)
}
```

For non-suspending tests (no `runTest`), use `runBlocking` instead — but never both:

```kotlin
@Test fun loadsUser() = runBlocking {
    whenever(repo.fetchUser(1)) doReturn User("Jane")
    val result = UserService(repo).get(1)
    assertThat(result.name).isEqualTo("Jane")
}
```

- [ ] **6. For Mockito 4.x, install the inline mock-maker via the resource file.** Mockito 5.x already enables it by default. For older versions, drop a single-line file at `src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker` containing:

```text
mock-maker-inline
```

This is exactly what androidx ships in `compose/material/material/src/androidHostTest/resources/mockito-extensions/org.mockito.plugins.MockMaker`, `compose/foundation/foundation/.../mockito-extensions/...`, and `room3/room3-runtime/.../mockito-extensions/...`. The alternative is swapping `mockito-core` for `mockito-inline` in Gradle (which transitively pulls `mockito-core`).

Inline still has limits: `final` private members, `equals()`/`hashCode()`, `String`, and `Class` cannot be mocked.

- [ ] **7. For instrumented tests on API < 28, guard final-class mocks.** `mockito-android` uses dexmaker, which only supports final-class mocking on **API 28+**. This is a **dexmaker constraint, not a Mockito version constraint** — upgrading to Mockito 5.x does not lift it; only Android API 28 (P) introduced the JVMTI hooks dexmaker needs. The androidx convention is to add `@SdkSuppress(minSdkVersion = 28)` to the test class with a one-line comment. Real example from `compose/ui/ui/src/androidHostTest/kotlin/androidx/compose/ui/input/IndirectPointerEventWithInputDeviceMockTest.kt`:

```kotlin
import androidx.test.filters.SdkSuppress

@RunWith(MockitoJUnitRunner::class)
@SdkSuppress(minSdkVersion = 28)   // Mocks for final classes can only be done on 28 and higher
class IndirectPointerEventWithInputDeviceMockTest { /* ... */ }
```

- [ ] **8. Apply `Strictness.STRICT_STUBS` for new test classes.** Strict stubs surface unused stubs and argument mismatches as failures, catching tests that "pass" because they don't actually exercise the stub. Wire it via the rule:

```kotlin
@get:Rule val mockitoRule: MockitoRule =
    MockitoJUnit.rule().strictness(Strictness.STRICT_STUBS)
```

- [ ] **9. Verify the suite runs and produces the expected interactions.** `./gradlew :<module>:testDebugUnitTest --tests com.example.UserViewModelTest`. Failed verifications produce the canonical "Wanted but not invoked" / "Argument(s) are different" output — these messages are usually self-diagnosing.

## Patterns

### Pattern: WRONG vs RIGHT — Kotlin `when` keyword clash

```kotlin
// WRONG — raw Mockito from Kotlin
import org.mockito.Mockito.`when`

`when`(repo.fetch()).thenReturn(User("Jane"))
// WRONG because: `when` requires backtick-escaping in Kotlin (it is a reserved keyword),
// the import line is awkward, and any() returns Java null which crashes Kotlin
// non-nullable parameters. mockito-kotlin's whenever / null-safe matchers solve both.
```

```kotlin
// RIGHT — mockito-kotlin DSL
import org.mockito.kotlin.whenever
import org.mockito.kotlin.doReturn

whenever(repo.fetch()) doReturn User("Jane")
```

### Pattern: WRONG vs RIGHT — argument captor verbosity

```kotlin
// WRONG
import org.mockito.ArgumentCaptor

val captor = ArgumentCaptor.forClass(User::class.java)
verify(repo).save(captor.capture())
assertEquals("Jane", captor.value.name)
// WRONG because: ArgumentCaptor.forClass requires .java, captor.value collides with
// Kotlin's value class semantics for some types, and there's no firstValue/lastValue
// accessor for sequences of captures.
```

```kotlin
// RIGHT
import org.mockito.kotlin.argumentCaptor

val captor = argumentCaptor<User>()
verify(repo, atLeastOnce()).save(captor.capture())
assertEquals("Jane", captor.lastValue.name)
val all: List<User> = captor.allValues
```

### Pattern: WRONG vs RIGHT — final-class mocking on Mockito 4.x

```kotlin
// WRONG — Mockito 4.x without the inline plugin, mocking a Kotlin final class
val service: PaymentService = mock()
// WRONG because: Kotlin classes are final by default. The classic subclass mock-maker
// throws "Cannot mock/spy class com.example.PaymentService — final class". Either
// upgrade to Mockito 5.x (inline by default) or install the plugin file.
```

```text
// RIGHT — src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker
mock-maker-inline
```

```kotlin
// Mockito 5.x: nothing else to do; mock<PaymentService>() works out of the box.
```

### Pattern: WRONG vs RIGHT — strict stubs catch the dead stub

```kotlin
// WRONG — silent passing test
class MyTest {
    @get:Rule val mockitoRule = MockitoJUnit.rule()   // default lenient

    @Test fun loadsUser() {
        whenever(repo.findById(7L)) doReturn User("Jane")
        val vm = UserViewModel(repo)
        vm.load(8L)                                    // wrong id — stub is never used
        verify(repo).findById(any())                   // passes because findById was called
    }
}
// WRONG because: the test claims to verify behaviour for id=7 but actually exercises id=8.
// The stub is dead. STRICT_STUBS catches this with PotentialStubbingProblem.
```

```kotlin
// RIGHT
@get:Rule val mockitoRule: MockitoRule =
    MockitoJUnit.rule().strictness(Strictness.STRICT_STUBS)

// Now PotentialStubbingProblem fires at runtime if the stub argument doesn't match
// what the SUT actually calls.
```

## Mandatory rules

- **MUST** add both `org.mockito:mockito-core:5.x` AND `org.mockito.kotlin:mockito-kotlin:5.x` for Kotlin tests. The DSL artifact is non-optional in any modern Kotlin codebase.
- **MUST** use `whenever(...)` from `org.mockito.kotlin.*`, not `` `when`(...) `` from `org.mockito.Mockito`. The keyword-escape form is a code smell.
- **MUST** use `org.mockito.kotlin`'s `any()` / `anyOrNull()` / `eq()` for Kotlin tests. Raw Mockito's `any()` returns Java `null` and crashes Kotlin non-nullable parameters at runtime.
- **MUST** stub suspend functions from a suspending context — either `runTest { whenever(...) doReturn ... }` (preferred, integrates with virtual time) or `runBlocking { ... }` for non-`runTest` setups. **MUST NOT** nest `runBlocking` inside `runTest` — it blocks the test dispatcher and can deadlock under `MainDispatcherRule`. Mockito has no native suspend-stubbing surface; either suspend context is sufficient.
- **MUST** use `MockitoAnnotations.openMocks(this)` and close the returned `AutoCloseable` in `@After`. **MUST NOT** use the deprecated `MockitoAnnotations.initMocks(this)`.
- **MUST** ship `mockito-extensions/org.mockito.plugins.MockMaker` with literal contents `mock-maker-inline` for any pre-5.x project that mocks Kotlin final classes. Mockito 5.x ships inline by default and needs nothing.
- **MUST** annotate test classes that mock final classes with `@SdkSuppress(minSdkVersion = 28)` when they run as `androidTestImplementation` against `mockito-android`.
- **MUST** keep matchers consistent within a single `verify`/`whenever` call: all matchers OR no matchers. Wrap raw values in `eq(...)` to mix.
- **MUST** prefer fakes over mocks for collaborators with behaviour (per developer.android.com/training/testing/fundamentals/test-doubles: "fakes ... are preferred"). Use Mockito for verifying interactions, not for re-implementing behaviour.
- **MUST NOT** mix `Mockito.mock(...)` and `org.mockito.kotlin.mock<T>()` in the same file — pick one and be consistent. Prefer the kotlin DSL.
- **PREFERRED:** apply `Strictness.STRICT_STUBS` to catch dead stubs and argument mismatches. Lenient mode hides bugs.

## Verification

- [ ] `./gradlew :<module>:testDebugUnitTest --tests <YourTestClass>` passes.
- [ ] `grep -rn "org.mockito.Mockito.\`when\`" src/test src/androidTest` returns NO matches (use `whenever` instead).
- [ ] `grep -rn "MockitoAnnotations.initMocks" src/test src/androidTest` returns NO matches (use `openMocks` instead).
- [ ] `grep -rn "ArgumentCaptor.forClass" src/test src/androidTest` returns NO matches in new code (use `argumentCaptor<T>()` instead).
- [ ] If the project is on Mockito 4.x and mocks a Kotlin final class, the file `src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker` exists with literal contents `mock-maker-inline`.
- [ ] `MockitoJUnit.rule().strictness(Strictness.STRICT_STUBS)` is applied (or the `@MockitoSettings(strictness = Strictness.STRICT_STUBS)` JUnit5 annotation).
- [ ] No unused stubs reported under `Strictness.STRICT_STUBS`.
- [ ] Final-class instrumented tests carry `@SdkSuppress(minSdkVersion = 28)`.

## References

- Mockito reference: https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mockito.html
- mockito-kotlin: https://github.com/mockito/mockito-kotlin
- Mockito 5 release notes: https://github.com/mockito/mockito/releases/tag/v5.0.0
- Android Developers — Local tests: https://developer.android.com/training/testing/local-tests
- Android Developers — Test doubles: https://developer.android.com/training/testing/fundamentals/test-doubles
- `androidx/car/app/app/src/test/java/androidx/car/app/serialization/ListDelegateTest.kt` — canonical mockito-kotlin DSL (`mock<T>()`, `argumentCaptor<T>()`, `verify(... atLeastOnce())`, `lastValue`).
- `androidx/compose/ui/ui/src/androidHostTest/kotlin/androidx/compose/ui/input/IndirectPointerEventWithInputDeviceMockTest.kt` — `@RunWith(MockitoJUnitRunner::class)` + `@SdkSuppress(minSdkVersion = 28)` + raw `` `when` `` (legacy form).
- `androidx/compose/material/material/src/androidHostTest/resources/mockito-extensions/org.mockito.plugins.MockMaker` — literal `mock-maker-inline` contents.
- `androidx/compose/foundation/foundation/src/androidHostTest/resources/mockito-extensions/org.mockito.plugins.MockMaker` — same plugin file convention.
- `androidx/room3/room3-runtime/src/androidHostTest/resources/mockito-extensions/org.mockito.plugins.MockMaker` — same plugin file convention.
