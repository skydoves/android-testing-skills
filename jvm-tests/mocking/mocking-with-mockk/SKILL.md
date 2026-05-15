---
name: mocking-with-mockk
description: Use this skill to wire MockK (the Kotlin-first mocking framework) into a JVM unit-test suite, especially when coroutines, singleton/`object` mocking, or constructor mocking dominate. Covers `io.mockk:mockk-jvm:1.14.x` (and `mockk-android` + `mockk-agent` for instrumented), `every { } returns`, `coEvery` / `coVerify` (native suspend support — the big win over Mockito), `mockk(relaxed = true)` / `relaxUnitFun = true`, `slot<T>()` capture, `verifySequence` / `verifyOrder` / `confirmVerified`, `mockkStatic` / `mockkObject` / `mockkConstructor` with `unmockkAll()` cleanup, `@MockK` / `@RelaxedMockK` / `@SpyK` / `@InjectMockKs` annotations, and the Mockito-vs-MockK decision criteria. Use when the user mentions `coEvery`, `mockkObject`, `mockkStatic`, `slot`, `MockKException`, `every returns vs coEvery`, `relaxed = true`, or asks how to mock a Kotlin `object`/singleton.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - mockk
  - kotlin-mocking
  - coroutines
  - coEvery
  - coVerify
  - mockkStatic
  - mockkObject
  - mockkConstructor
  - relaxed-mock
---

# Mocking with MockK — The Kotlin-First Mocking Stack

MockK is the Kotlin-first alternative to Mockito. Its headline wins are native suspend support (`coEvery` / `coVerify` with no `runBlocking` dance), first-class singleton / `object` / static / constructor mocking, and Kotlin-native syntax (`every { … } returns x`). The trade-off: androidx itself does NOT use MockK (a `grep -r "io.mockk"` over the AOSP checkout returns zero hits — see `../mocking-with-mockito/SKILL.md` for the dominant pattern). New Kotlin code outside Google often picks MockK; new Kotlin code inside Google or matching AOSP conventions picks Mockito. Both are valid.

## When to use this skill

