---
name: structuring-a-compose-test
description: Use this skill to structure a Jetpack Compose UI test class the way androidx itself writes them — `@MediumTest` + `@RunWith(AndroidJUnit4::class)`, a single `createComposeRule(StandardTestDispatcher())` rule, hoisted `mutableStateOf` declared above `setContent { }`, and the Test → Find → Assert → Act → Re-assert flow. Covers when to use `createAndroidComposeRule<MyActivity>()` for custom Activities, how to drive state from outside the composition via `runOnIdle { }`, why state must NOT be hoisted inside `setContent`, and why `@get:Rule createComposeRule()` and `runComposeUiTest { }` MUST NOT both appear in the same test. Use when the developer asks "how do I write a Compose test", "where do I declare state", "how do I name the test class", "MediumTest vs LargeTest", "rule.setContent inside @Before", or shows a test class that won't compile or doesn't drive state from outside.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - compose-test-rule
  - createComposeRule
  - createAndroidComposeRule
  - test-class-skeleton
  - hoisted-state
  - runOnIdle
  - androidjunit4
  - mediumtest
---

# Structuring a Compose Test — The androidx-Canonical Class Skeleton

Compose tests fail or rot because their structure is wrong, not because the assertions are wrong. State that lives inside `setContent { }` cannot be driven from the test thread; `setContent` called inside `@Before` becomes a hidden race; combining `@get:Rule` with `runComposeUiTest { }` creates two competing test environments. This skill encodes the exact class skeleton used by `androidx.compose.material3` and `androidx.compose.foundation` so the developer's tests behave the same way.

## When to use this skill

- The developer is creating a new `*Test.kt` and asks where to put state, the rule, and `setContent`.
- A test compiles but cannot mutate state mid-test ("how do I flip the checkbox after the first assertion?").
- The developer pasted `setContent` into `@Before` and the test is now flaky or will not start.
- A reviewer flags a test for declaring state inside `setContent { }`.
- The developer mixed `@get:Rule createComposeRule()` and `runComposeUiTest { }` and gets confusing failures.

## When NOT to use this skill

- The developer is choosing **which** entry point to use (rule vs. `runComposeUiTest`); use `../../setup/choosing-test-rule-vs-runtest/SKILL.md`.
- The build cannot resolve `createComposeRule`; use `../../setup/configuring-test-dependencies/SKILL.md`.
- The decision is host (Robolectric) vs. device (instrumentation); use `../../setup/setting-up-host-vs-device-tests/SKILL.md`.
- The test is specifically for a `LazyColumn`/`LazyRow`; layer `../testing-lazy-lists/SKILL.md` on top of this skeleton.

## Prerequisites

- `androidx.compose.ui:ui-test-junit4` on `androidTestImplementation`, `androidx.compose.ui:ui-test-manifest` on `debugImplementation`. See `../../setup/configuring-test-dependencies/SKILL.md`.
- Default activity is `androidx.activity.ComponentActivity` (provided by `ui-test-manifest`). For a custom Activity the `androidTest` source set MUST declare it in `AndroidManifest.xml`.
- Kotlin source set: `src/androidTest/kotlin/...` (instrumentation) or `src/test/kotlin/...` (Robolectric host).

## Workflow

- [ ] **1. Mirror the production package.** Place `MyButtonTest.kt` in the same package as `MyButton.kt`. The class name is `<ProductionFile>Test` (`SwitchTest.kt`, `LazyListTest.kt`).

- [ ] **2. Annotate the class.** `@MediumTest` is the default budget. Use `@LargeTest` only for tests that intentionally take seconds (long-clicks via `performTouchInput { longClick() }`, multi-second animations). `@RunWith(AndroidJUnit4::class)` is non-negotiable for instrumentation.

- [ ] **3. Declare exactly one rule.** Prefer the v2 entry point — it defaults the composition dispatcher to `StandardTestDispatcher`, matching `kotlinx.coroutines.test.runTest`.

```kotlin
import androidx.compose.ui.test.junit4.v2.createComposeRule
import kotlinx.coroutines.test.StandardTestDispatcher

@MediumTest
@RunWith(AndroidJUnit4::class)
class MyButtonTest {
    @get:Rule val rule = createComposeRule(StandardTestDispatcher())
}
```

For a custom Activity (e.g. `FragmentActivity`, a screen-under-test Activity), use the reified v2 helper:

```kotlin
import androidx.compose.ui.test.junit4.v2.createAndroidComposeRule
@get:Rule val rule = createAndroidComposeRule<FragmentActivity>(StandardTestDispatcher())
```

Cited from `compose/foundation/foundation/src/androidDeviceTest/kotlin/androidx/compose/foundation/textfield/TextFieldFocusCustomDialogTest.kt:60`.

- [ ] **4. Hoist state ABOVE `setContent`.** Anything the test wants to mutate or read must be a property declared in the test method, then captured by reference inside the composable.

