---
name: writing-tests-with-kotlin-test
description: Use this skill to write tests with the `kotlin.test` library — the multiplatform assertion + annotation API that compiles the same in `commonTest` and on the JVM (JUnit4 or JUnit5), Android, JS, Native, and Wasm. Covers the `kotlin.test` annotations (`@Test`, `@Ignore`, `@BeforeTest`, `@AfterTest`) and how they typealias onto the underlying framework; the `assert*` functions (`assertEquals`/`assertNotEquals`, `assertSame`, `assertNull`/`assertNotNull`, `assertIs`/`assertIsNot`, `assertContentEquals`, `assertContains`, `assertTrue`/`assertFalse`, `fail`, `expect`, `assertFails`, `assertFailsWith`); the `@OnlyInputTypes` compile-time type check; the `Asserter`/`AsserterContributor`/`DefaultAsserter` extension point that routes JVM failures to `org.junit.Assert` for clickable diffs; and the `kotlin("test")` capability that auto-selects JUnit4 vs JUnit5. Use when the user mentions `kotlin.test`, `import kotlin.test.*`, `@BeforeTest`, `assertFailsWith`, `assertContentEquals`, `assertIs`, or `kotlin("test")`.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - kotlin
  - kotlin-test
  - multiplatform-testing
  - test-assertions
  - assertFailsWith
  - assertContentEquals
  - assertIs
  - BeforeTest
  - kotlin-test-junit
  - Asserter
---

# Writing Tests With kotlin.test — One Assertion API Across Every Target

`kotlin.test` is Kotlin's own thin test library: framework-agnostic `assert*` functions plus `@Test`/`@BeforeTest`/`@AfterTest` annotations that typealias onto whatever runner is on the classpath (JUnit4, JUnit5, TestNG, the JS/Native runners). Write `import kotlin.test.*` once and the same test source compiles in `commonTest`, on the JVM, and on every Kotlin Multiplatform target. This skill covers the API surface, the `@OnlyInputTypes` compile-time guard, the `Asserter` extension point, and the Gradle wiring. It is the assertion layer; the *runner* layer for Android instrumentation is `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`.

## When to use this skill

- The module is Kotlin Multiplatform and tests live in `commonTest` — JUnit's `org.junit.*` is not available there; `kotlin.test` is.
- The user wants one assertion vocabulary that reads the same on JVM, Android, JS, and Native.
- The user reaches for `assertFailsWith<IllegalStateException> { … }`, `assertContentEquals(...)`, `assertIs<Foo>(value)`, `assertContains(list, x)`, `expect(3) { compute() }`, or `@BeforeTest`/`@AfterTest`.
- A JVM module has `testImplementation(kotlin("test"))` and the test "runs on JUnit but I never added JUnit" — explaining the capability-based resolution.
- `kotlin.test.assertEquals` failures render as a plain `AssertionError` with no diff, and the user wants the JUnit-style comparison failure back.

## When NOT to use this skill

- The user wants rich fluent assertions (`assertThat(x).isEqualTo(...)`, soft assertions, collection matchers) — that is AssertJ / Google Truth / Kotest assertions, deliberately out of `kotlin.test`'s minimal scope.
- The user is writing an Android instrumentation test and asking about the *runner* (`@RunWith(AndroidJUnit4::class)`, `AndroidJUnitRunner`, `androidx.test:*`) — use `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md` and `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`. (You can still use `kotlin.test` assertions inside those tests.)
- The user is testing coroutines / `Flow` — `runTest`, `TestScope`, `Dispatchers.setMain` is `../../../jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md`; Turbine is `../../../jvm-tests/coroutines/testing-flows-with-turbine/SKILL.md`. (`kotlin.test` supplies the `assert*` calls inside those.)
- The user wants Compose UI assertions (`assertIsDisplayed`, `assertTextEquals`) — that is the `compose/assertions/` skills, unrelated to `kotlin.test`.

## Prerequisites

- Gradle: for a JVM/Android module, `testImplementation(kotlin("test"))`. For a Kotlin Multiplatform module, `kotlin.test` is added to the `commonTest` source set: `sourceSets { commonTest.dependencies { implementation(kotlin("test")) } }`.
- The `kotlin("test")` shorthand is a *capability-based* dependency (Kotlin 1.4+). On the JVM it resolves to the JUnit4 variant (`kotlin-test-junit`) by default; if the `Test` task is configured with `useJUnitPlatform()`, it resolves to the JUnit5 variant (`kotlin-test-junit5`) instead. You can also depend on a specific adapter explicitly: `org.jetbrains.kotlin:kotlin-test-junit`, `:kotlin-test-junit5`, `:kotlin-test-testng`, `:kotlin-test-js`.
- A test runner on the classpath for the chosen target — `kotlin.test`'s `@Test` is only an annotation; something has to execute it (JUnit, TestNG, the Kotlin/JS or Kotlin/Native test infra). The adapter artifact pulls it in.