- The codebase is Kotlin-only and dominated by suspend functions / coroutines, and the user wants ergonomic suspend stubbing.
- The user needs to mock a Kotlin `object` (singleton), companion object, top-level function (`Kt`-suffixed file), or every newly-constructed instance of a class.
- The user wants strict-by-default verification with the option to opt out per-mock via `relaxed = true` / `relaxUnitFun = true`.
- The user wants `slot<T>()` capture with `slot.captured` (cleaner than Mockito's `ArgumentCaptor`).
- The user is writing tests for `viewModelScope`, `Flow` collectors, `LaunchedEffect`-style suspend collaborators.

## When NOT to use this skill

- The codebase is mixed Java/Kotlin or already standardized on Mockito. Use `../mocking-with-mockito/SKILL.md`. Do not migrate without a reason — both frameworks are valid.
- The user is matching AOSP / androidx conventions (which are Mockito-only). Use `../mocking-with-mockito/SKILL.md`.
- Behaviour matters more than interactions, e.g. a `Repository` with caching logic — write a fake instead. See `../../../fundamentals/doubles/picking-test-doubles/SKILL.md` (Google explicitly prefers fakes per /test-doubles).
- The runner / Gradle matrix isn't set up yet — start with `../../runner/configuring-junit4-on-android/SKILL.md`.
- The user is testing `Flow` emissions over time — pair MockK with Turbine. See `../../coroutines/testing-flows-with-turbine/SKILL.md`.
- The user needs `runTest` semantics — see `../../coroutines/testing-coroutines-with-runtest/SKILL.md`.

## Prerequisites

- The base test wiring from `../../runner/configuring-junit4-on-android/SKILL.md` is already in place.
- A `MainDispatcherRule` is installed if any code-under-test touches `Dispatchers.Main` — see the runner skill.
- For instrumented MockK (Android runtime), the device must be API 21+ (MockK supports back to API 21 on dexmaker).

## Workflow

- [ ] **1. Add the MockK coordinates.** Pin to a specific 1.14.x version (current latest at the time of writing):

```kotlin
dependencies {
    // JVM unit tests (src/test/)
    testImplementation("io.mockk:mockk-jvm:1.14.0")

    // Instrumented tests (src/androidTest/) — different artifact + agent
    androidTestImplementation("io.mockk:mockk-android:1.14.0")
    androidTestImplementation("io.mockk:mockk-agent:1.14.0")
}
```

`mockk-android` swaps the bytecode-generation backend so MockK runs inside Dalvik/ART where stock ByteBuddy doesn't work. Use `mockk-jvm` (NOT `mockk-android`) for `src/test/` Robolectric tests — Robolectric runs on the JVM.

- [ ] **2. Wire mocks via `MockKAnnotations.init(this)` or direct `mockk<T>()` construction.** The annotation route reads cleaner for tests with many mocks; direct construction wins for one-off mocks:

```kotlin
import io.mockk.MockKAnnotations
import io.mockk.impl.annotations.MockK
import io.mockk.impl.annotations.RelaxedMockK
import io.mockk.impl.annotations.SpyK
import io.mockk.impl.annotations.InjectMockKs

@RunWith(AndroidJUnit4::class)
class UserViewModelTest {
    @get:Rule val mainDispatcherRule = MainDispatcherRule()

    @MockK lateinit var repo: UserRepository
    @RelaxedMockK lateinit var logger: Logger
    @SpyK var realClock: Clock = SystemClock()
    @InjectMockKs lateinit var subject: UserViewModel

    @Before fun setUp() = MockKAnnotations.init(this, relaxUnitFun = true)
    @After fun tearDown() = unmockkAll()   // see step 5 — cleanup global hooks
}
```

`MockKAnnotations.init` does **not** itself install global hooks (`mockkStatic` / `mockkObject` / `mockkConstructor`); but if the test class adds any of those, the `@After unmockkAll()` is needed to prevent leakage into sibling tests. JUnit5's `MockKExtension` does this automatically — JUnit4 does not.

Direct construction:

```kotlin
val car = mockk<Car>(
    name = "carA",
    relaxed = false,
    relaxUnitFun = true,           // Unit-returning calls auto-relax; reference returns still throw
    moreInterfaces = arrayOf(Comparable::class),
)
```

- [ ] **3. Stub with `every { … } returns …` for plain functions and `coEvery { … } returns …` for suspend.** This is the headline ergonomic win over Mockito — no `runBlocking { whenever(...) }` wrap is needed for suspend stubs:

```kotlin
import io.mockk.coEvery
import io.mockk.coVerify
import io.mockk.every
import io.mockk.verify

every { car.drive(Direction.NORTH) } returns Outcome.OK
every { car.brake() } throws IllegalStateException("frozen")
every { car.gear } returnsMany listOf(1, 2, 3)              // sequence
every { car.compute(any()) } answers { firstArg<Int>() * 2 }

// Suspend — the big win:
coEvery { repo.fetchUser(1) } returns User("Jane")
coEvery { repo.observeUser(1) } returns flowOf(User("Jane"))
```

- [ ] **4. Verify interactions with `verify { … }`, `coVerify { … }`, and the strict variants.** MockK's verifiers are richer than Mockito's:

```kotlin
verify { car.drive(Direction.NORTH) }
verify(exactly = 2) { car.drive(any()) }
verify(exactly = 0) { car.brake() }                          // = never
verify(atLeast = 1, atMost = 3) { car.drive(any()) }

verifyAll { /* must list every interaction — exhaustive, unordered */ }
verifySequence { /* every interaction in this exact order */ }
verifyOrder { /* listed interactions occurred in this order, others may also have happened */ }

coVerify(exactly = 1) { repo.fetchUser(1) }

confirmVerified(car)                                         // strict: nothing else was invoked
```

`confirmVerified` is the safety net to catch silent extra interactions — pair it with `verifyAll`/`verifySequence` for a fully constrained test.

- [ ] **5. Capture arguments with `slot<T>()`.** Cleaner than Mockito's captor — `slot.captured` is a property:

```kotlin
import io.mockk.slot

val slot = slot<User>()
every { repo.save(capture(slot)) } returns Unit

subject.register(User("Jane"))

assertEquals("Jane", slot.captured.name)

// Multiple captures over a sequence of calls:
val users = mutableListOf<User>()
every { repo.save(capture(users)) } returns Unit
// ... many calls ...
assertEquals(3, users.size)
```

- [ ] **6. Use `relaxed = true` or `relaxUnitFun = true` deliberately.** MockK is **strict by default** — any unstubbed call throws `MockKException`. The two relaxation modes:

  - `relaxed = true` — every unstubbed call returns a type-default value (0, false, empty string, nested mock for reference types). Cuts boilerplate but hides bugs.
  - `relaxUnitFun = true` — only `Unit`-returning calls auto-relax; reference returns still throw. Recommended middle ground for logger / metrics / framework-callback collaborators.

```kotlin
val logger = mockk<Logger>(relaxed = true)             // any call -> default
logger.info("anything")                                // no every{} needed
val n = logger.lineCount                               // returns 0

val analytics = mockk<Analytics>(relaxUnitFun = true)  // safer middle ground
analytics.track(Event("foo"))                          // OK — Unit return is relaxed
analytics.session                                      // throws — reference return needs every{}
```

- [ ] **7. Mock singletons / statics / constructors with `mockkObject` / `mockkStatic` / `mockkConstructor` — and ALWAYS clean up.** These install GLOBAL hooks that persist beyond the current test unless explicitly unmocked. Leakage across tests is the #1 MockK footgun.

```kotlin
import io.mockk.mockkObject
import io.mockk.mockkStatic
import io.mockk.mockkConstructor
import io.mockk.unmockkAll

@After fun tearDown() = unmockkAll()    // belt-and-braces cleanup

@Test fun mocksObject() {
    mockkObject(MySingleton)
    every { MySingleton.flag } returns true
    // ... assertions ...
}

@Test fun mocksTopLevel() {
    // Kotlin top-level functions compile as static methods on the file's *Kt class.
    mockkStatic("com.example.UtilsKt")
    every { hashSomething(any()) } returns "deadbeef"
}

@Test fun mocksJavaStatic() {
    mockkStatic(System::class)
    every { System.currentTimeMillis() } returns 0L
}

@Test fun mocksConstructor() {
    mockkConstructor(OkHttpClient::class)
    every { anyConstructed<OkHttpClient>().newCall(any()) } returns fakeCall
}
```

The JUnit5 `MockKExtension` auto-cleans, but JUnit4 tests must call `unmockkAll()` in `@After`.

- [ ] **8. Verify the test runs and the strict-mode failures actually fire.** `./gradlew :<module>:testDebugUnitTest --tests <YourTestClass>`. A passing test that uses `verifyAll` / `verifySequence` / `confirmVerified` on a strict mock means every interaction is accounted for. If MockK throws `MockKException: no answer found for: …`, the SUT is calling something the test didn't stub — either stub it, relax the mock, or fix the SUT.

## Patterns

### Pattern: WRONG vs RIGHT — suspend stubbing

```kotlin
// WRONG
every { mock.suspendFn() } returns x
// WRONG because: every { } produces a non-suspend stub. At call time MockK throws
// MockKException because the actual call carries a Continuation argument that
// every {} did not match against. This is the #1 trip-up when migrating from Mockito.
```

```kotlin
// RIGHT
coEvery { mock.suspendFn() } returns x
coVerify { mock.suspendFn() }
```

### Pattern: WRONG vs RIGHT — leaking static / object mocks across tests

```kotlin
// WRONG
@Test fun firstTest() {
    mockkStatic(System::class)
    every { System.currentTimeMillis() } returns 0L
    // ... assertions ...
    // No unmockkStatic, no unmockkAll, no JUnit5 extension.
}

@Test fun secondTest() {
    // System.currentTimeMillis() is STILL mocked from firstTest. Bizarre failures
    // depending on test ordering follow.
}
// WRONG because: mockkStatic / mockkObject / mockkConstructor install global hooks.
// JUnit4 does not auto-clean MockK state.
```

```kotlin
// RIGHT — JUnit4 with explicit teardown
@After fun tearDown() = unmockkAll()

@Test fun firstTest() {
    mockkStatic(System::class)
    every { System.currentTimeMillis() } returns 0L
    // ...
}
// secondTest now starts with a clean slate.
```

### Pattern: WRONG vs RIGHT — picking a relaxation mode

```kotlin
// WRONG
val service = mockk<PaymentService>(relaxed = true)
service.charge(amount = 100, account = "acct-7")
// Test passes because charge() silently returns a relaxed default. No verify call,
// no every{} stub. The intended interaction was never asserted, and a behaviour
// regression would not surface here.
```

```kotlin
// RIGHT
val service = mockk<PaymentService>()                // strict
every { service.charge(any(), any()) } returns ChargeResult.OK

service.charge(amount = 100, account = "acct-7")

verify(exactly = 1) { service.charge(eq(100), eq("acct-7")) }
confirmVerified(service)
```

`relaxed = true` is appropriate for loggers / metrics where the test really doesn't care about return values. For domain services, prefer the default strict mode plus explicit `every` / `verify`.

## Mandatory rules

- **MUST** use `coEvery { … } returns …` (NOT `every { … } returns …`) when stubbing a `suspend` function. **MUST** use `coVerify { … }` (NOT `verify { … }`) when verifying a `suspend` call.
- **MUST** depend on `io.mockk:mockk-jvm` for `src/test/` and `io.mockk:mockk-android` + `io.mockk:mockk-agent` for `src/androidTest/`. Mixing the artifacts crashes the agent.
- **MUST** call `unmockkAll()` in an `@After` method whenever a JUnit4 test uses `mockkObject`, `mockkStatic`, or `mockkConstructor`. Leakage across tests is the #1 MockK footgun.
- **MUST** prefer `relaxUnitFun = true` over `relaxed = true` when in doubt. The middle ground hides far fewer bugs.
- **MUST** call `MockKAnnotations.init(this, relaxUnitFun = …)` in `@Before` when using `@MockK` / `@RelaxedMockK` / `@SpyK` / `@InjectMockKs`. Without it, the lateinit fields never initialise.
- **MUST** prefer the JUnit5 `MockKExtension` over manual cleanup when the project is on JUnit 5 — it handles teardown of static / object mocks automatically.
- **MUST** match the rest of the codebase's mocking choice. Do not introduce MockK into a Mockito-standardized module without team alignment, and vice versa.
- **MUST** prefer fakes for behaviour-heavy collaborators (per developer.android.com/training/testing/fundamentals/test-doubles: "fakes ... are preferred"). Use MockK for verifying interactions, not for re-implementing behaviour.
- **MUST NOT** mock `inline` functions or `inline class` (value classes) — MockK cannot intercept them. Wrap or refactor instead.
- **MUST NOT** rely on `mockkStatic` for production-code time access (`System.currentTimeMillis`, `Clock.system`). Inject a `Clock` abstraction instead — production code is testable without bytecode rewriting.
- **PREFERRED:** combine `verifySequence` (or `verifyAll`) with `confirmVerified(mock)` for a fully constrained test that fails if any unexpected interaction occurs.

## Verification

- [ ] `./gradlew :<module>:testDebugUnitTest --tests <YourTestClass>` passes.
- [ ] `every { mock.<suspend-fn>() } returns ...` triggers `MockKException("no answer found")` at runtime — let your test failures, not a grep, surface this. (A grep on common suspend-fn name fragments produces too many false positives — `loadConfig`, `fetchSync`, `observeForever` are typically not suspending.)
- [ ] Every test file using `mockkStatic` / `mockkObject` / `mockkConstructor` has either `@After fun tearDown() = unmockkAll()` or extends `MockKExtension` (JUnit5).
- [ ] No file imports both `io.mockk.*` and `org.mockito.*` — pick one framework per module.
- [ ] `@MockK` / `@RelaxedMockK` / `@SpyK` / `@InjectMockKs` fields are paired with a `@Before` method calling `MockKAnnotations.init(this, ...)`.
- [ ] No production code path in tests relies on `mockkStatic(System::class)` for time — a `Clock` abstraction is injected instead.
- [ ] Strict-mode failures actually fire: removing a stub triggers `MockKException: no answer found for: …` when the SUT runs.

## References

- MockK official site: https://mockk.io/
- MockK API reference: https://mockk.io/ANDROID.html
- MockK + coroutines guide: https://mockk.io/#coroutines
- JUnit5 MockKExtension: https://mockk.io/#junit5
- Android Developers — Test doubles: https://developer.android.com/training/testing/fundamentals/test-doubles
- Android Developers — Local tests: https://developer.android.com/training/testing/local-tests
- Mockito-vs-MockK decision context: see `../mocking-with-mockito/SKILL.md` — androidx itself uses Mockito exclusively (400+ files import `org.mockito`, zero import `io.mockk`).
- For coroutine test infrastructure that pairs with MockK's `coEvery`, see `../../coroutines/testing-coroutines-with-runtest/SKILL.md`.
- For testing `Flow` emissions, see `../../coroutines/testing-flows-with-turbine/SKILL.md`.
