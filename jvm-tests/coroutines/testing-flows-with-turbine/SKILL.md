---
name: testing-flows-with-turbine
description: Use this skill to assert Flow emissions in tests with Cash App Turbine. Covers flow.test entry, ReceiveTurbine API (awaitItem, awaitComplete, awaitError, expectMostRecentItem, skipItems, expectNoEvents, cancel and friends, plus the non-suspending takeItem/takeComplete/takeError), turbineScope/testIn for multi-flow tests, the standalone Turbine<T> channel (add/close/asChannel/ensureAllEventsConsumed) for verifying non-Flow callbacks and listeners, the hot vs cold cancellation contract (StateFlow/SharedFlow never auto-complete), Turbine's wall-clock timeout decoupled from runTest virtual time, and the decision matrix between Turbine and raw flow.toList. If the user mentions Turbine, awaitItem, awaitComplete, expectMostRecentItem, takeItem, "test hangs on collect", "SharedFlow never completes", "StateFlow first emission", "TurbineAssertionError", "withTurbineTimeout", "Turbine() channel", "capture a callback in a test", or asserts a ViewModel's StateFlow/SharedFlow, use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - kotlin-coroutines
  - flow-testing
  - turbine
  - state-flow
  - shared-flow
  - await-item
  - await-complete
  - hot-flow
  - cold-flow
  - cash-app-turbine
---

# Testing Flows With Turbine — Assert Emissions Without `toList` Hangs

Turbine is the canonical way to assert `Flow` emissions one-at-a-time, with built-in cancellation that prevents hot flows (`StateFlow`, `SharedFlow`) from hanging the test. This skill covers `flow.test`, the `ReceiveTurbine` API, `turbineScope` for multi-flow tests, and when raw `flow.toList()` actually beats Turbine. The non-Flow coroutine plumbing is `../testing-coroutines-with-runtest/SKILL.md`.

## When to use this skill

- The class under test exposes a `StateFlow<UiState>` or `SharedFlow<Event>` and the test must assert each emission.
- A test hangs on `flow.toList()` over a `SharedFlow` or `StateFlow` — neither completes.
- The developer wants to assert "no more emissions for a beat" or "the next event is an error", which `toList()` cannot express.
- Two flows must be observed concurrently (e.g. `uiState` plus a one-shot `events` flow) and asserted in interleaved order.
- The build error is `TurbineAssertionError: Unconsumed events found` or `Expected an item but found Complete/Error`.
- The developer mentions Turbine APIs: `awaitItem`, `awaitComplete`, `awaitError`, `expectMostRecentItem`, `skipItems`, `expectNoEvents`, `withTurbineTimeout`, `turbineScope`, `testIn`.

## When NOT to use this skill

- The test is about `runTest`, `TestScope`, `Dispatchers.setMain`, or virtual time. Use `../testing-coroutines-with-runtest/SKILL.md` first; Turbine sits on top of `runTest`.
- The flow is cold and completes deterministically, and the assertion is "the full list equals X". `flow.toList()` is shorter — see Decision matrix below.
- The "flow" is actually Compose snapshot state. Read it via the test rule, not Turbine.

## Prerequisites

- Gradle: `testImplementation("app.cash.turbine:turbine:1.x")` (`docs/CORPUS.md` §G.1; R5).
- Turbine multiplatform — JVM, Android, iOS, JS, Native, wasm. Apache 2.0, Square / Cash App.
- The surrounding test must be a `runTest { … }` body (or any `suspend` context). `flow.test { }` is itself `suspend`.
- For multi-flow tests, `@OptIn(ExperimentalCoroutinesApi::class)` because `turbineScope` interacts with `TestScope.backgroundScope`.

## flow.test signature

```kotlin
public suspend fun <T> Flow<T>.test(
    timeout: Duration = 3.seconds,
    name: String? = null,
    validate: suspend ReceiveTurbine<T>.() -> Unit,
)
```

