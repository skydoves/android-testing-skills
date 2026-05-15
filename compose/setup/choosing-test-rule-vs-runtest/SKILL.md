---
name: choosing-test-rule-vs-runtest
description: Use this skill to pick the correct Compose UI test entry point. Compares `createComposeRule()`, `createAndroidComposeRule<A>()`, `createEmptyComposeRule()`, `runComposeUiTest { }`, and `runAndroidComposeUiTest<A> { }`, plus the v1 vs v2 split (`UnconfinedTestDispatcher` vs `StandardTestDispatcher`). Encodes the rule that mixing `runComposeUiTest { }` and a `ComposeTestRule` in the same test is forbidden, that `setContent` may only run once, and that `createEmptyComposeRule()` returns `ComposeTestRule` (no `setContent`). Use when the user asks "ComposeTestRule vs ComposeUiTest", reports `IllegalStateException: setContent can only be called once`, mentions `runComposeUiTest`, the v2 deprecation warning, `effectContext`, custom `ComponentActivity`, or "host tests need a coroutine scope".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - createComposeRule
  - createAndroidComposeRule
  - createEmptyComposeRule
  - runComposeUiTest
  - ComposeTestRule
  - ComposeUiTest
  - v2-test-api
  - StandardTestDispatcher
  - effect-context
---

# Choosing Test Rule vs runComposeUiTest — One Entry Point Per Test

Compose ships two parallel test entry points: a JUnit4 `TestRule` (`createComposeRule()` and friends) and a multiplatform suspending lambda (`runComposeUiTest { }`). Each one independently sets up the recomposer, `MainTestClock`, and `IdlingResource`. Mixing them in a single test produces double-environment bugs that are hard to diagnose. This skill picks the right one and surfaces the v1 → v2 deprecation that bites every existing codebase.

## When to use this skill

- The user is starting a new Compose UI test and is choosing between `createComposeRule()` and `runComposeUiTest { }`.
- The IDE shows a deprecation `WARNING` on `import androidx.compose.ui.test.junit4.createComposeRule` or `import androidx.compose.ui.test.runComposeUiTest`.
- The user reports tests passing on v1 entry points but breaking after migrating because `LaunchedEffect`s no longer run synchronously.
- The user has a custom `ComponentActivity` subclass and is unsure between `createAndroidComposeRule` and `createEmptyComposeRule`.
- The user reports `IllegalStateException: setContent can only be called once per ComposeTestRule` or "the launched Activity already calls setContent".
- The user is writing Compose Multiplatform (KMP) tests and needs the suspending entry point.

## When NOT to use this skill

- The dependencies do not yet compile — start with `./configuring-test-dependencies/SKILL.md`.
- The choice is host vs device, not rule vs lambda — see `./setting-up-host-vs-device-tests/SKILL.md`.
- The test compiles and runs but flakes on idle/animation — see `../../synchronization/synchronizing-with-idle/SKILL.md` and `../../synchronization/testing-animations-deterministically/SKILL.md`.

## Prerequisites

- `androidx.compose.ui:ui-test`, `androidx.compose.ui:ui-test-junit4`, and (for `createComposeRule()` / `runComposeUiTest { }`) `androidx.compose.ui:ui-test-manifest` on the right configurations — see `./configuring-test-dependencies/SKILL.md`.
- A test class with JUnit4 on the classpath if the developer chooses the rule path.
- Working knowledge that `MainTestClock` and the recomposer share one `kotlinx.coroutines.test.TestCoroutineScheduler`.

## Workflow

- [ ] **1. Decide JUnit4 rule vs suspending lambda.**

  | Path | Choose when |
  |---|---|
  | `createComposeRule()` / `createAndroidComposeRule<A>()` | The codebase already uses JUnit4 rules; the developer wants one rule chained with `RuleChain` (e.g. `HiltAndroidRule`); the test is Android-only. |
  | `runComposeUiTest { }` / `runAndroidComposeUiTest<A> { }` | Compose Multiplatform; the developer wants `effectContext: CoroutineContext` to inject a `TestDispatcher`; the test is suspending end-to-end. |

  Both surfaces expose the same matchers, finders, actions, and `MainTestClock`. The rule path is more familiar to existing Android engineers; the lambda path is the long-term recommendation for multiplatform.

