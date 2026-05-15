---
name: testing-coroutines-with-runtest
description: Use this skill to test suspend functions and coroutine-using classes on the JVM with kotlinx-coroutines-test. Covers runTest, TestScope, StandardTestDispatcher vs UnconfinedTestDispatcher, virtual time via TestCoroutineScheduler (advanceTimeBy, advanceUntilIdle, runCurrent), the canonical MainDispatcherRule wrapper for Dispatchers.setMain/resetMain, the TestResult Promise contract on KMP, and the runBlockingTest -> runTest migration. If the user mentions runTest, runBlockingTest deprecated, advanceUntilIdle, advanceTimeBy, TestScope, StandardTestDispatcher, UnconfinedTestDispatcher, MainDispatcherRule, "Module with the Main dispatcher is missing", "test hangs forever", dispatchTimeoutMs, viewModelScope test, or a ViewModel that posts state from a coroutine, use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - kotlin-coroutines
  - jvm-testing
  - run-test
  - test-scope
  - test-dispatcher
  - virtual-time
  - main-dispatcher-rule
  - viewmodel-testing
  - runblockingtest-migration
  - coroutines-test
---

# Testing Coroutines With runTest — Virtual Time Without Wall Clock

`runTest` is the only correct entry point for testing suspend functions and `viewModelScope`-backed code on the JVM. This skill nails down the `TestScope` contract, the two `TestDispatcher` flavors, the shared `TestCoroutineScheduler` virtual clock, and the `MainDispatcherRule` plumbing that keeps `Dispatchers.setMain` from leaking between tests. Flow-specific assertions live in `../testing-flows-with-turbine/SKILL.md`.

## When to use this skill

- The class under test exposes a `suspend fun` or launches into `viewModelScope` / `lifecycleScope`.
- The developer reaches for `runBlocking { … }` and the test hangs, or a 10-minute `delay` makes the test slow.
- A ViewModel emits `Loading -> Success` from a coroutine and the test needs to assert each intermediate state.
- The build emits the deprecation `runBlockingTest is deprecated. Use runTest` or `runTest(... dispatchTimeoutMs = ...)` errors.
- The test fails with `IllegalStateException: Module with the Main dispatcher is missing` because `viewModelScope` routed work through `Dispatchers.Main`.
- The developer needs to advance virtual time (`advanceTimeBy(5.seconds)`, `advanceUntilIdle()`) to trigger a timeout / retry / debounce.

## When NOT to use this skill

- The test asserts `Flow` emissions across time. Use `../testing-flows-with-turbine/SKILL.md` (Turbine handles cancellation, hot vs cold, `awaitItem`/`awaitComplete`).
- The test exercises Compose's `MainTestClock` rather than `kotlinx.coroutines.test`. Use `../../../compose/synchronization/controlling-the-test-clock/SKILL.md`.
- The test runs on an emulator/device through AndroidJUnit4. JUnit4 setup itself is `../../runner/configuring-junit4-on-android/SKILL.md`.

## Prerequisites

- Gradle: `testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.10.x")` (`docs/CORPUS.md` §G.1). MUST be `testImplementation` only — never `implementation`. The library installs `Dispatchers.setMain` plumbing that is unsafe to ship to production.
- `@file:OptIn(ExperimentalCoroutinesApi::class)` (or per-call) on test files. `currentTime`, `advanceTimeBy`, `advanceUntilIdle`, `runCurrent`, and the dispatcher constructors are still `@ExperimentalCoroutinesApi` on 1.10.x (R5).
- For ViewModel tests, an injected `CoroutineDispatcher` (a "DispatcherProvider") so the SUT can route work onto the same `TestCoroutineScheduler` the test body advances. Hard-coded `Dispatchers.IO` / `Dispatchers.Default` will NOT see virtual time.
- For KMP libraries that compile to JS / wasm, use the single-expression form `fun foo() = runTest { … }` so the platform `TestResult` is returned (R5; see "TestResult contract" below).

## runTest signature

```kotlin
public fun runTest(
    context: CoroutineContext = EmptyCoroutineContext,
    timeout: Duration = 60.seconds,
    testBody: suspend TestScope.() -> Unit
): TestResult
```

(`docs/CORPUS.md` §G.4.) Three things to internalize:

1. `timeout` is the **whole-test deadline**, NOT a per-dispatch quiescence timeout. The deprecated `dispatchTimeoutMs` overload had different semantics — see migration below.
2. After `testBody` returns, `runTest` waits for every child coroutine launched on `TestScope` to complete. Children launched on `TestScope.backgroundScope` are auto-cancelled at end of test instead. Use `backgroundScope` for hot-flow collectors.
3. Uncaught exceptions in children are aggregated and rethrown as a single failure at the end of the test.

## TestScope members

```kotlin
sealed interface TestScope : CoroutineScope {
    val testScheduler: TestCoroutineScheduler   // shared virtual clock
    val backgroundScope: CoroutineScope         // children auto-cancelled at test end
    val currentTime: Long                       // virtual ms elapsed
    val testTimeSource: TimeSource.WithComparableMarks
}
```

Plus library extensions on `TestScope`: `advanceTimeBy(delay)`, `advanceTimeBy(durationMillis)`, `advanceUntilIdle()`, `runCurrent()`. (R5.)

## TestDispatcher truth table

| Dispatcher | Behavior | Pick when |
|---|---|---|
| `StandardTestDispatcher` | Queues continuations on the scheduler. Nothing runs until `runCurrent()`/`advanceTimeBy(...)`/`advanceUntilIdle()`. **Default for `runTest`.** | Asserting intermediate states (`Loading` -> `Success`); precise virtual-time control; race-free repro. |
| `UnconfinedTestDispatcher` | Eager: a `launch { }` runs synchronously up to its first real suspension. | Hot `StateFlow`/`SharedFlow` collector setup so the first emission is observed without explicit `runCurrent()`. |

(`docs/CORPUS.md` §G.5.)

**Default to `StandardTestDispatcher`.** Reach for `UnconfinedTestDispatcher` only when collector eagerness genuinely matters and you can defend why; it hides ordering bugs that production code under `Dispatchers.Default` would lose.

## TestCoroutineScheduler — the virtual clock

`TestCoroutineScheduler` is the single source of virtual time shared by every `TestDispatcher` participating in a test.

| API | Effect |
|---|---|
| `runCurrent()` | Drains tasks already due at `currentTime`. Does NOT advance the clock. |
| `advanceTimeBy(delta)` | Advances `currentTime` by `delta`, runs every task whose deadline is reached. |
| `advanceUntilIdle()` | Loops `runCurrent` + advance until the queue is empty. "Run everything to completion." |
| `currentTime` | Read-only virtual milliseconds elapsed. |

(R5.)

CRITICAL: `Dispatchers.IO`, `Dispatchers.Default`, and any `newSingleThreadContext(...)` are real thread pools and do NOT participate in the virtual clock. `delay()` inside `withContext(Dispatchers.IO)` uses real wall-clock time. The fix is dependency injection — pass a `CoroutineDispatcher` and substitute `mainRule.dispatcher` (or `StandardTestDispatcher(testScheduler)`) in tests.

## Dispatchers.setMain / resetMain — required for ViewModels

Android's `Dispatchers.Main` is a `HandlerContext` over the main `Looper` that does not exist on a JVM unit test. Anything launched via `viewModelScope`, `lifecycleScope`, or `flowOn(Dispatchers.Main)` will throw `IllegalStateException: Module with the Main dispatcher is missing` without `Dispatchers.setMain`.

### Canonical MainDispatcherRule (androidx)

Verbatim from `androidx/testutils/testutils-ktx/src/jvmMain/kotlin/androidx/testutils/MainDispatcherRule.jvm.kt` (`docs/CORPUS.md` §G.3):

```kotlin
class MainDispatcherRule(
    private val dispatcher: CoroutineDispatcher,
) : TestRule {
    @OptIn(ExperimentalCoroutinesApi::class)
    override fun apply(base: Statement?, description: Description?) =
        object : Statement() {
            override fun evaluate() {
                Dispatchers.setMain(dispatcher)
                try { base!!.evaluate() } finally { Dispatchers.resetMain() }
            }
        }
}
```

### TestWatcher flavor — when you need to expose the dispatcher

```kotlin
@ExperimentalCoroutinesApi
class MainCoroutineRule(
    val dispatcher: TestDispatcher = StandardTestDispatcher(),
) : TestWatcher() {
    override fun starting(description: Description?) {
        super.starting(description); Dispatchers.setMain(dispatcher)
    }
    override fun finished(description: Description?) {
        super.finished(description); Dispatchers.resetMain()
    }
}
```