(`docs/CORPUS.md` §G; R5.) Three things to internalize:

1. `test { }` launches a new coroutine that calls `flow.collect { }` and feeds every emission into a `ReceiveTurbine<T>` channel. The collector is attached **before** `validate { }` runs, so it is safe to emit into a hot flow from inside the block without dropping the first item.
2. The 3-second default timeout is **wall clock** — it is NOT driven by `TestCoroutineScheduler` and does NOT honor `runTest`'s virtual time. `awaitItem()` waits up to 3 real seconds.
3. After `validate { }` returns, Turbine asserts no unconsumed events remain. If any item / completion / error is pending, it throws `TurbineAssertionError`. The collecting coroutine is then cancelled, so cold flows do not leak.

The `name` argument is appended to error messages — invaluable when multiple turbines run in one test.

## ReceiveTurbine API

```kotlin
public interface ReceiveTurbine<T> {
    // suspending
    public suspend fun awaitItem(): T
    public suspend fun awaitComplete()
    public suspend fun awaitError(): Throwable
    public suspend fun awaitEvent(): Event<T>
    public suspend fun skipItems(count: Int)
    public suspend fun cancel()
    public suspend fun cancelAndIgnoreRemainingEvents()
    public suspend fun cancelAndConsumeRemainingEvents(): List<Event<T>>

    // non-suspending
    public fun expectNoEvents()
    public fun expectMostRecentItem(): T
    public fun ensureAllEventsConsumed()
}
```

(R5.)

| Call | Use when |
|---|---|
| `awaitItem(): T` | Next event must be an item; fails on `Complete`/`Error`. |
| `awaitComplete()` | Next event must be flow completion; fails on item/error. |
| `awaitError(): Throwable` | Next event must be an error; returns the throwable for further assertions. |
| `awaitEvent(): Event<T>` | Sealed-class entry: `Event.Item(value)`, `Event.Complete`, `Event.Error(throwable)`. |
| `skipItems(n)` | Drop n item events; fails if anything but an item is next. Useful to skip a `StateFlow`'s seed. |
| `expectMostRecentItem()` | Non-suspending; drains buffered items and returns the latest. The "final state" assertion for `StateFlow`-driven UIs where intermediate values are lossy. |
| `expectNoEvents()` | Non-suspending; throws if anything is buffered. Pair with `runCurrent()`/`advanceTimeBy(...)` to assert no spurious emissions. |
| `cancel()` | Stop the collector; any remaining events become an assertion failure on scope exit. |
| `cancelAndIgnoreRemainingEvents()` | Stop, drain silently. The default tail of any hot-flow test. |
| `cancelAndConsumeRemainingEvents()` | Returns `List<Event<T>>` for ad-hoc inspection. |

## Hot vs cold flow contract

| Flow type | Completes naturally? | First `awaitItem()` returns | Need `cancel()`? |
|---|---|---|---|
| `flowOf(...)` / `flow { }` (cold) | yes | first emission | no — `awaitComplete()` terminates |
| `MutableSharedFlow<T>` (replay = 0) | no | next emission (or hangs) | always |
| `MutableStateFlow<T>` | no | seed value (replay = 1) | always |

(`docs/CORPUS.md` §G; R5.)

CRITICAL: `SharedFlow` and `StateFlow` never call `onCompletion`. `awaitComplete()` on either will hang to the 3-second wall-clock timeout. Always end with `cancel()` (or `cancelAndIgnoreRemainingEvents()`).

`StateFlow.test` always emits the seed value first because `StateFlow` has `replay = 1`. If you do not care about the seed, `skipItems(1)` it.

## turbineScope and testIn — multiple flows in one test

```kotlin
public suspend fun <R> turbineScope(
    timeout: Duration? = null,
    validate: suspend TurbineTestContext.() -> R,
): R

public fun <T> Flow<T>.testIn(
    scope: CoroutineScope,
    timeout: Duration? = null,
    name: String? = null,
): ReceiveTurbine<T>
```