- [ ] **2. Pick the right rule constructor for the JUnit4 path.**

  | Constructor | Returns | Hosts | `setContent` available? |
  |---|---|---|---|
  | `createComposeRule()` | `ComposeContentTestRule` | `androidx.activity.ComponentActivity` (from `ui-test-manifest`) | Yes |
  | `createAndroidComposeRule<A : ComponentActivity>()` | `AndroidComposeTestRule<ActivityScenarioRule<A>, A>` | Custom Activity `A` (developer-declared in test manifest) | Yes |
  | `createEmptyComposeRule()` | `ComposeTestRule` (NOT `ComposeContentTestRule`) | None — developer launches their own scenario | **No** |

  Source: `compose/ui/ui-test-junit4/src/jvmAndAndroidMain/kotlin/androidx/compose/ui/test/junit4/ComposeTestRule.jvmAndAndroid.kt` declares `interface ComposeContentTestRule : ComposeTestRule { fun setContent(...) }`; `createEmptyComposeRule()` returns the parent type only.

- [ ] **3. Pick the right suspending entry point for the lambda path.**

  | Function | Receiver | Hosts |
  |---|---|---|
  | `runComposeUiTest { }` | `ComposeUiTest` | `ComponentActivity` |
  | `runAndroidComposeUiTest<A> { }` | `AndroidComposeUiTest<A>` (adds `val activity: A?`) | Custom Activity `A` |
  | `runEmptyComposeUiTest { }` | `ComposeUiTest` | None — `setContent` throws `IllegalStateException` |

  `runComposeUiTest` and `runAndroidComposeUiTest` accept `effectContext: CoroutineContext = EmptyCoroutineContext`, `runTestContext: CoroutineContext = EmptyCoroutineContext`, and `testTimeout: Duration = 60.seconds`. **`runEmptyComposeUiTest` is the exception — it accepts only `block: ComposeUiTest.() -> Unit`** (no scheduling parameters); its purpose is to let the test launch its own `ActivityScenario` inside the block. Source: `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeUiTest.android.kt:184, 232, 352` (v1) and the v2 actuals at `…/v2/ComposeUiTest.android.kt:75, 122, 238`.

- [ ] **4. Prefer the v2 entry points over v1.** This is non-negotiable for new code. The v1 forms (`androidx.compose.ui.test.junit4.createComposeRule`, `androidx.compose.ui.test.runComposeUiTest`) carry `@Deprecated(level = DeprecationLevel.WARNING)`. The deprecation message on `runComposeUiTest` (verbatim from `ComposeUiTest.android.kt`) is:

```text
"Use `androidx.compose.ui.test.v2.runComposeUiTest` instead. The v2 APIs align with
 standard coroutine behavior by queuing tasks rather than executing them
 immediately. Tests relying on immediate execution may require explicit
 synchronization. Please refer to the migration guide for more details."
```

`createComposeRule()` carries an analogous but separate deprecation message (in `ComposeTestRule.jvmAndAndroid.kt:347-355`) pointing at `androidx.compose.ui.test.junit4.v2.createComposeRule`. The two messages are NOT byte-identical — quote whichever applies to the API the developer is actually migrating.

  Migration mapping:

  | v1 (deprecated WARNING) | v2 (recommended) |
  |---|---|
  | `androidx.compose.ui.test.junit4.createComposeRule` | `androidx.compose.ui.test.junit4.v2.createComposeRule` |
  | `androidx.compose.ui.test.junit4.createAndroidComposeRule` | `androidx.compose.ui.test.junit4.v2.createAndroidComposeRule` |
  | `androidx.compose.ui.test.junit4.createEmptyComposeRule` | `androidx.compose.ui.test.junit4.v2.createEmptyComposeRule` |
  | `androidx.compose.ui.test.runComposeUiTest` | `androidx.compose.ui.test.v2.runComposeUiTest` |
  | `androidx.compose.ui.test.runAndroidComposeUiTest` | `androidx.compose.ui.test.v2.runAndroidComposeUiTest` |
  | `androidx.compose.ui.test.runEmptyComposeUiTest` | `androidx.compose.ui.test.v2.runEmptyComposeUiTest` |

  Behavior delta: v1 uses `UnconfinedTestDispatcher` (eager), v2 uses `StandardTestDispatcher` (queued). After migration, tests that relied on a `LaunchedEffect` / `rememberCoroutineScope` block running synchronously may need an explicit `mainClock.advanceTimeBy(0)` or `runCurrent()` to drain queued work. This is the only common breaking change.

- [ ] **5. NEVER mix the two surfaces in the same test.** Both manage independent test environments. The KDoc on `runComposeUiTest` (lines 157-160 of `ComposeUiTest.android.kt`) is explicit:

```text
"Keeping a reference to the [ComposeUiTest] outside of this function is an error. Also avoid
 using [androidx.compose.ui.test.junit4.ComposeTestRule] (e.g., createComposeRule) inside
 [runComposeUiTest][block] or any of their respective variants. Since these APIs independently
 manage the test environment, mixing them may lead to unexpected behavior."
```

  Symptoms of accidental mixing: doubled `setContent` calls, recompositions running on the wrong scheduler, `MainTestClock` advancing in one environment but not the other, `waitForIdle` returning instantly because it queries the wrong recomposer.

