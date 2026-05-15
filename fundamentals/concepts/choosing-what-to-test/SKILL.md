---
name: choosing-what-to-test
description: Use this skill to pick which behaviors to cover in an Android test suite using Google's five-category state vocabulary plus the explicit "what NOT to test" list from `/training/testing/fundamentals/what-to-test`. Covers state on screen, state held in memory (ViewModels), persisted state (DB / DataStore / files), other state (system bars / system services), and errors / edge cases. Includes Google's verbatim "Tests to Avoid" list (framework entry points such as activities / fragments / services should not have business logic) and an edge-case mining checklist. Use when the user asks "what should I write tests for", "should I unit-test this Activity", "do I need a test for this util class", "what edge cases am I missing", or "should I test the ViewModel or the Composable".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - what-to-test
  - test-coverage
  - viewmodel-tests
  - data-layer-tests
  - edge-cases
  - tests-to-avoid
  - ui-tests
  - compose-testing
  - test-strategy
---

# Choosing What to Test — Cover State, Skip Framework Entry Points

Android test suites under-cover business logic and over-cover framework glue. Google's `/training/testing/fundamentals/what-to-test` page sorts test value by *what state the code touches* and explicitly names the categories of tests to avoid. This skill encodes those categories and the edge-case checklist so the agent can advise concretely instead of saying "test everything important".

## When to use this skill

- The user asks "what should I write tests for in this module" or "do I need a test for X".
- The user asks whether to unit-test an `Activity`, `Fragment`, or `Service` (answer: usually no — see "What NOT to test").
- The user is reviewing a PR and the test gap is unclear.
- The user asks "what edge cases am I missing for this function".
- The user asks where the testing effort should land between ViewModel, Repository, and Composable.

## When NOT to use this skill

- The user already knows *what* to test and is choosing the test scope (small/medium/big) — use `../understanding-the-testing-pyramid/SKILL.md`.
- The user is choosing between fakes / mocks / stubs — use `../../doubles/picking-test-doubles/SKILL.md`.
- The user is wiring source sets (`src/test/` vs `src/androidTest/`) — use `../../strategies/organizing-test-source-sets/SKILL.md`.

## Prerequisites

- A module with at least one `ViewModel`, repository, or screen.
- Familiarity with the Android architecture layers (UI / Domain / Data) so the agent can map "data layer" to a real path.
- The 3-layer pyramid framing from `../understanding-the-testing-pyramid/SKILL.md`.

## Five categories of state to cover

The five state categories below are the synthesis the user asked for. They map onto the verbatim sections of `developer.android.com/training/testing/fundamentals/what-to-test` (the page itself uses layer-style headers — "Essential unit tests", "UI tests", "Testing Edge Cases" — but the underlying axis is *what kind of state changes*).

### 1. State on screen — what the user observes right now

What is rendered: a button label, a list count, an enabled/disabled state, a checkbox toggle. Tested via UI tests:

> "**Screen UI tests** check critical user interactions in a single screen. They perform actions such as clicking on buttons, typing in forms, and checking visible states. One test class per screen is a good starting point."
> — `developer.android.com/training/testing/fundamentals/what-to-test`

> "**User flow tests** or **Navigation tests**, covering most common paths. These tests simulate a user moving through a navigation flow."
> — `developer.android.com/training/testing/fundamentals/what-to-test`

For Compose: assert via `onNodeWithTag(...).assertIsDisplayed()` / `assertTextEquals(...)` / `assertIsEnabled()`. For Views: `onView(...).check(matches(isDisplayed()))`. Use the cross-category skill `../../../instrumentation/espresso/writing-espresso-tests/SKILL.md`.

### 2. State held in memory — `ViewModel` and presenter state

`ViewModel.uiState`, in-memory caches, the current selection. Tested via small unit tests:

> "**Unit tests** for **ViewModels**, or presenters."
> — `developer.android.com/training/testing/fundamentals/what-to-test`

A ViewModel test launches the ViewModel with a fake repository, drives an event, and asserts on the exposed `StateFlow` / `LiveData`. See `../../../jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md` and `../../../jvm-tests/coroutines/testing-flows-with-turbine/SKILL.md`.

### 3. Persisted state — DB, DataStore, SharedPreferences, files

What survives process death:

> "**Unit tests** for the **data** layer, especially repositories. Most of the data layer should be platform-independent. Doing so enables test doubles to replace database modules and remote data sources in tests."
> — `developer.android.com/training/testing/fundamentals/what-to-test`

Two flavors:

- **Repository unit tests** — fake the storage source; verify the repository's caching, mapping, error handling.
- **Storage integration tests** — exercise Room / DataStore / SharedPreferences with their real implementations. These are *small instrumented tests* per `/fundamentals` ("**Small instrumented test**: You can verify that your code works well with a framework feature, such as a SQLite database").

### 4. Other state — system bars, system services, alarms, broadcasts

Window-insets state, dark-mode flag, alarm scheduling, background-job state. The `/what-to-test` page does not give this its own header but the `/fundamentals` page covers it under instrumented tests. Test via Robolectric (medium-local) when shadows exist, or instrumented tests when the real system is required.

### 5. Errors and edge cases

> "Unit tests should focus on both normal and edge cases. Edge cases are uncommon scenarios that human testers and larger tests are unlikely to catch. Examples include the following:
> - Math operations using negative numbers, zero, and boundary conditions.
> - All the possible network connection errors.
> - Corrupted data, such as malformed JSON.
> - Simulating full storage when saving to a file.
> - Object recreated in the middle of a process (such as an activity when the device is rotated)."
> — `developer.android.com/training/testing/fundamentals/what-to-test`

This is the highest-value column in any test budget — humans miss these, big tests miss these, and a small fake-driven unit test catches them in 5 ms.

## What NOT to test (verbatim)

The page is unusually direct here:

> "Some unit tests should be avoided because of their low value:
> - Tests that verify the correct operation of the framework or a library, not your code.
> - Framework entry points such as **activities, fragments, or services** should not have business logic so unit testing shouldn't be a priority. Unit tests for activities have little value, because they would cover mostly framework code and they require a more involved setup. Instrumented tests such as UI tests can cover these classes."
> — `developer.android.com/training/testing/fundamentals/what-to-test`

Operational consequences:

- **MUST NOT** write `Robolectric` unit tests of an `Activity`'s `onCreate` to assert that views are inflated. That is testing the framework.
- **MUST NOT** write a unit test of `RoomDatabase.Builder`'s migration registration. That tests Room.
- **MUST NOT** write a unit test that mocks `Retrofit` to assert that `@GET` paths resolve. That tests Retrofit.
- **MUST NOT** test private implementation details that change every refactor. Test public observable behavior.
- **PREFERRED:** when a class is hard to unit-test, refactor business logic out of it (`Activity` → `ViewModel` → `UseCase` → `Repository`) — the page closes the loop with "With a testable app architecture, the code follows a structure that allows you to easily test different parts of it in isolation." (cited from `/fundamentals`).

The page also flags additional categories briefly:

> "There are more specialized tests such as screenshot tests, performance tests, and monkey tests."
> — `developer.android.com/training/testing/fundamentals/what-to-test`

These are out of scope for the "what to test" decision but should be mentioned when the user asks about visual regressions or render correctness — see the cross-category skill `../../../compose/audit/auditing-compose-test-suite/SKILL.md`.

## Patterns

### Pattern: WRONG vs RIGHT — testing an Activity instead of its ViewModel

```kotlin
// WRONG
@RunWith(AndroidJUnit4::class)
class CheckoutActivityTest {
    @get:Rule val rule = ActivityScenarioRule(CheckoutActivity::class.java)

    @Test fun whenTotalIsNegative_showsError() {
        rule.scenario.onActivity { activity ->
            activity.viewModel.applyCoupon("FREE_BEER_NEGATIVE_PRICE")
            // assert on the activity's TextView contents
            assertEquals("Error", activity.findViewById<TextView>(R.id.error).text)
        }
    }
}
// WRONG because: the test rebuilds the framework lifecycle, the Activity, the inflater,
// and the View graph just to assert business logic that lives in the ViewModel. Per
// /what-to-test: "Unit tests for activities have little value, because they would cover
// mostly framework code".
```

```kotlin
// RIGHT — small, JVM, sub-50 ms
class CheckoutViewModelTest {
    @Test fun applyCoupon_negativePrice_emitsError() = runTest {
        val vm = CheckoutViewModel(FakePricingRepository(allowNegative = true))
        vm.applyCoupon("FREE_BEER_NEGATIVE_PRICE")
        assertEquals(CheckoutUiState.Error("Negative total"), vm.uiState.value)
    }
}
```