Use these when assertions are interleaved between two flows:

```kotlin
runTest(mainRule.dispatcher) {
    turbineScope {
        val items = repo.items.testIn(backgroundScope, name = "items")
        val errors = repo.errors.testIn(backgroundScope, name = "errors")
        assertEquals(0, items.awaitItem())
        assertEquals(emptyList<Throwable>(), errors.expectMostRecentItem())
        items.cancel()
        errors.cancel()
    }
}
```

`testIn` requires explicit `cancel()` (or `awaitComplete()` / `awaitError()`) before the `turbineScope` ends; otherwise the scope assertion throws.

## Standalone `Turbine<T>` — capturing callbacks and one-shot signals

Turbine is not only a `Flow` adapter. The standalone `Turbine<T>` is a send-and-receive channel with the same `ReceiveTurbine` assertion surface — for when the thing under test is **not** a `Flow`: a `(T) -> Unit` callback, a listener interface, a `Channel` you own, a hand-rolled fake that needs to record invocations.

```kotlin
public fun <T> Turbine(timeout: Duration? = null, name: String? = null): Turbine<T>

public interface Turbine<T> : ReceiveTurbine<T> {
    public val isClosed: Boolean
    public fun add(item: T)                       // producer side — push an item
    public fun close(cause: Throwable? = null)    // signal Complete (or Error if cause != null)
    public fun asChannel(): Channel<T>            // hand the producer a plain SendChannel
}
```

(`docs/CORPUS.md` §G; R5.) It turns "was this callback invoked with X, and only that?" into the same `awaitItem()` vocabulary:

```kotlin
@Test fun emitsClickedId() = runTest(mainRule.dispatcher) {
    val clicks = Turbine<Long>()
    val adapter = ItemAdapter(onClick = clicks::add)        // production code calls onClick(id)

    adapter.bind(item)
    adapter.simulateClick()

    assertEquals(42L, clicks.awaitItem())
    clicks.ensureAllEventsConsumed()                        // no extra invocations slipped through
}
```

`ensureAllEventsConsumed()` asserts nothing is left buffered — the standalone-Turbine equivalent of the check `flow.test { }` runs implicitly on block exit. A standalone `Turbine` never auto-completes, so `awaitComplete()` on one hangs to the timeout unless you `close()` it; treat it like a hot flow.

### Synchronous (non-`suspend`) consumption: `takeItem` / `takeComplete` / `takeError`

When the assertion site cannot suspend — a plain `@Test` body after the producing coroutine already ran, an `assert` inside a non-`suspend` callback — `ReceiveTurbine` exposes non-suspending takers that require the event to be buffered already and throw if it is not:

```kotlin
public fun takeItem(): T
public fun takeComplete()
public fun takeError(): Throwable
```

```kotlin
@Test fun synchronousDrain() = runTest(mainRule.dispatcher) {
    val turbine = vm.uiState.testIn(backgroundScope)
    advanceUntilIdle()                              // let the producer run to quiescence (virtual time)
    assertEquals(Idle, turbine.takeItem())          // no suspension — the value is already buffered
    assertEquals(Loaded, turbine.takeItem())
    turbine.cancel()
}
```

Prefer suspending `awaitItem()` inside `runTest` / `flow.test` blocks; reach for `takeItem()` only when the call site genuinely cannot suspend.

## Turbine timeout vs runTest virtual time

Turbine's 3-second default is **wall clock**, never virtual. `runTest`'s 60-second default is also wall clock — but `advanceTimeBy(...)` inside the test body manipulates `TestCoroutineScheduler` virtual time, which Turbine does not see.

Implication: a test using `delay(10.minutes)` finishes in microseconds because `runTest` skips the delay. But a `flow.test { awaitItem() }` that waits for an emission whose source is real network I/O still observes real wall-clock time. If the wait is genuinely longer than 3 seconds in real time, override:

```kotlin
flow.test(timeout = 10.seconds) { … }            // per-call

withTurbineTimeout(10.seconds) {                 // scoped
    flow.test { … }
}
```

## Decision matrix — Turbine vs raw `flow.toList()`

| Scenario | Pick |
|---|---|
| Cold flow that completes; "the full list equals X" | `flow.toList()` (shorter) |
| Hot `StateFlow` / `SharedFlow` | Turbine (handles cancellation) |
| Need to assert intermediate state then trigger more emissions | Turbine |
| Need to assert "no more events for a while" | Turbine (`expectNoEvents`) |
| Need to assert error type / completion | Turbine (`awaitError`, `awaitComplete`) |
| Multiple flows interleaved in one test | Turbine (`turbineScope`/`testIn`) |
| Multiplatform test on JS/Native | Both work; Turbine is easier |
| Zero extra dependency | `flow.toList()` |

(R5.)

Rule of thumb: prefer Turbine for any UI-state / ViewModel test (almost always `StateFlow`-driven), and `flow.toList()` for repository/transformation tests of cold flows that complete deterministically.

## Patterns

### Pattern: WRONG — `flow.toList()` on a SharedFlow

```kotlin
// WRONG
@Test fun events() = runTest {
    val events: SharedFlow<Event> = vm.events
    val collected = events.toList()                 // never returns
    assertEquals(listOf(LoggedIn), collected)
}
// WRONG because: SharedFlow never completes. toList() suspends until the upstream
// completes, which it never does. The test hangs to the 60s runTest timeout.
```

```kotlin
// RIGHT
@Test fun events() = runTest(mainRule.dispatcher) {
    vm.events.test {
        vm.login()
        assertEquals(LoggedIn, awaitItem())
        cancelAndIgnoreRemainingEvents()             // hot flow — must cancel
    }
}
```

### Pattern: WRONG — forgetting StateFlow's seed value

```kotlin
// WRONG
@Test fun stateFlowEmits() = runTest(mainRule.dispatcher) {
    vm.uiState.test {
        vm.refresh()
        assertEquals(Loading, awaitItem())           // FAIL: actually Idle (seed)
        cancel()
    }
}
// WRONG because: StateFlow has replay = 1. The collector immediately receives the
// current value (Idle) before any new emission. The first awaitItem() returns
// the seed, not the post-refresh state.
```

```kotlin
// RIGHT — assert the seed
@Test fun stateFlowEmits() = runTest(mainRule.dispatcher) {
    vm.uiState.test {
        assertEquals(Idle, awaitItem())              // seed
        vm.refresh()
        assertEquals(Loading, awaitItem())
        assertEquals(Success(items), awaitItem())
        cancelAndIgnoreRemainingEvents()
    }
}

// RIGHT — skip the seed if you do not care
@Test fun stateFlowEmits_skipSeed() = runTest(mainRule.dispatcher) {
    vm.uiState.test {
        skipItems(1)
        vm.refresh()
        assertEquals(Loading, awaitItem())
        assertEquals(Success(items), awaitItem())
        cancelAndIgnoreRemainingEvents()
    }
}
```

### Pattern: WRONG — `awaitComplete()` on a hot flow

```kotlin
// WRONG
@Test fun events() = runTest {
    vm.events.test {
        vm.login()
        assertEquals(LoggedIn, awaitItem())
        awaitComplete()                              // hangs 3s, then TurbineAssertionError
    }
}
// WRONG because: SharedFlow never completes. awaitComplete() waits 3 wall-clock
// seconds (Turbine default) and then throws. The test is slow AND fails.
```

```kotlin
// RIGHT
@Test fun events() = runTest(mainRule.dispatcher) {
    vm.events.test {
        vm.login()
        assertEquals(LoggedIn, awaitItem())
        cancelAndIgnoreRemainingEvents()
    }
}
```

