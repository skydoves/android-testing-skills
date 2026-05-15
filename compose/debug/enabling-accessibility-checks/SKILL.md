---
name: enabling-accessibility-checks
description: Use this skill to enable Espresso's `AccessibilityValidator` against the Compose semantics tree via `enableAccessibilityChecks(...)` from `androidx.compose.ui:ui-test-accessibility` (for `ComposeUiTest`) or `androidx.compose.ui:ui-test-junit4-accessibility` (for `ComposeTestRule` / `AndroidComposeTestRule`). Covers the API surface, the auto-check after every UI-mutating action (`performMultiModalInput` is the lone exception), the manual `tryPerformAccessibilityChecks()` entry point, sharing the validator with Espresso via `AccessibilityChecks.enable()`, the API 34+ requirement, and the Robolectric inconclusive behavior (logs a `Log.w` warning and still installs the validator, but Robolectric does not faithfully drive accessibility services — b/332778271). Use when the developer asks "how do I enable a11y checks in a Compose test", "AccessibilityChecks.enable", "AccessibilityValidator throws on click", "robolectric a11y not supported", or "ComposeTestRule IllegalStateException".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - accessibility
  - enableAccessibilityChecks
  - AccessibilityValidator
  - tryPerformAccessibilityChecks
  - api-34
  - robolectric
  - espresso-a11y-interop
---

# Enabling Accessibility Checks — Espresso's `AccessibilityValidator` on the Compose Tree

The Compose accessibility test artifacts wire Espresso's `AccessibilityValidator` (from `com.google.android.apps.common.testing.accessibility.framework.integrations.espresso`) into the Compose action pipeline. Once enabled, every action that mutates the UI (`performClick`, `performScrollTo*`, `performTextInput`, …) runs the validator against the underlying View first. Two distinct artifacts cover two test entry points (`ComposeUiTest` and `ComposeTestRule`), with one stable and one experimental. Real device + API 34+ for meaningful results — on Robolectric the validator is still installed but a `Log.w` warning is emitted and any pass is inconclusive (Robolectric does not faithfully drive the accessibility services).

## When to use this skill

- The developer is adding accessibility regression coverage to a Compose-only screen.
- A reviewer asks for an a11y test that fails the build when contrast / touch-target / label rules regress.
- The developer mentions "AccessibilityChecks.enable", "AccessibilityValidator", or "Compose accessibility validator".
- A hybrid Compose + Views screen needs a single shared `AccessibilityValidator` between Espresso and Compose.
- The developer's a11y check appears to do nothing on Robolectric (the `Log.w` warning explains why).

## When NOT to use this skill

- The check fails because of a bad finder, not an a11y violation. Use `../printing-the-semantics-tree/SKILL.md`.
- The test runs on Robolectric (host JVM). Cited limitation at `compose/ui/ui-test-accessibility/src/androidMain/kotlin/androidx/compose/ui/test/accessibility/ComposeUiTestExt.android.kt:50-53`. Move the test to `androidDeviceTest` and a real device API 34+ — see `../../setup/setting-up-host-vs-device-tests/SKILL.md`.
- The test target API is below 34. The annotation `@RequiresApi(34)` makes calls fail to compile on lower minSdk. Use `@SdkSuppress(minSdkVersion = 34)` on the test method and skip on older devices.
- The user wants semantics test coverage (roles, content descriptions, click actions). Use `../../assertions/asserting-node-state-and-text/SKILL.md`.

## Prerequisites

- Real Android device (or emulator) running API **34+** (Android U).
- One of:
  - `androidx.compose.ui:ui-test-accessibility` on `androidTestImplementation` — extends `ComposeUiTest`. `@RequiresApi(34) @ExperimentalTestApi`. Needs `@OptIn(ExperimentalTestApi::class)`.
  - `androidx.compose.ui:ui-test-junit4-accessibility` on `androidTestImplementation` — extends `AndroidComposeTestRule` and `ComposeTestRule`. `@RequiresApi(34)` only — NOT experimental.
- The test class skeleton from `../../patterns/structuring-a-compose-test/SKILL.md`.

## Workflow

- [ ] **1. Enable checks once per test (or in `@Before`).** API for `AndroidComposeTestRule`:

```kotlin
import androidx.compose.ui.test.junit4.accessibility.enableAccessibilityChecks
import com.google.android.apps.common.testing.accessibility.framework.integrations.espresso.AccessibilityValidator

@Before
fun before() {
    rule.enableAccessibilityChecks()      // default: AccessibilityValidator().setRunChecksFromRootView(true)
}
```

Default-validator signature, cited at `compose/ui/ui-test-junit4-accessibility/src/androidMain/kotlin/androidx/compose/ui/test/junit4/accessibility/AndroidComposeTestRuleExt.android.kt:44-49`:

```kotlin
@RequiresApi(34)
public fun <R : TestRule, A : ComponentActivity> AndroidComposeTestRule<R, A>.enableAccessibilityChecks(
    accessibilityValidator: AccessibilityValidator =
        AccessibilityValidator().setRunChecksFromRootView(true),
)
```

For `ComposeUiTest` (e.g. inside `runComposeUiTest { }`), the equivalent lives in `androidx.compose.ui.test.accessibility` and requires `@OptIn(ExperimentalTestApi::class)`:

```kotlin
@OptIn(ExperimentalTestApi::class)
@Test
fun runs() = runAndroidComposeUiTest<ComponentActivity> {
    enableAccessibilityChecks()
    onNodeWithText("Submit").performClick()
}
```

Cited from `compose/ui/ui-test-accessibility/samples/src/main/java/androidx/compose/ui/test/accessibility/samples/AccessibilityChecksSamples.kt:42-61`.

- [ ] **2. Configure stricter modes via the `AccessibilityValidator` itself.** Pass a custom validator to flip the threshold from WARNING-only to ERROR.

```kotlin
val validator = AccessibilityValidator().apply {
    setThrowExceptionFor(AccessibilityCheckResultType.ERROR)
}
rule.enableAccessibilityChecks(validator)
```

Cited from `AccessibilityChecksSamples.kt:68-82`.

- [ ] **3. Auto-checks fire before every UI-mutating action.** Cited at `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Actions.kt:79, 160, 189, 236, 402, 451, 496, 535, 697, 780, 948` — `tryPerformAccessibilityChecks()` is invoked at the top of `performClick`, `performScrollTo`, `performScrollToIndex/Key/Node`, `performTouchInput`, `performMouseInput`, `performKeyInput`, `performTrackpadInput`, `performRotaryScrollInput`, `performIndirectPointerInput`, `performFirstLinkClick`, and the text-action family. **`performMultiModalInput` does NOT auto-check** (`Actions.kt:582`); the developer must call `tryPerformAccessibilityChecks()` explicitly. `requestFocus()` (Actions.kt:600-601) goes through `performSemanticsAction` and likewise does NOT auto-check — call `tryPerformAccessibilityChecks()` after focusing if accessibility coverage matters there.

- [ ] **4. Run a check manually when needed (for `performMultiModalInput`, or to gate a phase).**

```kotlin
rule.onRoot().tryPerformAccessibilityChecks()
```

Cited from `AccessibilityChecksSamples.kt:53`.

- [ ] **5. Share the validator with Espresso for hybrid Compose + View screens.** `AccessibilityChecks.enable()` returns the same validator instance Espresso uses; pass it to Compose so both layers report against one configuration.

```kotlin
import androidx.test.espresso.accessibility.AccessibilityChecks

@OptIn(ExperimentalTestApi::class)
@Test
fun hybrid() = runAndroidComposeUiTest<ComponentActivity> {
    val validator = AccessibilityChecks.enable()
    enableAccessibilityChecks(validator)
}
```

Cited from `AccessibilityChecksSamples.kt:88-98`.

- [ ] **6. Disable checks when leaving an isolated phase.**

```kotlin
rule.disableAccessibilityChecks()
```

Cited at `AndroidComposeTestRuleExt.android.kt:71-75`.

- [ ] **7. Use the `ComposeTestRule` overload only when the rule is concretely an `AndroidComposeTestRule`.** Cited at `compose/ui/ui-test-junit4-accessibility/src/androidMain/kotlin/androidx/compose/ui/test/junit4/accessibility/ComposeTestRuleExt.android.kt:38-49`. Non-Android rules throw `NotImplementedError`:

```text
Enabling accessibility checks is currently only supported for AndroidComposeTestRule.
If you have a custom ComposeTestRule implementation that wraps an AndroidComposeTestRule,
directly call enableAccessibilityChecks on the AndroidComposeTestRule instead
```

## Patterns

### Pattern: enabling on Robolectric — inconclusive results

```kotlin
// WRONG (assertion is meaningless under Robolectric)
@Config(sdk = [34])
@RunWith(RobolectricTestRunner::class)
class MyA11yTest {
    @get:Rule val rule = createComposeRule(StandardTestDispatcher())

    @Test fun checks() {
        rule.enableAccessibilityChecks()                      // logs Log.w; validator IS installed
        rule.setContent { /* contrast violation */ }
        rule.onNodeWithTag("submit").performClick()           // may PASS, may FAIL — inconclusive
    }
}
// WRONG because: HasRobolectricFingerprint = (Build.FINGERPRINT.lowercase() == "robolectric")
// triggers a Log.w but the validator is STILL installed (ComposeUiTestExt.android.kt:50-61
// AND AndroidComposeTestRuleExt.android.kt:50-64). Robolectric does not faithfully drive
// the accessibility services, so any result is inconclusive. Tracking bug: b/332778271.
// Run accessibility checks ONLY on a real device API 34+ for trustworthy results.
```

```kotlin
// RIGHT — move the test to androidDeviceTest and run on a real device API 34+
@MediumTest
@RunWith(AndroidJUnit4::class)
@SdkSuppress(minSdkVersion = 34)
class MyA11yTest {
    @get:Rule val rule = createComposeRule(StandardTestDispatcher())

    @Before fun before() { rule.enableAccessibilityChecks() }

    @Test fun submitPassesA11y() {
        rule.setContent { MyScreen() }
        rule.onNodeWithTag("submit").performClick()
    }
}
```

### Pattern: `performMultiModalInput` skips the auto-check

```kotlin
// WRONG
@Test
fun multiModal() {
    rule.enableAccessibilityChecks()
    rule.setContent { MyScreen() }
    rule.onNodeWithTag("canvas").performMultiModalInput {
        touch { down(center); up() }
        key  { pressKey(Key.Enter) }
    }
    // No a11y check ran for this action — Actions.kt:582 omits tryPerformAccessibilityChecks().
}
```

```kotlin
// RIGHT
@Test
fun multiModal() {
    rule.enableAccessibilityChecks()
    rule.setContent { MyScreen() }
    rule.onNodeWithTag("canvas").performMultiModalInput { … }
    rule.onRoot().tryPerformAccessibilityChecks()             // explicit
}
```

### Pattern: `ComposeTestRule` (non-Android) overload throws

```kotlin
// WRONG
val rule: ComposeTestRule = MyCustomRuleThatWrapsAndroid()
rule.enableAccessibilityChecks()                              // throws NotImplementedError
// WRONG because: only AndroidComposeTestRule is supported. Cited at
// ComposeTestRuleExt.android.kt:38-49.
```

```kotlin
// RIGHT — call on the underlying AndroidComposeTestRule
val rule = AndroidComposeTestRule(/* … */)
rule.enableAccessibilityChecks()
```

### Pattern: shared validator with Espresso

```kotlin
// RIGHT
@OptIn(ExperimentalTestApi::class)
@Test
fun shared() = runAndroidComposeUiTest<ComponentActivity> {
    val validator = AccessibilityChecks.enable().apply {
        setThrowExceptionFor(AccessibilityCheckResultType.ERROR)
    }
    enableAccessibilityChecks(validator)
    onNodeWithText("Submit").performClick()
}
```

## Mandatory rules

