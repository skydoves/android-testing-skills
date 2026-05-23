---
name: testing-state-restoration
description: >-
  Use this skill to test `rememberSaveable` round-trips with `StateRestorationTester`, the only supported tool for proving Compose state survives process death and configuration change. Covers the constructor (`StateRestorationTester(rule: ComposeContentTestRule)`), why `restorationTester.setContent { }` MUST replace `rule.setContent { }`, the `state = null` between phases trick that proves restoration actually happened, the 1 MB Bundle cap, and what is NOT exercised (Activity lifecycle, configuration changes, plain `remember`). Use when the developer asks "how do I test rememberSaveable", "test state survives rotation", "state is lost after restore", "Bundle exceeds maximum size", or shows a test that re-reads the same state reference after `emulateSavedInstanceStateRestore` and is confused why nothing changed.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - state-restoration-tester
  - rememberSaveable
  - SaveableStateRegistry
  - emulateSavedInstanceStateRestore
  - process-death
  - configuration-change
  - parcel
---

# Testing State Restoration — `StateRestorationTester` Round-Trips Only `rememberSaveable`

`StateRestorationTester` is the only first-party way to verify that a Composable's state survives a save/restore cycle. It is narrow — it injects a `LocalSaveableStateRegistry`, snapshots whatever registered through it, throws away the composition, and re-composes with the snapshot restored. It does **not** restart the Activity, it does **not** trigger `Application.onCreate`, and it has a hard 1 MB cap on the serialized Bundle. Misunderstanding the scope produces tests that look passing but prove nothing.

## When to use this skill

- The developer is verifying that `rememberSaveable` survives — for a `LazyListState`, a TextField value, a screen toggle, etc.
- The developer asks "how do I test `rememberSaveable` in a unit test?"
- A test calls `emulateSavedInstanceStateRestore()` and the state appears unchanged or null.
- The developer hits `IllegalStateException: Bundle exceeds maximum size (1 MB)`.
- A reviewer asks for a regression test for a save/restore bug fix.

## When NOT to use this skill

- The bug under test is an **Activity recreation** issue (rotation re-creates the Activity, callbacks fire, ViewModels rebind). `StateRestorationTester` does not exercise that path; use Espresso `ActivityScenario.recreate()` plus this skill if both phases need coverage.
- The state is in a `ViewModel` and the developer wants to test `SavedStateHandle`. Use the AndroidX ViewModel testing tools.
- The class skeleton is wrong — fix `../structuring-a-compose-test/SKILL.md` first.

## Prerequisites

- `androidx.compose.ui:ui-test-junit4` on `androidTestImplementation`. `StateRestorationTester` is in package `androidx.compose.ui.test.junit4`.
- A `ComposeContentTestRule` (i.e. one of `createComposeRule()` / `createAndroidComposeRule<A>()`). `createEmptyComposeRule()` is NOT compatible because it returns `ComposeTestRule`, which has no `setContent`.
- The Composable under test stores its state with `rememberSaveable { … }` (or registers via `SaveableStateRegistry`). Plain `remember` does NOT survive the round-trip — the KDoc states it explicitly: "the state stored via regular state() or remember() will be lost." (cited at `compose/ui/ui-test-junit4/src/androidMain/kotlin/androidx/compose/ui/test/junit4/StateRestorationTester.android.kt:67-68`).

## Workflow

- [ ] **1. Construct the tester from the rule.**

```kotlin
import androidx.compose.ui.test.junit4.StateRestorationTester

@get:Rule val rule = createComposeRule(StandardTestDispatcher())
val restorationTester = StateRestorationTester(rule)
```

The constructor signature is `class StateRestorationTester(private val composeTestRule: ComposeContentTestRule)`, file `StateRestorationTester.android.kt:43`.

- [ ] **2. Use `restorationTester.setContent { }` instead of `rule.setContent { }`.** This is non-negotiable: the tester replaces the `LocalSaveableStateRegistry` for the wrapped content. Calling `rule.setContent` directly bypasses the injection and `emulateSavedInstanceStateRestore` will fail with `"setContent should be called first!"` (cited at `StateRestorationTester.android.kt:71`).

- [ ] **3. Hoist the state reference as a nullable `var` declared above `setContent`.** This is the canonical shape — see `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/LazyListTest.kt:890-913`:

```kotlin
@Test
fun stateIsRestored() {
    val restorationTester = StateRestorationTester(rule)
    var state: LazyListState? = null

    restorationTester.setContent {
        state = rememberLazyListState()
        LazyColumn(Modifier.requiredSize(100.dp).testTag(LazyListTag), state = state!!) {
            items(20) { Spacer(Modifier.requiredSize(20.dp).testTag("$it")) }
        }
    }

    rule.onNodeWithTag(LazyListTag).performScrollToIndex(2)

    val (index, scrollOffset) = rule.runOnIdle {
        state!!.firstVisibleItemIndex to state!!.firstVisibleItemScrollOffset
    }

    state = null                                              // (*) clear the reference

    restorationTester.emulateSavedInstanceStateRestore()      // save → dispose → restore

    rule.runOnIdle {
        assertThat(state!!.firstVisibleItemIndex).isEqualTo(index)
        assertThat(state!!.firstVisibleItemScrollOffset).isEqualTo(scrollOffset)
    }
}
```

- [ ] **4. Drive the state to a non-default value.** A test that asserts a default cannot prove restoration occurred. Scroll the list, type into the TextField, toggle the switch.

- [ ] **5. Capture the values to compare against.** Read the state inside `runOnIdle { }` and store them in `val`s.

- [ ] **6. Set `state = null` between the capture and `emulateSavedInstanceStateRestore()`.** This is the load-bearing trick. The first composition assigned a value to `state`; nulling it forces the second `state!!` access to read whatever the **second** composition assigns — proving the second composition really happened. Without nulling, the test asserts on the original instance and would pass even if the round-trip silently restored nothing.

- [ ] **7. Call `emulateSavedInstanceStateRestore()`.** The internal flow (cited at `StateRestorationTester.android.kt:70-77`) is three `runOnIdle` blocks: `saveStateAndDisposeChildren`, `emitChildrenWithRestoredState`, then a no-op block to wait for emission.

- [ ] **8. Re-read the state inside `runOnIdle` and assert equality with the captured values.**

## Patterns

### Pattern: `remember` vs `rememberSaveable`

```kotlin
// WRONG
@Test
fun counterIsRestored() {
    val tester = StateRestorationTester(rule)
    var counter: MutableState<Int>? = null
    tester.setContent {
        counter = remember { mutableStateOf(0) }              // <-- not saveable
        Text("${counter!!.value}")
    }
    rule.runOnIdle { counter!!.value = 7 }
    counter = null
    tester.emulateSavedInstanceStateRestore()
    rule.runOnIdle { assertThat(counter!!.value).isEqualTo(7) }   // FAILS: 0
}
// WRONG because: plain remember is dropped on dispose. The KDoc states "state stored via
// regular state() or remember() will be lost" (StateRestorationTester.android.kt:67-68).
```

```kotlin
// RIGHT
@Test
fun counterIsRestored() {
    val tester = StateRestorationTester(rule)
    var counter: MutableState<Int>? = null
    tester.setContent {
        counter = rememberSaveable { mutableStateOf(0) }      // <-- saveable
        Text("${counter!!.value}")
    }
    rule.runOnIdle { counter!!.value = 7 }
    counter = null
    tester.emulateSavedInstanceStateRestore()
    rule.runOnIdle { assertThat(counter!!.value).isEqualTo(7) }
}
```

### Pattern: `rule.setContent` vs `restorationTester.setContent`

```kotlin
// WRONG
@Test
fun stateIsRestored() {
    val tester = StateRestorationTester(rule)
    rule.setContent { /* … */ }                               // <-- bypasses the tester
    tester.emulateSavedInstanceStateRestore()                 // throws
}
// WRONG because: emulateSavedInstanceStateRestore checks `registry != null` and otherwise
// throws IllegalStateException("setContent should be called first!")
// (StateRestorationTester.android.kt:71). The injection only happens via tester.setContent.
```

```kotlin
// RIGHT
@Test
fun stateIsRestored() {
    val tester = StateRestorationTester(rule)
    tester.setContent { /* … */ }
    tester.emulateSavedInstanceStateRestore()
}
```

### Pattern: re-reading the same instance vs nulling between phases

```kotlin
// WRONG
@Test
fun listScrollIsRestored() {
    val tester = StateRestorationTester(rule)
    lateinit var state: LazyListState
    tester.setContent { state = rememberLazyListState(); /* list */ }
    rule.onNodeWithTag(LazyListTag).performScrollToIndex(5)
    val captured = rule.runOnIdle { state.firstVisibleItemIndex }
    tester.emulateSavedInstanceStateRestore()
    rule.runOnIdle { assertThat(state.firstVisibleItemIndex).isEqualTo(captured) }
}
// WRONG because: `state` still points at the FIRST composition's instance. The assertion
// passes even if the second composition silently created a fresh state and restored nothing.
// Use a nullable var and null it between phases to force the second access to read from the
// post-restoration composition.
```