## Workflow

- [ ] **1. Add `kotlin("test")` to the test source set, not a hardcoded JUnit dependency.** In a multiplatform module put it in `commonTest`; in a JVM/Android module use `testImplementation(kotlin("test"))`. Do not also add `junit:junit` or `org.junit.jupiter:*` directly unless you have framework-specific needs — the adapter brings the right one.

- [ ] **2. Import the framework-agnostic symbols.** `import kotlin.test.*` (or the specific ones). The annotations come from `kotlin.test`, not `org.junit`:

```kotlin
import kotlin.test.Test
import kotlin.test.BeforeTest
import kotlin.test.AfterTest
import kotlin.test.Ignore
import kotlin.test.assertEquals

class CartTest {
    private lateinit var cart: Cart

    @BeforeTest fun setUp() { cart = Cart() }      // == @Before (JUnit4) / @BeforeEach (JUnit5)
    @AfterTest  fun tearDown() { cart.clear() }    // == @After  (JUnit4) / @AfterEach  (JUnit5)

    @Test fun emptyCartTotalsZero() {
        assertEquals(Money.ZERO, cart.total())
    }

    @Ignore @Test fun flaky() { /* skipped — == @Disabled on JUnit5 */ }
}
```

  On JVM-JUnit4, `kotlin.test.Test` is `typealias`'d to `org.junit.Test`, `BeforeTest` to `org.junit.Before`, `AfterTest` to `org.junit.After`, `Ignore` to `org.junit.Ignore`. On JUnit5 they map to `org.junit.jupiter.api.Test`/`BeforeEach`/`AfterEach`/`Disabled`. So the same source runs on either engine, and in `commonTest`.

- [ ] **3. Pick the precise assertion.** `kotlin.test` is small on purpose — there is exactly one obvious call per intent:

| Intent | Call |
|---|---|
| Values equal (by `==`) | `assertEquals(expected, actual, message?)` |
| Floating-point within tolerance | `assertEquals(expected, actual, absoluteTolerance, message?)` (`Double`/`Float`, since 1.5) |
| Values not equal | `assertNotEquals(illegal, actual, message?)` (also a float-tolerance overload) |
| Same / not-same instance (`===`) | `assertSame` / `assertNotSame` |
| `null` / not-`null` | `assertNull(actual)` / `assertNotNull(actual)` — `assertNotNull` *returns* the non-null `T` (and has an `assertNotNull(actual) { it -> … }` block form) |
| Is / is-not a type | `assertIs<T>(value)` / `assertIsNot<T>(value)` (since 1.5) — `assertIs<T>` returns `T` smart-cast, so chain assertions on it |
| Ordered content equal | `assertContentEquals(expected, actual, message?)` for `Iterable?`/`Sequence?`/`Array?`/primitive arrays (since 1.5). Order-sensitive — for `Set` use `assertEquals` instead (the `Set` overload of `assertContentEquals` is deprecated as ambiguous) |
| Membership | `assertContains(container, element, message?)` — `Iterable`/`Sequence`/`Array`/`Map` (key)/`IntRange`/`CharSequence`/etc. (since 1.5) |
| Boolean condition | `assertTrue(actual, message?)` / `assertFalse(actual, message?)`, or `assertTrue(message?) { lazyCondition }` when the condition itself is expensive |
| Unconditional failure | `fail(message?)` or `fail(message?, cause)` (since 1.4) — returns `Nothing`, so it satisfies the type checker in `when`/`?:` |
| "this block returns X" | `expect(expected) { block }` / `expect(expected, message) { block }` |
| A block throws | `assertFails { block }` returns the `Throwable`; `assertFailsWith<E> { block }` / `assertFailsWith(E::class) { block }` returns the typed `E` for further assertions on its message |

- [ ] **4. Assert on a thrown exception with `assertFailsWith`, not a `try/catch`.** It returns the exception so you can assert its message/cause:

```kotlin
@Test fun withdrawOverdraws() {
    val e = assertFailsWith<InsufficientFundsException> { account.withdraw(BigDecimal(100)) }
    assertEquals("balance is 20", e.message)
}
```