The `Activity` still gets a thin big-test (Espresso / Compose-test) that asserts the error string is rendered. The logic lives in the ViewModel test. Two layers, two responsibilities.

### Pattern: WRONG vs RIGHT — over-mocking a util class

```kotlin
// WRONG
class FormatDurationTest {
    private val clock = mockk<Clock>()
    private val locale = mockk<Locale>()
    @Test fun formatsMinutesAndSeconds() {
        every { clock.now() } returns Instant.parse("2024-01-01T00:01:30Z")
        every { locale.language } returns "en"
        assertEquals("1m 30s", formatDuration(90.seconds, clock, locale))
    }
}
// WRONG because: this is a pure-function utility per /what-to-test ("Unit tests for
// utility classes such as string manipulation and math"). Mocks add ceremony with
// no value. Pass real values.
```

```kotlin
// RIGHT
class FormatDurationTest {
    @Test fun formatsMinutesAndSeconds() {
        assertEquals("1m 30s", formatDuration(90.seconds))
    }

    @Test fun handlesZero() {
        assertEquals("0s", formatDuration(0.seconds))
    }

    @Test fun handlesHoursBoundary() {
        assertEquals("1h 0m 0s", formatDuration(3600.seconds))
    }
}
```

### Pattern: WRONG vs RIGHT — skipping the edge cases checklist

```kotlin
// WRONG — only the happy path
class ParseAmountTest {
    @Test fun parsesValidAmount() = assertEquals(12.34, parseAmount("12.34"))
}
```

```kotlin
// RIGHT — edge cases per /what-to-test
class ParseAmountTest {
    @Test fun parsesValidAmount() = assertEquals(12.34, parseAmount("12.34"))
    @Test fun parsesZero() = assertEquals(0.0, parseAmount("0"))
    @Test fun parsesNegative() = assertEquals(-5.0, parseAmount("-5"))
    @Test fun parsesScientific() = assertEquals(1e6, parseAmount("1e6"))
    @Test fun rejectsEmpty() = assertNull(parseAmount(""))
    @Test fun rejectsMalformed() = assertNull(parseAmount("12.34.56"))
    @Test fun rejectsLeadingWhitespace() = assertNull(parseAmount(" 12"))
    @Test fun rejectsLocaleComma() = assertNull(parseAmount("12,34"))  // German format
    @Test fun handlesMaxDouble() = assertEquals(Double.MAX_VALUE, parseAmount("1.7976931348623157E308"))
}
```

## Edge-case mining checklist

Run this checklist before declaring a unit-test pass complete. Items map to the verbatim list on `/training/testing/fundamentals/what-to-test` plus skydoves additions for Compose/coroutines code.

- [ ] **Numeric boundaries** — zero, negative, `Int.MAX_VALUE`, `Long.MIN_VALUE`, `Double.NaN`, `Double.POSITIVE_INFINITY`.
- [ ] **Empty / single / many collections** — `emptyList()`, `listOf(x)`, lists with thousands of items.
- [ ] **Null** — every nullable input. Even if the type system would refuse, deserialization can produce nulls (Moshi / kotlinx-serialization with `coerceInputValues`).
- [ ] **Malformed strings** — empty, whitespace-only, length-1, length-MAX, non-ASCII, RTL text, emoji, ZWJ sequences, surrogate pairs.
- [ ] **Locale variation** — German number format `12,34`, Arabic digits `٠١٢`, Turkish dotted-i.
- [ ] **Network errors** — `IOException`, HTTP 4xx, HTTP 5xx, malformed JSON, truncated stream, timeout.
- [ ] **Storage errors** — disk-full, permission-denied, corrupt SQLite, encrypted preference.
- [ ] **Process death / state restoration** — "Object recreated in the middle of a process (such as an activity when the device is rotated)" per the page. Cover `rememberSaveable`, `SavedStateHandle`, Bundle-restore.
- [ ] **Concurrency** — same operation from two coroutines, cancellation mid-flight, slow producer + fast consumer (use `kotlinx-coroutines-test`).
- [ ] **Time** — DST transitions, leap seconds, midnight crossings, timezone changes, monotonic vs wall clock.
- [ ] **Configuration changes** — dark mode flip, locale change, font scale at 1.5x and 2.0x, RTL layout.