- [ ] **5. Call `setContent { }` INSIDE the `@Test` method, not in `@Before`.** Each test owns its own composition. Tests with shared `setContent` in `@Before` cannot vary content between cases and lose the per-test state setup the framework expects.

- [ ] **6. Follow Test → Find → Assert → Act → Re-assert.** The canonical shape from `compose/material3/material3/src/androidDeviceTest/kotlin/androidx/compose/material3/SwitchTest.kt:240-262` (the `switch_stateChange_movesThumb` test):

```kotlin
@Test
fun switch_stateChange_movesThumb() {
    var checked by mutableStateOf(false)                          // STATE (hoisted)
    rule.setMaterialContent(lightColorScheme()) {                  // SET CONTENT (in test body)
        val spacer = @Composable { Spacer(Modifier.size(16.dp).testTag("spacer")) }
        Switch(
            modifier = Modifier.testTag(defaultSwitchTag),
            checked = checked,
            thumbContent = spacer,
            onCheckedChange = { checked = it },
        )
    }

    rule.onNodeWithTag("spacer", useUnmergedTree = true)           // FIND + ASSERT
        .assertLeftPositionInRootIsEqualTo(8.dp)

    rule.runOnIdle { checked = true }                              // ACT (mutate from outside)

    rule.onNodeWithTag("spacer", useUnmergedTree = true)           // RE-ASSERT
        .assertLeftPositionInRootIsEqualTo(28.dp)

    rule.runOnIdle { checked = false }
    rule.onNodeWithTag("spacer", useUnmergedTree = true).assertLeftPositionInRootIsEqualTo(8.dp)
}
```

- [ ] **7. Read state from outside via `rule.runOnIdle { … }`.** The block runs after `waitForIdle()`, so observed values reflect the latest composition.

```kotlin
val (index, offset) = rule.runOnIdle {
    state.firstVisibleItemIndex to state.firstVisibleItemScrollOffset
}
```

Cited from `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/LazyListTest.kt:903-904`.

- [ ] **8. Write state from outside via `rule.runOnIdle { … }`.** Direct assignment from the test thread races the recomposer.

```kotlin
rule.runOnIdle { count = 5 }
```

## Patterns

### Pattern: state hoisted inside vs. above `setContent`

```kotlin
// WRONG
@Test
fun toggle() {
    rule.setContent {
        var checked by remember { mutableStateOf(false) }   // <-- trapped inside composition
        Switch(checked = checked, onCheckedChange = { checked = it })
    }
    rule.runOnIdle { checked = true }                       // does not compile / has no reference
}
// WRONG because: the test thread has no reference to `checked`. The state cannot be driven
// from outside, and the test can only verify the very first frame.
```

```kotlin
// RIGHT
@Test
fun toggle() {
    var checked by mutableStateOf(false)                    // <-- hoisted to test scope
    rule.setContent {
        Switch(checked = checked, onCheckedChange = { checked = it })
    }
    rule.onNodeWithTag("switch").assertIsOff()
    rule.runOnIdle { checked = true }                       // mutate via runOnIdle
    rule.onNodeWithTag("switch").assertIsOn()
}
```

### Pattern: `setContent` in `@Before`

```kotlin
// WRONG
@Before
fun setUp() {
    rule.setContent { MyScreen(state) }                     // <-- shared across all tests
}

@Test fun stateA() { /* mutate `state` here, but it was already composed without this case */ }
```

```kotlin
// RIGHT
@Test
fun stateA() {
    val state = ScreenState(initial = "A")
    rule.setContent { MyScreen(state) }                     // <-- per-test composition
    rule.onNodeWithTag("title").assertTextEquals("A")
}
```

### Pattern: one rule per class — never mix with `runComposeUiTest`

```kotlin
// WRONG
class MyTest {
    @get:Rule val rule = createComposeRule(StandardTestDispatcher())

    @Test fun a() = runComposeUiTest {                      // <-- second test environment
        setContent { /* … */ }
    }
}
// WRONG because: createComposeRule and runComposeUiTest each manage an independent test
// environment (composition + MainTestClock + IdlingResourceRegistry). Mixing them produces
// undefined behavior. KDoc warns explicitly: see the ComposeUiTest.android.kt KDoc on
// runComposeUiTest, runAndroidComposeUiTest, runEmptyComposeUiTest, and the v2 variants.
```

```kotlin
// RIGHT — pick exactly one entry point per class
class MyRuleTest {
    @get:Rule val rule = createComposeRule(StandardTestDispatcher())
    @Test fun a() { rule.setContent { /* … */ } }
}

// or
class MyFunctionTest {
    @Test fun a() = runComposeUiTest { setContent { /* … */ } }
}
```

### Pattern: custom Activity needs its own manifest entry