- [ ] **5. Lean on `@OnlyInputTypes` — type mismatches are compile errors, not runtime failures.** `assertEquals`, `assertContentEquals`, `assertContains`, `assertSame`, `expect` are declared with `<@OnlyInputTypes T>`, so `assertEquals(1, "1")` does not compile. Don't widen to `Any` to "make it pass" — fix the test.

- [ ] **6. Restore JUnit-style diffs on the JVM by keeping the JUnit adapter on the classpath.** `kotlin.test` resolves an `Asserter` at runtime via `ServiceLoader<AsserterContributor>`. The `kotlin-test-junit` artifact registers a contributor that, when `org.junit.Assert` is present, returns a `JUnitAsserter` delegating `assertEquals` to `org.junit.Assert.assertEquals` — which throws `ComparisonFailure`, giving the clickable expected/actual diff in IDEs. With no contributor, `DefaultAsserter` throws a plain `AssertionError`. So `kotlin.test` failures look like JUnit failures automatically as long as the adapter is there (it is, via `kotlin("test")`). Only implement a custom `Asserter` + register an `AsserterContributor` (JVM: `META-INF/services/kotlin.test.AsserterContributor`) if you need a non-JUnit reporting backend.

## Patterns

### Pattern: importing `org.junit.*` annotations in a multiplatform-shared test

```kotlin
// WRONG — in src/commonTest, or in src/test that you later want to share
import org.junit.Test            // unresolved in commonTest; couples JVM source to JUnit4
import org.junit.Assert.assertEquals
class FooTest {
    @Test fun bar() { assertEquals(2, foo()) }
}
// WRONG because: org.junit is a JVM-JUnit4 dependency. It does not exist in commonMain/commonTest,
// it pins you to JUnit4 (not 5), and it cannot run on Kotlin/JS or Kotlin/Native.
```

```kotlin
// RIGHT — kotlin.test only; runs everywhere, on whatever engine the build selects
import kotlin.test.Test
import kotlin.test.assertEquals
class FooTest {
    @Test fun bar() { assertEquals(2, foo()) }
}
```

### Pattern: hand-rolled exception checking

```kotlin
// WRONG
@Test fun parseRejectsGarbage() {
    try {
        parse("garbage")
        fail("expected ParseException")
    } catch (e: ParseException) {
        // ok — but verbose, and a different exception type slips through as a test error not a failure
    }
}
// WRONG because: assertFailsWith already does exactly this, returns the exception for assertions,
// and reports a clear "expected ParseException but was X" message.
```

```kotlin
// RIGHT
@Test fun parseRejectsGarbage() {
    val e = assertFailsWith<ParseException> { parse("garbage") }
    assertEquals(0, e.offset)
}
```

### Pattern: `assertTrue(a == b)` instead of `assertEquals`

```kotlin
// WRONG
assertTrue(result == expected)
// WRONG because: on failure the message is just "Expected value to be true." — no expected/actual.
// assertEquals reports both values (and a diff via the JUnit asserter).
```

```kotlin
// RIGHT
assertEquals(expected, result)
// reserve assertTrue for genuine boolean predicates: assertTrue(result.isValid, "result was rejected: ${result.errors}")
```

### Pattern: comparing list content

```kotlin
// WRONG — assertEquals on lists works, but assertEquals on the wrong shape is silent type widening
assertEquals(expected as List<Any>, actual)         // never widen to Any to dodge a type error
```

```kotlin
// RIGHT — assertContentEquals is the explicit "same elements, same order" check; @OnlyInputTypes keeps it honest
assertContentEquals(listOf(1, 2, 3), service.ids())
// for a Set (order-insensitive) use assertEquals: assertEquals(setOf("a", "b"), service.tags())
```

## Mandatory rules