- [ ] **6. Call `setContent` exactly once per test.** Both `ComposeContentTestRule.setContent` and `ComposeUiTest.setContent` throw `IllegalStateException` on the second call. Recompose with state mutations, do NOT re-call `setContent`.

- [ ] **7. If the launched Activity calls `setContent` itself, do NOT call `composeTestRule.setContent`.** The Activity has already installed Compose content; the rule call would override and orphan the previous tree. Use `createAndroidComposeRule<MyActivity>()` and read the existing tree directly via finders. The KDoc at line 200-204 of `ComposeUiTest.android.kt` warns: *"if the Activity sets content during its launch, you cannot use `setContent` on the ComposeUiTest anymore as this would override the content and can lead to subtle bugs."*

- [ ] **8. When custom coroutine semantics are needed, use `effectContext`.** The lambda path takes a `CoroutineContext` that becomes the parent of `LaunchedEffect`s and `rememberCoroutineScope` scopes. From `ComposeUiTest.android.kt`: *"If this context contains a TestDispatcher or TestCoroutineScheduler (in that order), it will be used for composition and the MainTestClock."* Pass a shared `TestDispatcher` to coordinate the test body's coroutine scheduler with composition's scheduler. The JUnit4 rule has no equivalent first-class hook — that is a real reason to prefer the lambda path when the production code does heavy `LaunchedEffect` work.

- [ ] **9. Default `testTimeout` is 60 seconds.** From the same file, `testTimeout: Duration = 60.seconds`. Tests that exceed it throw `AndroidComposeUiTestTimeoutException`. Override per-test only; never globally raise it to mask flakes.

## Patterns

### Pattern: WRONG vs RIGHT — v1 vs v2 import

```kotlin
// WRONG — emits @Deprecated WARNING and uses UnconfinedTestDispatcher
import androidx.compose.ui.test.junit4.createComposeRule

class MyTest {
    @get:Rule val rule = createComposeRule()
}
// WRONG because: v1 dispatches LaunchedEffect work eagerly. After bumping
// kotlinx-coroutines-test the same test will produce different observable
// behavior than runTest { } in the rest of the codebase.
```

```kotlin
// RIGHT — v2, StandardTestDispatcher, no deprecation warning
import androidx.compose.ui.test.junit4.v2.createComposeRule

class MyTest {
    @get:Rule val rule = createComposeRule()
}
```

### Pattern: WRONG vs RIGHT — mixing rule and lambda

```kotlin
// WRONG
class MyTest {
    @get:Rule val rule = createComposeRule()      // env #1

    @Test
    fun bad() = runComposeUiTest {                // env #2 — independent recomposer + clock
        rule.setContent { App() }                  // sets content in env #1
        onNodeWithTag("save").performClick()       // queries env #2 — finds nothing
    }
}
// WRONG because: each environment owns its own test scheduler and idling resource.
// The KDoc forbids this configuration explicitly.
```

```kotlin
// RIGHT — pick one
class MyTest {
    @Test
    fun good() = runComposeUiTest {
        setContent { App() }
        onNodeWithTag("save").performClick()
    }
}
```

### Pattern: WRONG vs RIGHT — `createEmptyComposeRule` misuse

```kotlin
// WRONG
@get:Rule val rule = createEmptyComposeRule()
@Test fun broken() {
    rule.setContent { App() }   // compile error — ComposeTestRule has no setContent
}
// WRONG because: createEmptyComposeRule() returns ComposeTestRule, not ComposeContentTestRule.
// It is for tests that launch their own ActivityScenario.
```

```kotlin
// RIGHT — launch your own scenario, then query
@get:Rule val rule = createEmptyComposeRule()
@Test fun ok() {
    ActivityScenario.launch(MyActivity::class.java).use {
        // MyActivity.onCreate calls setContent { App() }
        rule.onNodeWithTag("save").performClick()
    }
}
```

### Pattern: WRONG vs RIGHT — Activity already sets content

```kotlin
// WRONG
@get:Rule val rule = createAndroidComposeRule<MainActivity>()
@Test fun bad() {
    rule.setContent { App() }     // throws IllegalStateException — MainActivity.onCreate already set content
}
```

```kotlin
// RIGHT
@get:Rule val rule = createAndroidComposeRule<MainActivity>()
@Test fun good() {
    rule.onNodeWithTag("save").performClick()   // query content the Activity already installed
}
```

### Pattern: lambda path with custom dispatcher