### Pattern: conflated StateFlow — use `expectMostRecentItem`

```kotlin
@Test fun finalState() = runTest(mainRule.dispatcher) {
    vm.uiState.test {
        skipItems(1)                                 // seed
        vm.flipFast()                                // posts 5 transitions in <1ms
        advanceUntilIdle()                           // virtual; let coroutines settle
        // StateFlow conflates — only the latest value is kept.
        // awaitItem() may see anything from the burst; expectMostRecentItem is deterministic.
        assertEquals(Final, expectMostRecentItem())
        cancelAndIgnoreRemainingEvents()
    }
}
```

### Pattern: assert "no more events"

```kotlin
@Test fun noSpuriousEmissions() = runTest(mainRule.dispatcher) {
    vm.uiState.test {
        skipItems(1)
        vm.refresh()
        assertEquals(Loading, awaitItem())
        assertEquals(Success(items), awaitItem())
        advanceTimeBy(5.seconds)                     // virtual — flush any timers
        runCurrent()
        expectNoEvents()                             // confirms quiescence
        cancelAndIgnoreRemainingEvents()
    }
}
```

### Pattern: cold flow that completes — Turbine vs toList

```kotlin
// FINE — toList() is shorter for terminal cold flows
@Test fun coldList() = runTest {
    val collected = flowOf(1, 2, 3).toList()
    assertEquals(listOf(1, 2, 3), collected)
}

// ALSO FINE — Turbine if you need intermediate-state assertions
@Test fun coldTurbine() = runTest {
    flowOf(1, 2, 3).test {
        assertEquals(1, awaitItem())
        assertEquals(2, awaitItem())
        assertEquals(3, awaitItem())
        awaitComplete()                              // cold flow — completes naturally
    }
}
```

### Pattern: two flows interleaved with `turbineScope`

```kotlin
@Test fun interleaved() = runTest(mainRule.dispatcher) {
    turbineScope {
        val state = vm.uiState.testIn(backgroundScope, name = "state")
        val events = vm.events.testIn(backgroundScope, name = "events")
        assertEquals(Idle, state.awaitItem())
        vm.login()
        assertEquals(Loading, state.awaitItem())
        assertEquals(LoggedIn, events.awaitItem())
        assertEquals(Success, state.awaitItem())
        state.cancel(); events.cancel()
    }
}
```

## Mandatory rules

- **MUST** wrap any `StateFlow` / `SharedFlow` assertion in `flow.test { … cancelAndIgnoreRemainingEvents() }`. **MUST NOT** call `flow.toList()` on a hot flow — the test hangs.
- **MUST** call `cancel()` / `cancelAndIgnoreRemainingEvents()` at the end of every `test { }` block over a hot flow. `awaitComplete()` is wrong for hot flows — they never complete.
- **MUST** account for `StateFlow`'s seed value: the first `awaitItem()` returns the current value (replay = 1). Either assert it or `skipItems(1)`.
- **MUST** depend on Turbine from `testImplementation` only. **MUST NOT** put `app.cash.turbine` on `implementation`.
- **MUST** override `withTurbineTimeout(...)` (or per-call `timeout =`) when the awaited emission genuinely takes more than 3 wall-clock seconds. The 3 s default is wall clock and is independent from `runTest` virtual time.
- **MUST** use `cancel()` (NOT `awaitComplete()`) inside `testIn` blocks before `turbineScope` ends; otherwise the scope assertion throws.
- **MUST NOT** mix Turbine timeouts with `runTest`'s virtual clock semantics. `advanceTimeBy(10.minutes)` is virtual; Turbine's 3-second wait is real.
- **PREFERRED:** `expectMostRecentItem()` for deterministic "final state" assertions on conflated `StateFlow` where intermediate values may be collapsed.
- **PREFERRED:** raw `flow.toList()` for cold flows that complete deterministically — Turbine is overhead in that case.
- **PREFERRED:** name turbines (`flow.test(name = "uiState")`, `flow.testIn(scope, name = "events")`) when more than one runs in a test — error messages cite the name.
- **PREFERRED:** the standalone `Turbine<T>()` (with `add` / `close` / `ensureAllEventsConsumed`) over a manually accumulated `MutableList` when verifying a `(T) -> Unit` callback or a listener was invoked — it gives the same `awaitItem()` diagnostics and the unconsumed-event check.
- **PREFERRED:** suspending `awaitItem()` inside `runTest` / `flow.test` blocks over the non-suspending `takeItem()` / `takeComplete()` / `takeError()`; reach for the takers only when the call site genuinely cannot suspend.