- **MUST** target real device API 34+ for any test that calls `enableAccessibilityChecks`. **MUST NOT** rely on the check producing meaningful results under Robolectric — `Build.FINGERPRINT.lowercase() == "robolectric"` triggers `Log.w("...", "Accessibility checks are currently not supported by Robolectric")` (`ComposeUiTestExt.android.kt:50-53`, `AndroidComposeTestRuleExt.android.kt:50-56`), and the validator is still installed but cannot rely on the accessibility services Robolectric does not faithfully emulate. Treat any pass under Robolectric as inconclusive. Tracking bug: b/332778271.
- **MUST** prefer `androidx.compose.ui:ui-test-junit4-accessibility` (stable) when the rule entry point is used; **MUST** apply `@OptIn(ExperimentalTestApi::class)` when using `androidx.compose.ui:ui-test-accessibility` against `ComposeUiTest`.
- **MUST** call `tryPerformAccessibilityChecks()` manually after `performMultiModalInput { … }` and after `requestFocus()` if the test wants validation for those actions — `Actions.kt:582` omits the auto-check for `performMultiModalInput`, and `requestFocus()` (Actions.kt:600-601) routes through `performSemanticsAction` which likewise does not auto-check.
- **MUST NOT** call `enableAccessibilityChecks` on a `ComposeTestRule` that is not also an `AndroidComposeTestRule`. The non-Android overload throws `NotImplementedError` (`ComposeTestRuleExt.android.kt:38-49`).
- **PREFERRED:** raise the threshold to `AccessibilityCheckResultType.ERROR` via `AccessibilityValidator().setThrowExceptionFor(...)` so violations fail the test instead of merely logging.
- **PREFERRED:** in hybrid Compose + Views suites, share one `AccessibilityValidator` via `AccessibilityChecks.enable()` and pass it to both Espresso and Compose. Cited at `AccessibilityChecksSamples.kt:88-98`.

## Verification

- [ ] The test runs on real device API 34+. `@SdkSuppress(minSdkVersion = 34)` is present if the module's minSdk is lower.
- [ ] No Robolectric `Log.w` warning "Accessibility checks are currently not supported by Robolectric" appears in test output.
- [ ] The relevant artifact is on `androidTestImplementation`: `androidx.compose.ui:ui-test-junit4-accessibility` (rule path) or `androidx.compose.ui:ui-test-accessibility` + `@OptIn(ExperimentalTestApi::class)` (`ComposeUiTest` path).
- [ ] `enableAccessibilityChecks(...)` is called once per test (or in `@Before`); `disableAccessibilityChecks()` is called only when intentionally suppressing checks.
- [ ] Any `performMultiModalInput { … }` site is followed by an explicit `rule.onRoot().tryPerformAccessibilityChecks()`.
- [ ] Hybrid Compose+View suites obtain the validator via `AccessibilityChecks.enable()` and pass it to `enableAccessibilityChecks(validator)`.
- [ ] No `enableAccessibilityChecks` call is made on a non-`AndroidComposeTestRule` instance.

## References

- `enableAccessibilityChecks` (rule path): `compose/ui/ui-test-junit4-accessibility/src/androidMain/kotlin/androidx/compose/ui/test/junit4/accessibility/AndroidComposeTestRuleExt.android.kt:44-75`
- `ComposeTestRule` overload — NotImplementedError: `compose/ui/ui-test-junit4-accessibility/src/androidMain/kotlin/androidx/compose/ui/test/junit4/accessibility/ComposeTestRuleExt.android.kt:38-66`
- `enableAccessibilityChecks` (ComposeUiTest path): `compose/ui/ui-test-accessibility/src/androidMain/kotlin/androidx/compose/ui/test/accessibility/ComposeUiTestExt.android.kt:44-77`
- Robolectric inconclusive behavior (warns + still installs validator): `ComposeUiTestExt.android.kt:50-61`, `AndroidComposeTestRuleExt.android.kt:50-64`, b/332778271
- Auto-check call sites in `Actions.kt`: `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Actions.kt:79, 160, 189, 236, 402, 451, 496, 535, 582 (omitted), 697, 780, 948`
- Samples: `compose/ui/ui-test-accessibility/samples/src/main/java/androidx/compose/ui/test/accessibility/samples/AccessibilityChecksSamples.kt:42-98`
- Espresso accessibility checks: https://developer.android.com/training/testing/espresso/accessibility-checking
- Test for accessibility (Compose): https://developer.android.com/develop/ui/compose/accessibility/testing
- Compose UI testing release notes: https://developer.android.com/jetpack/androidx/releases/compose-ui