```kotlin
// RIGHT — ComponentActivity comes free from ui-test-manifest
@get:Rule val rule = createComposeRule(StandardTestDispatcher())
```

```kotlin
// RIGHT — a custom Activity must be declared in src/androidTest/AndroidManifest.xml
@get:Rule val rule = createAndroidComposeRule<MyActivity>(StandardTestDispatcher())
```

```xml
<!-- src/androidTest/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application>
        <activity android:name=".MyActivity" />
    </application>
</manifest>
```

### Pattern: `MediumTest` vs `LargeTest`

```kotlin
@MediumTest                     // default — sub-second tests
class SwitchTest { /* … */ }

@LargeTest                      // long-clicks, multi-second animations
class CombinedClickableTest { /* … */ }
```

`@MediumTest` is used by `SwitchTest.kt:76` and most material3/foundation suites. `@LargeTest` is reserved for tests whose total wall time intentionally exceeds the medium budget.

## Mandatory rules

- **MUST** annotate the class with `@RunWith(AndroidJUnit4::class)` and a size annotation (`@MediumTest` by default).
- **MUST** declare exactly one rule per class — either `createComposeRule(...)`, `createAndroidComposeRule<A>(...)`, or `createEmptyComposeRule()`. **MUST NOT** combine a `@get:Rule` rule with `runComposeUiTest { }` in the same class. KDoc cited at `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeUiTest.android.kt:158, 204, 247, 338`.
- **MUST** prefer the v2 entry points `androidx.compose.ui.test.junit4.v2.createComposeRule` / `createAndroidComposeRule`. The v1 forms are `@Deprecated(level = WARNING)` because they use `UnconfinedTestDispatcher` instead of `StandardTestDispatcher` (skydoves directive #6).
- **MUST** hoist mutable state as `var x by mutableStateOf(...)` **above** `setContent { }` whenever the test must drive or observe it.
- **MUST** call `rule.setContent { }` **inside** the `@Test` method. **MUST NOT** call it in `@Before`.
- **MUST** mutate state from outside via `rule.runOnIdle { state = … }` (or `rule.runOnUiThread { … }` for cases where idling first is undesirable, e.g. `mainClock.autoAdvance = false`). See skydoves directive #5. Detail in `../../synchronization/synchronizing-with-idle/SKILL.md`.
- **MUST** read state from outside via `rule.runOnIdle { state.value }`. Direct reads from the test thread snapshot the wrong frame.
- **PREFERRED:** find by `Modifier.testTag(...)` declared as a `const` in production (skydoves directive #1). Text finders are i18n-fragile. See `../../finders/finding-nodes-by-tag-text-content/SKILL.md`.
- **PREFERRED:** Test → Find → Assert → Act → Re-assert per logical state change. Multi-step tests stack additional Act → Re-assert pairs (`SwitchTest.kt:140-147`).

## Verification

- [ ] Class has `@RunWith(AndroidJUnit4::class)` + `@MediumTest` (or `@LargeTest`).
- [ ] Exactly one `@get:Rule` declared and it is a `createComposeRule(...)` / `createAndroidComposeRule(...)` / `createEmptyComposeRule()` instance.
- [ ] No `runComposeUiTest { }` invocation appears in the same class.
- [ ] `setContent { }` appears inside `@Test` methods only — not in `@Before` / `@BeforeEach`.
- [ ] Every state the test mutates is declared with `var x by mutableStateOf(...)` above `setContent`, captured by reference inside the composable.
- [ ] All state mutations after `setContent` go through `rule.runOnIdle { }` or `rule.runOnUiThread { }`.
- [ ] All cross-frame state reads happen inside `rule.runOnIdle { }`.
- [ ] No `Thread.sleep` appears in the test method (skydoves directive #7). See `../../synchronization/synchronizing-with-idle/SKILL.md`.
- [ ] Custom Activities are declared in `src/androidTest/AndroidManifest.xml` (not just the production `AndroidManifest.xml`).

## References

- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
- Compose testing setup: https://developer.android.com/develop/ui/compose/testing#setup
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- Canonical class skeleton: `compose/material3/material3/src/androidDeviceTest/kotlin/androidx/compose/material3/SwitchTest.kt:76-263`
- Custom Activity rule: `compose/foundation/foundation/src/androidDeviceTest/kotlin/androidx/compose/foundation/textfield/TextFieldFocusCustomDialogTest.kt:57-60`
- v2 rule factory: `compose/ui/ui-test-junit4/src/androidMain/kotlin/androidx/compose/ui/test/junit4/v2/AndroidComposeTestRule.android.kt`
- "Do not mix rule and runComposeUiTest" KDoc: `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeUiTest.android.kt:158, 204, 247, 338`
- `runOnIdle` semantics: `compose/ui/ui-test-junit4/src/jvmAndAndroidMain/kotlin/androidx/compose/ui/test/junit4/ComposeTestRule.jvmAndAndroid.kt`