```kotlin
@OptIn(ExperimentalTestApi::class, ExperimentalCoroutinesApi::class)
@Test
fun launchedEffectFlow() = runComposeUiTest(
    effectContext = StandardTestDispatcher(),     // share scheduler with composition
) {
    setContent { ScreenWithLaunchedEffect() }
    mainClock.advanceTimeBy(0)                     // drain queued LaunchedEffect work
    onNodeWithTag("counter").assertTextEquals("1")
}
```

### Pattern: rule chain with Hilt + Compose

```kotlin
@get:Rule(order = 0) val hilt = HiltAndroidRule(this)
@get:Rule(order = 1) val compose = createAndroidComposeRule<HiltTestActivity>()

@Test fun feed() {
    hilt.inject()
    compose.onNodeWithTag("feed_list").assertIsDisplayed()
}
```

This is the canonical reason to keep the JUnit4 rule path: `RuleChain` ordering. The lambda path has no equivalent — the developer has to manage Hilt initialization manually inside the suspending block.

## Mandatory rules

- **MUST** prefer v2 imports (`androidx.compose.ui.test.junit4.v2.*`, `androidx.compose.ui.test.v2.*`) over v1 for new code. The v1 forms are `@Deprecated(level = WARNING)`.
- **MUST NOT** mix `runComposeUiTest { }` and a `@get:Rule` `ComposeTestRule` in the same test class. They manage independent test environments.
- **MUST NOT** call `setContent` more than once per test. Both surfaces throw on the second call.
- **MUST NOT** call `composeTestRule.setContent` when the launched Activity has already called `setContent` itself. Use `createAndroidComposeRule<A>()` and query the existing tree.
- **MUST** match the rule constructor to the host requirements: `createComposeRule()` for `ComponentActivity`, `createAndroidComposeRule<A>()` for a custom Activity, `createEmptyComposeRule()` only when the test launches its own `ActivityScenario`.
- **MUST** declare any custom Activity used by `createAndroidComposeRule<A>()` in `src/androidTest/AndroidManifest.xml` (or `src/debug/AndroidManifest.xml`) — see `./configuring-test-dependencies/SKILL.md`.
- **PREFERRED:** when production code uses heavy `LaunchedEffect`s or `rememberCoroutineScope`, use the lambda path with `effectContext = StandardTestDispatcher()` so composition and the test body share one scheduler.
- **PREFERRED:** rely on `mainClock.advanceTimeBy(0)` (test clock) rather than `runOnIdle { }` (wall clock + idle wait) to drain v2 queued work — see `../../synchronization/testing-animations-deterministically/SKILL.md` for the autoAdvance contract.

## Verification

- [ ] No `import androidx.compose.ui.test.junit4.createComposeRule` (v1) remains in new code; v2 path imported instead.
- [ ] No `@Deprecated` `WARNING` from the IDE on the test entry point.
- [ ] The test class either uses a `@get:Rule` `ComposeTestRule` OR a `runComposeUiTest { }` block, never both.
- [ ] `setContent` is called exactly once per test method (zero times when the Activity sets content itself).
- [ ] `createEmptyComposeRule()` only appears in tests that explicitly launch their own `ActivityScenario`.
- [ ] If `effectContext` is passed, the test compiles with `@OptIn(ExperimentalTestApi::class)` (or the v2 equivalent) and `mainClock.advanceTimeBy(0)` is added where queued work needs draining.

## References

- Compose testing overview (Android Developers): https://developer.android.com/develop/ui/compose/testing
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- Compose Multiplatform testing: https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-test.html
- Compose UI release notes: https://developer.android.com/jetpack/androidx/releases/compose-ui
- `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeUiTest.android.kt` — `runComposeUiTest`, `runAndroidComposeUiTest`, `runEmptyComposeUiTest`, plus the v1 deprecation message. KDocs at lines 157-160, 204, 247, 338 forbid mixing rule + lambda.
- `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/v2/ComposeUiTest.android.kt` — v2 `runComposeUiTest` actuals using `StandardTestDispatcher`.
- `compose/ui/ui-test-junit4/src/jvmAndAndroidMain/kotlin/androidx/compose/ui/test/junit4/ComposeTestRule.jvmAndAndroid.kt` — `interface ComposeTestRule`, `interface ComposeContentTestRule : ComposeTestRule { fun setContent(...) }`.
- `compose/ui/ui-test-junit4/src/androidMain/kotlin/androidx/compose/ui/test/junit4/AndroidComposeTestRule.android.kt` — actual `createComposeRule()`, `createAndroidComposeRule<A>()`, `createEmptyComposeRule()`.
- `compose/ui/ui-test-junit4/src/androidMain/kotlin/androidx/compose/ui/test/junit4/v2/AndroidComposeTestRule.android.kt` — v2 actuals (StandardTestDispatcher default).