```kotlin
// RIGHT
@Test
fun listScrollIsRestored() {
    val tester = StateRestorationTester(rule)
    var state: LazyListState? = null
    tester.setContent { state = rememberLazyListState(); /* list */ }
    rule.onNodeWithTag(LazyListTag).performScrollToIndex(5)
    val captured = rule.runOnIdle { state!!.firstVisibleItemIndex }
    state = null                                              // <-- forces re-read
    tester.emulateSavedInstanceStateRestore()
    rule.runOnIdle { assertThat(state!!.firstVisibleItemIndex).isEqualTo(captured) }
}
```

### Pattern: 1 MB Bundle cap

```kotlin
// WRONG
@Test
fun megaListSurvives() {
    val tester = StateRestorationTester(rule)
    tester.setContent {
        rememberSaveable { ByteArray(2 * 1024 * 1024) }       // 2 MB
    }
    tester.emulateSavedInstanceStateRestore()                 // throws IllegalStateException
}
// WRONG because: platformEncodeDecode in StateRestorationTester.android.kt:163 enforces
// `check(bytes.size <= 1024 * 1024) { "Bundle exceeds maximum size (1 MB): ${bytes.size} bytes." }`
// The same cap applies on real devices via Binder transaction limits.
```

```kotlin
// RIGHT — keep saved state small. Persist large data via Room/DataStore, save only an id.
@Test
fun megaListSurvives() {
    val tester = StateRestorationTester(rule)
    tester.setContent {
        rememberSaveable { largeRowsId }                       // a stable key, not the data
    }
    tester.emulateSavedInstanceStateRestore()
}
```

## Mandatory rules

- **MUST** call `restorationTester.setContent { }` (NOT `rule.setContent { }`) for the round-trip to work.
- **MUST** declare the state reference as a nullable `var ... = null` and reassign to `null` between the capture and `emulateSavedInstanceStateRestore()`. The assignment forces the second `!!` access to read the post-restoration composition.
- **MUST** use `rememberSaveable { … }` (or another `SaveableStateRegistry` integration) for any state expected to survive. Plain `remember` and `mutableStateOf` not behind `rememberSaveable` ARE LOST. Cited at `StateRestorationTester.android.kt:67-68`.
- **MUST** keep the saved Bundle under 1 MB. The check is at `StateRestorationTester.android.kt:163`.
- **MUST NOT** treat `emulateSavedInstanceStateRestore()` as a stand-in for Activity recreation, configuration change, or process death. The KDoc explicitly says "It is not testing the integration with any other life cycles or Activity callbacks." (`StateRestorationTester.android.kt:40-41`).
- **MUST NOT** combine this skill with `createEmptyComposeRule()` — that rule is `ComposeTestRule`, not `ComposeContentTestRule`, so the constructor will not compile.
- **PREFERRED:** drive the state to a non-default value (scroll, type, toggle) before capturing — a test that asserts on the default value cannot prove restoration occurred.
- **PREFERRED:** read the captured values via `rule.runOnIdle { … }` and assert equality, not identity.

## Verification

- [ ] The test constructs `StateRestorationTester(rule)` once and calls only `restorationTester.setContent { … }`.
- [ ] The hoisted state is `var state: T? = null`, NOT `lateinit var` or non-null `var state: T`.
- [ ] The test drives the state to a non-default value before capturing.
- [ ] Capture happens inside `rule.runOnIdle { … }` and the result is stored in a `val`.
- [ ] `state = null` appears between the capture and `restorationTester.emulateSavedInstanceStateRestore()`.
- [ ] All persisted state uses `rememberSaveable { … }`. No plain `remember` is expected to survive.
- [ ] The total saved size is comfortably under 1 MB. Large blobs are excluded; only ids/keys are saved.
- [ ] The test does NOT also assume Activity recreation behaviors (callbacks, ViewModel rebind, etc.).

## References

- `StateRestorationTester` source: `compose/ui/ui-test-junit4/src/androidMain/kotlin/androidx/compose/ui/test/junit4/StateRestorationTester.android.kt:43-77, 128-169`
- `LazyListState` round-trip canonical test: `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/LazyListTest.kt:890-913`
- `Switch` saveable-state-holder regression test: `compose/material3/material3/src/androidDeviceTest/kotlin/androidx/compose/material3/SwitchTest.kt:283-310`
- Save your UI state guide: https://developer.android.com/topic/libraries/architecture/saving-states
- `rememberSaveable` reference: https://developer.android.com/reference/kotlin/androidx/compose/runtime/saveable/package-summary#rememberSaveable
- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