### Usage

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class MyVmTest {
    @get:Rule val mainRule = MainCoroutineRule()

    @Test fun loadsItems() = runTest(mainRule.dispatcher) {
        val vm = MyViewModel(repo, mainRule.dispatcher)
        vm.load()
        runCurrent()                              // see Loading state
        assertEquals(Loading, vm.state.value)
        advanceUntilIdle()                        // run network/db
        assertEquals(Success(items), vm.state.value)
    }
}
```

## TestResult contract — JVM vs JS

```kotlin
public expect class TestResult
```

| Platform | Actual | Caller obligation |
|---|---|---|
| JVM | `Unit` | Anything works. |
| Native | `Unit` | Same. |
| Kotlin/JS, wasm | `Promise<Unit>` | Test function MUST `return runTest { … }`. |

(R5.)

For Android-only projects, you can write `@Test fun foo() { runTest { … } }` because `TestResult` collapses to `Unit`. For a KMP module, that swallows the `Promise<Unit>` and the JS framework cannot await the test — write `@Test fun foo() = runTest { … }` (single-expression form) every time. **PREFERRED:** always use the single-expression form to avoid surprises if the module ever ships KMP targets.

## Migration from runBlockingTest

| 1.5.x and older | 1.7+ |
|---|---|
| `runBlockingTest { … }` | `runTest { … }` |
| `TestCoroutineScope` | `TestScope` |
| `TestCoroutineDispatcher` | `StandardTestDispatcher` / `UnconfinedTestDispatcher` |
| `pauseDispatcher { } / resumeDispatcher` | gone — explicit `runCurrent` / `advanceTimeBy` |
| `cleanupTestCoroutines()` | unnecessary — `runTest` handles cleanup |
| `runTest(... dispatchTimeoutMs = X) { … }` | `runTest(... timeout = X.milliseconds) { … }` (semantics changed) |

`runBlockingTest` is deprecated **with WARNING**; the `dispatchTimeoutMs` overload is deprecated **with ERROR** (R5; `docs/CORPUS.md` §G.2). The migration replacement is mechanical, but the timeout semantics are not identical: old `dispatchTimeoutMs` was per-dispatch quiescence, new `timeout` is the whole-test deadline.

## Patterns

### Pattern: WRONG — `runBlocking` to test a suspend function

```kotlin
// WRONG
@Test fun loadsItems() = runBlocking {
    val vm = MyViewModel(repo)
    vm.load()
    Thread.sleep(2_000)                          // wait for the launch
    assertEquals(Success(items), vm.state.value)
}
// WRONG because: runBlocking has no virtual time and no exception aggregation. The
// delay in load() runs in real time; the launch may not have completed; uncaught
// exceptions in children are silently swallowed.
```

```kotlin
// RIGHT
@Test fun loadsItems() = runTest(mainRule.dispatcher) {
    val vm = MyViewModel(repo, mainRule.dispatcher)
    vm.load()
    advanceUntilIdle()                           // virtual time flushes everything
    assertEquals(Success(items), vm.state.value)
}
```

### Pattern: WRONG — assert before advancing

```kotlin
// WRONG
@Test fun loadsItems() = runTest(mainRule.dispatcher) {
    val vm = MyViewModel(repo, mainRule.dispatcher)
    vm.load()
    assertEquals(emptyList(), vm.state.value.items)   // "passes" trivially
}
// WRONG because: with StandardTestDispatcher (the default), nothing runs until you
// advance. The launch inside vm.load() is queued but has not executed. The test
// passes for the wrong reason — the items list is empty because load() never ran.
```

```kotlin
// RIGHT
@Test fun loadsItems() = runTest(mainRule.dispatcher) {
    val vm = MyViewModel(repo, mainRule.dispatcher)
    vm.load()
    advanceUntilIdle()
    assertEquals(items, vm.state.value.items)
}
```

### Pattern: WRONG — withContext(Dispatchers.IO) under runTest

```kotlin
// WRONG: production code
class Repo {
    suspend fun fetch() = withContext(Dispatchers.IO) {
        delay(2_000)                              // real wall-clock time
        api.fetch()
    }
}
// WRONG because: Dispatchers.IO is a real thread pool. delay(2_000) takes 2 seconds
// of real time. advanceTimeBy(2_000) does NOT skip it. The whole test runs in real
// time and may exceed the 60s runTest timeout.
```

```kotlin
// RIGHT: production code
class Repo(private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO) {
    suspend fun fetch() = withContext(ioDispatcher) {
        delay(2_000)
        api.fetch()
    }
}
// RIGHT: test
@Test fun fetchesAfterTwoSeconds() = runTest(mainRule.dispatcher) {
    val repo = Repo(ioDispatcher = mainRule.dispatcher)   // shares scheduler
    val deferred = async { repo.fetch() }
    advanceTimeBy(2_001)                                  // virtual; instant in real time
    assertEquals(expected, deferred.await())
}
```

### Pattern: WRONG — collect on TestScope (test hangs)

```kotlin
// WRONG
@Test fun observes() = runTest {
    val seen = mutableListOf<Int>()
    vm.uiState.collect { seen += it }            // never returns
    assertEquals(listOf(0, 1), seen)             // unreachable
}
// WRONG because: vm.uiState is a hot StateFlow. collect on the test's TestScope
// never completes; runTest waits for child completion and hangs until the 60s
// timeout fires. Use backgroundScope (auto-cancelled at end of test) — or Turbine.
```

```kotlin
// RIGHT — backgroundScope
@Test fun observes() = runTest(mainRule.dispatcher) {
    val seen = mutableListOf<Int>()
    vm.uiState
        .onEach { seen += it }
        .launchIn(backgroundScope)
    advanceUntilIdle()
    vm.refresh()
    advanceUntilIdle()
    assertEquals(listOf(0, 1), seen)
}
```

For Flow assertions specifically, prefer Turbine — see `../testing-flows-with-turbine/SKILL.md`.

### Pattern: the "two schedulers" trap

```kotlin
// WRONG
@get:Rule val mainRule = MainCoroutineRule()       // scheduler A