- **MUST** depend on `kotlin("test")` (in `commonTest` for multiplatform, `testImplementation(kotlin("test"))` for JVM/Android) rather than importing `org.junit.*` directly, so the test source is engine-agnostic and multiplatform-portable.
- **MUST** import annotations from `kotlin.test` (`@Test`, `@BeforeTest`, `@AfterTest`, `@Ignore`), not `org.junit` / `org.junit.jupiter.api`.
- **MUST** use `assertEquals`/`assertNotEquals`/`assertContentEquals`/`assertIs`/`assertContains` for their specific intents instead of collapsing everything into `assertTrue(...)` — the specific calls produce expected/actual diagnostics.
- **MUST** check thrown exceptions with `assertFailsWith<E> { … }` (returning the exception for message/cause assertions), not `try/catch` + `fail`.
- **MUST NOT** widen argument types (`as Any`, `as List<Any>`) to silence an `@OnlyInputTypes` compile error — that error is catching a real mismatch.
- **MUST NOT** call `assertContentEquals` on a `Set` (deprecated, ambiguous); use `assertEquals` for unordered set equality.
- **MUST NOT** drop the `kotlin-test-junit`/`kotlin-test-junit5` adapter on the JVM — without an `AsserterContributor` on the classpath, failures degrade to bare `AssertionError` with no diff.
- **PREFERRED:** `assertNotNull(x)` / `assertIs<T>(x)` over `x!!` / `x as T` in tests — they fail with a clear message instead of an NPE/`ClassCastException` and return the narrowed value.
- **PREFERRED:** the `assertTrue(message) { expensiveCondition() }` lazy-block overload when computing the condition or message is costly.

## Verification

- [ ] No `import org.junit.*` or `import org.junit.jupiter.api.*` in source sets meant to be multiplatform/shared; `grep -rn 'import org.junit' src/commonTest src/test` is empty (or justified per-platform only).
- [ ] `./gradlew :module:dependencies --configuration testRuntimeClasspath` (JVM) shows `kotlin-test-junit` or `kotlin-test-junit5` resolved via the `kotlin("test")` capability.
- [ ] Test classes use `@Test`/`@BeforeTest`/`@AfterTest` from `kotlin.test`.
- [ ] Exception cases use `assertFailsWith<…> { … }` rather than `try/catch`.
- [ ] No `as Any` / `as List<Any>` widening was added to make an assertion compile.
- [ ] A deliberately failing `assertEquals` in a JVM module renders as a JUnit `ComparisonFailure` (clickable diff) — proof the `AsserterContributor` is on the classpath.

## References

- kotlinlang.org/api/latest/kotlin.test/ — the `kotlin.test` API reference: every `assert*` function, `expect`, `fail`, the annotations, `Asserter`/`AsserterContributor`/`DefaultAsserter`.
- kotlinlang.org/docs/jvm-test-using-junit.html and kotlinlang.org/docs/multiplatform-run-tests.html — using `kotlin("test")` on the JVM (JUnit4/JUnit5 selection) and across multiplatform targets.
- `libraries/kotlin.test/annotations-common/src/main/kotlin/kotlin.test/Annotations.kt` (Kotlin repo) — `expect annotation class Test/Ignore/BeforeTest/AfterTest`; the JVM `actual typealias`es in `libraries/kotlin.test/junit/src/main/kotlin/Annotations.kt` (→ `org.junit.*`) and `libraries/kotlin.test/junit5/src/main/kotlin/Annotations.kt` (→ `org.junit.jupiter.api.*`).
- `libraries/kotlin.test/common/src/main/kotlin/kotlin/test/Assertions.kt` (Kotlin repo) — `assertTrue`/`assertFalse`/`assertEquals`/`assertNotEquals`/`assertSame`/`assertNotSame`/`assertNull`/`assertNotNull`/`assertIs`/`assertIsNot`/`assertContains`/`assertContentEquals`/`fail`/`expect`/`assertFails`/`assertFailsWith` signatures, the `@OnlyInputTypes` annotations, and the `Asserter` / `AsserterContributor` interfaces.
- `libraries/kotlin.test/jvm/src/main/kotlin/AsserterLookup.kt` and `libraries/kotlin.test/junit/src/main/kotlin/JUnitSupport.kt` (Kotlin repo) — `ServiceLoader<AsserterContributor>` lookup; `JUnitContributor`/`JUnitAsserter` delegating to `org.junit.Assert` when present, else `DefaultAsserter`.
- Cross-set: `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md` — the JUnit4 runner / `AndroidJUnit4` stack; `kotlin.test` assertions slot into those tests unchanged.
- Cross-set: `../../../jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md` — `runTest` is the surrounding scope; `kotlin.test` supplies the `assert*` calls inside it.
- Cross-set: `../../../jvm-tests/coroutines/testing-flows-with-turbine/SKILL.md` — Turbine's `awaitItem()` paired with `kotlin.test`'s `assertEquals`.
- Cross-set: `../../../fundamentals/strategies/organizing-test-source-sets/SKILL.md` — where these tests live (`src/test/`, `src/commonTest/`) and the `testImplementation` vs `androidTestImplementation` split.
- Cross-set: `../../../fundamentals/doubles/picking-test-doubles/SKILL.md` — fakes/stubs the assertions are checking.