The first half is straight from Google's verbatim list; the second half is operational extension for modern Android.

## Decision matrix — should this code have a test?

```
What does the code touch?                              Test? Where?
-----------------------------------------------------------------------------
Pure function / utility (string, math, mapping)        YES   src/test/ (small)
ViewModel / presenter state                            YES   src/test/ (small)
Repository / use case (fake the storage)               YES   src/test/ (small)
Real Room / DataStore                                  YES   src/androidTest/ (small instrumented)
Composable rendering of state                          YES   src/androidTest/ or src/test/ (Robolectric)
Navigation flow (multi-screen)                         YES   src/androidTest/ (big)
Activity onCreate / inflation logic                    NO    framework code
Fragment onViewCreated wiring                          NO    framework code
Service onStartCommand wiring                          NO    framework code
Retrofit interface (annotation routing)                NO    library code
Hilt module declarations                               NO    library code
Compose Modifier internals                             NO    library code
Generated code                                         NO    generated
Private function with no public observable effect      NO    test the public caller
```

## Mandatory rules

- **MUST** test public observable behavior (state exposed by `ViewModel`, return values of `Repository`, rendered semantics) — never private internals.
- **MUST NOT** unit-test framework entry points (`Activity`, `Fragment`, `Service`) for business logic; refactor the logic out instead. Per `/what-to-test`: "framework entry points ... should not have business logic so unit testing shouldn't be a priority".
- **MUST** cover edge cases from the verbatim list before merging: numeric boundaries, network errors, malformed data, full storage, process recreation.
- **MUST NOT** verify framework or third-party library behavior. "Tests that verify the correct operation of the framework or a library, not your code" is on the explicit avoid list.
- **PREFERRED:** keep one Compose / Espresso big-test per screen for "state on screen" coverage; lean on small ViewModel tests for everything else.
- **PREFERRED:** when a class resists testing, treat that as an architecture signal — the production code is doing too many things or holds state in the wrong layer.

## Verification

- [ ] Every public function on every `ViewModel` and `Repository` in the module has at least one small test.
- [ ] Every screen has at least one big or medium test asserting "state on screen" via `assertIsDisplayed` / `check(matches(isDisplayed()))`.
- [ ] No test class targets only an `Activity` / `Fragment` / `Service` to assert framework-handled behavior.
- [ ] The edge-case checklist above is reviewed for every new public function with non-trivial input domain.
- [ ] No test mocks `Retrofit`, `Room`, `Hilt`, or `Compose` internals to verify their behavior.
- [ ] PR template (or review checklist) requires "what state does this code touch" to be answered before merge.

## References

- `developer.android.com/training/testing/fundamentals/what-to-test` — the canonical source for every quote in this skill.
- `developer.android.com/training/testing/fundamentals` — testable architecture rationale, "Not all unit tests are local" quote.
- `developer.android.com/training/testing/local-tests` — "Caution: Complex mocks should be avoided. Instead, you can use different types of test doubles such as fakes".
- `developer.android.com/training/testing/instrumented-tests` — when state-on-screen tests must be instrumented vs Robolectric.
- `tasks/research/R8-android-fundamentals.md` — verbatim "Essential unit tests" / "Testing Edge Cases" / "Unit Tests to Avoid" excerpts.
- Sibling skills: `../understanding-the-testing-pyramid/SKILL.md`, `../../doubles/picking-test-doubles/SKILL.md`, `../../strategies/applying-testing-strategies/SKILL.md`, `../../strategies/organizing-test-source-sets/SKILL.md`.
- Cross-category: `../../../jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md`, `../../../jvm-tests/coroutines/testing-flows-with-turbine/SKILL.md`, `../../../jvm-tests/mocking/mocking-with-mockito/SKILL.md`, `../../../jvm-tests/mocking/mocking-with-mockk/SKILL.md`, `../../../instrumentation/espresso/writing-espresso-tests/SKILL.md`, `../../../instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md`, `../../../instrumentation/scenarios/launching-fragments-with-fragmentscenario/SKILL.md`, `../../../compose/audit/auditing-compose-test-suite/SKILL.md`.