@Test fun broken() = runTest {                      // scheduler B (auto-created)
    val vm = MyViewModel(repo, mainRule.dispatcher) // posts to A
    vm.load()
    advanceUntilIdle()                              // flushes B only — A's work pending
    assertEquals(Success(items), vm.state.value)    // FAIL: still Loading
}
// WRONG because: by default runTest creates its own StandardTestDispatcher with a
// fresh TestCoroutineScheduler. mainRule's dispatcher has a different scheduler;
// advanceUntilIdle() only flushes the test body's scheduler. (Recent kotlinx-
// coroutines-test versions do try to reuse Main's scheduler when Main was replaced
// with a TestDispatcher BEFORE runTest starts, but the defensive contract — pass
// the dispatcher explicitly — never breaks across versions.)
```

```kotlin
// RIGHT — share the scheduler explicitly
@Test fun ok() = runTest(mainRule.dispatcher) {
    val vm = MyViewModel(repo, mainRule.dispatcher)
    vm.load()
    advanceUntilIdle()
    assertEquals(Success(items), vm.state.value)
}
```

(R5 gotcha #4.)

## Mandatory rules

- **MUST** use `runTest { … }`, not `runBlocking { … }` or the deprecated `runBlockingTest { … }`.
- **MUST** depend on `kotlinx-coroutines-test` from `testImplementation` only. **MUST NOT** put it on `implementation`.
- **MUST** apply `@file:OptIn(ExperimentalCoroutinesApi::class)` (or per-call) before using `currentTime`, `advanceTimeBy`, `advanceUntilIdle`, `runCurrent`, `StandardTestDispatcher(...)`, or `UnconfinedTestDispatcher(...)`.
- **MUST** install `Dispatchers.setMain(testDispatcher)` (via `MainDispatcherRule`/`MainCoroutineRule`) for any ViewModel/`viewModelScope` test. Reset in the rule's finally / `@After`.
- **MUST** pass `mainRule.dispatcher` into `runTest(...)` so the `TestScope` and Main share a single scheduler. Otherwise `advanceUntilIdle()` only flushes one of them.
- **MUST** inject any `CoroutineDispatcher` your production code uses for IO/Default work. Hard-coded `Dispatchers.IO` will run in real time even under `runTest`.
- **MUST** launch hot-flow collectors on `TestScope.backgroundScope` (or use Turbine). Never `collect` directly on the `TestScope` — the test hangs to the 60 s timeout.
- **MUST NOT** assert state without first calling `advanceUntilIdle()` (or `runCurrent()` for the "right now" microtask queue). With `StandardTestDispatcher`, queued work has not run yet.
- **MUST NOT** rely on `runTest(... dispatchTimeoutMs = X)` — deprecated with error. Use `timeout = X.milliseconds`; remember the semantics changed (whole-test deadline, not per-dispatch quiescence).
- **PREFERRED:** the single-expression form `@Test fun foo() = runTest { … }` for KMP correctness on JS/wasm where `TestResult` is `Promise<Unit>`.
- **PREFERRED:** default to `StandardTestDispatcher`. Use `UnconfinedTestDispatcher` only for hot-flow collector setup where eagerness is the point.
- **PREFERRED:** `kotlinx-coroutines-debug` on the test classpath so `runTest` timeouts include a coroutine stack dump.

## Verification

- [ ] `./gradlew :module:test` passes with no `Module with the Main dispatcher is missing` exception.
- [ ] No `runBlocking { … }` or `runBlockingTest { … }` calls remain in test sources (`grep -rn 'runBlocking[^T]' src/test`; `grep -rn 'runBlockingTest' src/test`).
- [ ] No `dispatchTimeoutMs` parameter remains (`grep -rn 'dispatchTimeoutMs' src/test`).
- [ ] Every ViewModel/`viewModelScope` test class declares a `@get:Rule MainCoroutineRule` (or equivalent) and passes its `dispatcher` into `runTest(...)`.
- [ ] Every hot-flow collector in tests uses `backgroundScope` or `flow.test { … }` (Turbine), never bare `collect` on `TestScope`.
- [ ] Production classes that use `Dispatchers.IO` / `Dispatchers.Default` accept an injectable `CoroutineDispatcher` and the test substitutes `mainRule.dispatcher`.
- [ ] `@file:OptIn(ExperimentalCoroutinesApi::class)` is on every test file using `advanceTimeBy` / `advanceUntilIdle` / `runCurrent` / `currentTime`.
- [ ] KMP modules (commonTest) use `fun foo() = runTest { … }` single-expression form.

## References

- JetBrains — kotlinx.coroutines.test: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/
- JetBrains — runTest: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/kotlinx.coroutines.test/run-test.html
- JetBrains — TestDispatcher: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/kotlinx.coroutines.test/-test-dispatcher/
- Android Developers — Test Kotlin coroutines on Android: https://developer.android.com/kotlin/coroutines/test
- Android Developers blog — Coroutines on Android, Part III (testing): https://medium.com/androiddevelopers/coroutines-on-android-part-iii-real-work-2ba8a2ec2f45
- `androidx/testutils/testutils-ktx/src/jvmMain/kotlin/androidx/testutils/MainDispatcherRule.jvm.kt` — canonical `MainDispatcherRule` (`docs/CORPUS.md` §G.3).
- `androidx/pdf/.../MainCoroutineRule.kt` — `TestWatcher` flavor that exposes the dispatcher.
- `androidx/navigationevent/...NavigationEventDispatcherTest.kt` — `runTest(UnconfinedTestDispatcher())` + `launchIn(backgroundScope)` pattern.
- `androidx/room3/.../InvalidationTrackerTest.kt` — injecting `TestScope().coroutineContext` into the SUT.
- Sibling: `../testing-flows-with-turbine/SKILL.md` — Flow assertions, `awaitItem`/`awaitComplete`, hot vs cold cancellation.
- Sibling: `../../mocking/mocking-with-mockito/SKILL.md` — mocking suspend functions with mockito-kotlin.
- Sibling: `../../mocking/mocking-with-mockk/SKILL.md` — `coEvery`/`coVerify` for suspend.
- Sibling: `../../runner/configuring-junit4-on-android/SKILL.md` — JUnit4 plumbing surrounding `runTest`.
- Sibling: `../../robolectric/using-robolectric-correctly/SKILL.md` — when the SUT needs Android framework on the JVM.
- Cross-set: `../../../fundamentals/strategies/applying-testing-strategies/SKILL.md` — small/medium/big test sizing.
- Cross-set: `../../../fundamentals/strategies/organizing-test-source-sets/SKILL.md` — where coroutine tests belong.
- Cross-set: `../../../compose/synchronization/controlling-the-test-clock/SKILL.md` — Compose's `MainTestClock` (separate from `TestCoroutineScheduler`).
- Cross-set: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md` — when the same SUT runs on a device.
- Research: `tasks/research/R5-coroutines-test-turbine.md`.