## Verification

- [ ] No `flow.toList()` call exists on any `StateFlow` or `SharedFlow` (`grep -rn 'toList()' src/test` cross-checked against flow types).
- [ ] Every `flow.test { … }` over a hot flow ends with `cancel()` or `cancelAndIgnoreRemainingEvents()`.
- [ ] No `awaitComplete()` is called on a `StateFlow` / `SharedFlow`.
- [ ] `StateFlow` tests either assert the seed via `awaitItem()` or `skipItems(1)` before further assertions.
- [ ] Turbines that may legitimately wait > 3 s use `withTurbineTimeout(...)` or per-call `timeout = N.seconds`.
- [ ] Multi-flow tests use `turbineScope { … testIn(backgroundScope) … }` rather than nested `test { }` blocks.
- [ ] Callback / listener assertions use a standalone `Turbine<T>()` (with `ensureAllEventsConsumed()`), not an ad-hoc `MutableList`.
- [ ] CI run completes 50 iterations without `TurbineAssertionError` flakes.

## References

- Cash App Turbine — README and API: https://github.com/cashapp/turbine
- Cash App Turbine — releases: https://github.com/cashapp/turbine/releases
- Android Developers — Test Kotlin Flow on Android: https://developer.android.com/kotlin/flow/test
- Android Developers — StateFlow & SharedFlow: https://developer.android.com/kotlin/flow/stateflow-and-sharedflow
- JetBrains — Asynchronous Flow: https://kotlinlang.org/docs/flow.html
- Research: `tasks/research/R5-coroutines-test-turbine.md` — Turbine API surface, hot/cold contract, comparison with `toList()`.
- `docs/CORPUS.md` §G.1 — Gradle coordinates and version pin.
- Sibling: `../testing-coroutines-with-runtest/SKILL.md` — `runTest`, `MainDispatcherRule`, virtual time foundations Turbine sits on top of.
- Sibling: `../../mocking/mocking-with-mockito/SKILL.md` — mocking suspend dependencies that produce flows.
- Sibling: `../../mocking/mocking-with-mockk/SKILL.md` — `coEvery { repo.items() } returns flowOf(...)`.
- Sibling: `../../runner/configuring-junit4-on-android/SKILL.md` — JUnit4 plumbing.
- Sibling: `../../robolectric/using-robolectric-correctly/SKILL.md` — when the SUT needs Android framework on the JVM.
- Cross-set: `../../../fundamentals/strategies/applying-testing-strategies/SKILL.md` — small/medium/big sizing for Flow tests.
- Cross-set: `../../../fundamentals/strategies/organizing-test-source-sets/SKILL.md` — where Flow tests belong (`src/test/`).
- Cross-set: `../../../compose/synchronization/controlling-the-test-clock/SKILL.md` — Compose's `MainTestClock` is independent of Turbine's wall clock.
- Cross-set: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md` — Turbine works on-device too, with the same API.
- Cross-set: `../../../kotlin/kotlin-test/writing-tests-with-kotlin-test/SKILL.md` — `awaitItem()` paired with `kotlin.test`'s `assertEquals` / `assertIs`; the assertion vocabulary inside `flow.test { }`.
