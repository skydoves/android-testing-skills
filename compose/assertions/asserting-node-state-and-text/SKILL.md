---
name: asserting-node-state-and-text
description: Use this skill to verify a Compose semantics node's properties from a UI test using `assertExists`, `assertDoesNotExist`, `assertIsDisplayed`, `assertIsNotDisplayed`, `assertIsDeactivated`, `assertIsEnabled`, `assertIsOn`, `assertIsOff`, `assertIsSelected`, `assertIsFocused`, `assertTextEquals`, `assertTextContains`, `assertContentDescriptionEquals`, `assertValueEquals`, `assertRangeInfoEquals`, `assertHasClickAction`, plus the generic `assert(matcher)` escape hatch and the boolean `isDisplayed()` / `isNotDisplayed()` for `waitUntil` predicates. Covers collection variants `assertCountEquals`, `assertAny`, `assertAll`. Use when the developer wants to verify a Switch is on, a Button is enabled, a Text shows the expected string, a node is displayed vs merely composed, or asks about `assertIsDisplayed` vs `assertExists`. If the developer mentions any `assert*` API on `SemanticsNodeInteraction`, use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - assertIsDisplayed
  - assertIsEnabled
  - assertTextEquals
  - assertCountEquals
  - assertHasClickAction
  - assertExists
  - semantics-assertions
  - compose-test-assert
---

# Asserting Node State and Text — Verify Through the Framework, Not Through `fetchSemanticsNode`

Once a finder resolves to a `SemanticsNodeInteraction`, the next step is asserting it. Compose ships a typed assertion for almost every semantic property; the generic `assert(matcher)` covers the rest. This skill picks the right assertion, distinguishes "exists" from "displayed", and shows when boolean predicates belong inside `waitUntil`.

## When to use this skill

- The developer wants to verify a Switch is on, a Checkbox is unchecked, a Button is enabled, a Text equals an expected string.
- The developer asks about `assertIsDisplayed` vs `assertExists` (one verifies on-screen, the other verifies presence in the tree).
- The developer needs a custom assertion via `assert(matcher)` for a property without a typed extension.
- The developer wants to verify a collection: "exactly 3 items", "at least one is selected", "all are enabled".
- A test compares `node.config[...]` directly instead of using a typed assertion.

## When NOT to use this skill

- The right node cannot be located — see `../../finders/finding-nodes-by-tag-text-content/SKILL.md`.
- The assertion is geometric (width, height, position) — see `./asserting-bounds-and-dimensions/SKILL.md`.
- The check needs a custom `SemanticsMatcher` — see `../../finders/composing-semantics-matchers/SKILL.md`.
- The state changes asynchronously and the assertion is timing-sensitive — see `../../synchronization/synchronizing-with-idle/SKILL.md`.

## Prerequisites

- A working `ComposeTestRule` / `ComposeUiTest`. See `../../setup/configuring-test-dependencies/SKILL.md`.
- The node has been located through `onNode*` / `onAllNodes*` / a navigator.
- For `assertIsDisplayed` semantics, the node must be composed AND placed AND at least partially on-screen post-clip (`Assertions.kt:30-39`).

## Workflow

- [ ] **1. Pick by question type.** Map the developer's question to one assertion call. Every API listed is in `commonMain/.../Assertions.kt` unless noted.

  | Question | API | File:line |
  |---|---|---|
  | Is it in the tree? | `assertExists(errorMessageOnFail)` | `SemanticsNodeInteraction.kt:124-127` |
  | Is it gone? | `assertDoesNotExist()` | `SemanticsNodeInteraction.kt:94-110` |
  | Is it a SubcomposeLayout retained child? | `assertIsDeactivated()` | `SemanticsNodeInteraction.kt:137-148` |
  | Is it visible to the user? | `assertIsDisplayed()` / `assertIsNotDisplayed()` | `Assertions.kt:32-53` |
  | Is it interactive? | `assertIsEnabled()` / `assertIsNotEnabled()` | `Assertions.kt:60-67` |
  | Is the toggle on? | `assertIsOn()` / `assertIsOff()` / `assertIsToggleable()` | `Assertions.kt:74-81, 103` |
  | Is the radio/tab selected? | `assertIsSelected()` / `assertIsNotSelected()` / `assertIsSelectable()` | `Assertions.kt:88-110` |
  | Does it have focus? | `assertIsFocused()` / `assertIsNotFocused()` | `Assertions.kt:117-124` |
  | Does it equal these strings? | `assertTextEquals(vararg, includeEditableText=true)` | `Assertions.kt:181-185` |
  | Does it contain this string? | `assertTextContains(value, substring=false, ignoreCase=false)` | `Assertions.kt:204-208` |
  | Content description equals these? | `assertContentDescriptionEquals(vararg)` | `Assertions.kt:139-141` |
  | Content description contains this? | `assertContentDescriptionContains(value, substring, ignoreCase)` | `Assertions.kt:158-163` |
  | StateDescription equals X? | `assertValueEquals(value)` | `Assertions.kt:216-217` |
  | ProgressBar at this position? | `assertRangeInfoEquals(rangeInfo)` | `Assertions.kt:225-227` |
  | Has a click action? | `assertHasClickAction()` / `assertHasNoClickAction()` | `Assertions.kt:234-243` |
  | Anything else | `assert(matcher, messagePrefixOnError = null)` | `Assertions.kt:254-267` |

- [ ] **2. `assertIsDisplayed` vs `assertExists` — pick deliberately.** A node `assertExists()` if it is in the semantics tree (composed). A node `assertIsDisplayed()` if it is composed AND placed AND at least partially visible after clipping (`Assertions.kt:30-39`). For LazyColumn items not yet scrolled into view, `assertExists()` may fail (item not composed) or succeed (item composed but off-screen — depends on prefetch); `assertIsDisplayed()` is the right gate for "the user sees this".

- [ ] **3. Use the typed assertion, not `fetchSemanticsNode().config[...]`.** Typed assertions delegate to the matcher library and produce framework-formatted errors that name the failed property and dump the node (`Assertions.kt:254-267` — the generic `assert` builds `buildGeneralErrorMessage(errorMessageOnFail, selector, node)`). Direct config reads bypass that error reporting and miss the auto-attached node dump.

- [ ] **4. For collections, use the collection-typed assertions.** `assertCountEquals(expectedSize)` for cardinality (`Assertions.kt:276-292`), `assertAny(matcher)` for "at least one matches" (fails on empty — `Assertions.kt:300-312`), `assertAll(matcher)` for "every node matches" (passes on empty — `Assertions.kt:323-339`).

- [ ] **5. For `waitUntil` predicates, use boolean APIs, not throwing assertions.** `isDisplayed()` returns `Boolean` (`Assertions.kt:351`) — perfect for `rule.waitUntil { node.isDisplayed() }`. `isNotDisplayed()` is its inverse (`Assertions.kt:361`). Both throw if multiple nodes match the finder, but never on zero. Cross-reference: `../../synchronization/synchronizing-with-idle/SKILL.md`.

- [ ] **6. For one-off properties without a typed assertion, use `assert(matcher)`.** Combine with prebuilt or composed matchers from `../../finders/composing-semantics-matchers/SKILL.md`. The `messagePrefixOnError` lambda is for adding context when this assert is a precondition for a larger operation.

## Patterns

### Pattern: `assertIsDisplayed` over `assertExists` for user-facing checks

```kotlin
// WRONG
@Test
fun submit_isVisibleAfterError() {
    rule.setContent { CheckoutScreen(state = state) }
    rule.onNodeWithTag(SubmitTag).assertExists()
}
// WRONG because: assertExists passes for nodes that are composed but off-screen, behind a
// dialog, or measured to zero size. The test does not prove the user actually sees the button.
```

```kotlin
// RIGHT
@Test
fun submit_isVisibleAfterError() {
    rule.setContent { CheckoutScreen(state = state) }
    rule.onNodeWithTag(SubmitTag).assertIsDisplayed()
}
```

### Pattern: typed `assertIsOn` over manual config reads

```kotlin
// WRONG
@Test
fun darkMode_switch_isOn() {
    rule.setContent { SettingsScreen() }
    val node = rule.onNodeWithTag(DarkModeSwitchTag).fetchSemanticsNode()
    assertEquals(ToggleableState.On, node.config[SemanticsProperties.ToggleableState])
}
// WRONG because: bypasses the framework's error reporting. Failure prints a JUnit assertEquals
// diff with no node dump, no selector description, no hint about which screen the failure
// happened on.
```

```kotlin
// RIGHT
@Test
fun darkMode_switch_isOn() {
    rule.setContent { SettingsScreen() }
    rule.onNodeWithTag(DarkModeSwitchTag).assertIsOn()
}
```

`assertIsOn` is `assert(isOn())` (`Assertions.kt:74`); `isOn()` is `expectValue(SemanticsProperties.ToggleableState, ToggleableState.On)` (`Filters.kt:61-62`). The error names the property and dumps the node automatically.

### Pattern: `assertTextEquals` (vararg) handles merged Text + EditableText

```kotlin
// production:
TextField(value = "hello", onValueChange = {}, label = { Text("Name") },
    modifier = Modifier.testTag(NameFieldTag))

// test:
rule.onNodeWithTag(NameFieldTag).assertTextEquals("Name", "hello")
```

`assertTextEquals(vararg)` matches the unordered set of `SemanticsProperties.Text` plus, by default, `SemanticsProperties.EditableText` (`Assertions.kt:181-185` → `Filters.kt:274-293`). To exclude editable text from the comparison: `assertTextEquals("Name", includeEditableText = false)`.

### Pattern: `assertCountEquals` instead of `onAllNodes(...).onFirst().assertExists()`

```kotlin
// WRONG
rule.onAllNodesWithTag(ItemTag).onFirst().assertExists()
// WRONG because: only proves there is at least one item. Drift to zero items causes
// "no node matched" instead of a count error; drift to many is silently accepted.
```

```kotlin
// RIGHT
rule.onAllNodesWithTag(ItemTag).assertCountEquals(3)
```

### Pattern: `assert(matcher)` for properties without a typed assertion

```kotlin
@Test
fun row_hasPriorityOne() {
    rule.setContent { TaskList(tasks = tasks) }

    rule.onNodeWithTag(TaskRowTag)
        .assert(SemanticsMatcher.expectValue(PriorityKey, 1))
}
```

`PriorityKey` is a custom `SemanticsPropertyKey<Int>` set via `Modifier.semantics { priority = … }`. See `../../finders/composing-semantics-matchers/SKILL.md` for matcher composition.

### Pattern: `assertAny` (fails on empty) vs `assertAll` (passes on empty)

```kotlin
// "at least one row is selected" — must have at least one row
rule.onAllNodesWithTag(RowTag).assertAny(isSelected())

// "every row is enabled" — accepts zero rows (passes vacuously)
rule.onAllNodesWithTag(RowTag).assertAll(isEnabled())
```

`assertAny` throws `AssertionError("Failed to assertAny … no node matched")` on an empty collection (`Assertions.kt:305-307`). `assertAll` returns successfully on an empty collection (`Assertions.kt:323-339`). Pick deliberately.

### Pattern: `isDisplayed()` inside `waitUntil`

```kotlin
// WRONG
rule.waitUntil { rule.onNodeWithTag(SnackbarTag).assertIsDisplayed(); true }
// WRONG because: assertIsDisplayed throws on the first poll where the snackbar isn't yet
// shown. waitUntil expects a boolean — predicates that throw will fail-fast.
```

```kotlin
// RIGHT
rule.waitUntil(timeoutMillis = 2_000) {
    rule.onAllNodesWithTag(SnackbarTag).fetchSemanticsNodes().isNotEmpty() &&
        rule.onNodeWithTag(SnackbarTag).isDisplayed()
}
rule.onNodeWithTag(SnackbarTag).assertTextContains("Saved")
```

`isDisplayed()` returns `false` when zero nodes match and only throws on multiple matches (`Assertions.kt:343-351`). Cross-reference: `../../synchronization/synchronizing-with-idle/SKILL.md` for `waitUntil` vs `mainClock.advanceTimeUntil`.

### Pattern: `assertIsDeactivated` for SubcomposeLayout retained children

```kotlin
// LookaheadLayout / SubcomposeLayout may retain a previously composed slot for reuse.
// To verify the slot is currently deactivated (kept, not active):
rule.onNodeWithTag(RetainedSlotTag).assertIsDeactivated()
```

`assertIsDeactivated` fetches the node *without* skipping deactivated ones (`SemanticsNodeInteraction.kt:137-148`) and checks `node.layoutInfo.isDeactivated`.

## Mandatory rules

- **MUST** use the typed assertion (`assertIsOn`, `assertIsEnabled`, `assertTextEquals`, …) over `fetchSemanticsNode().config[...]`. The framework's error message includes the selector, the node dump, and the failed clause.
- **MUST** use `assertIsDisplayed()` when the contract is "the user sees this"; **MUST** use `assertExists()` only when the contract is "this is in the semantics tree" (e.g. checking presence in unmerged tree without on-screen requirement).
- **MUST** use the boolean `isDisplayed()` / `isNotDisplayed()` inside `waitUntil { … }` blocks; **MUST NOT** call throwing `assertIs*` from inside `waitUntil`.
- **MUST** use `assertCountEquals` for collection cardinality; **MUST NOT** index `[0]` to imply "the only one".
- **MUST NOT** repeat the same assertion across chained calls when one composed matcher would do — see `../../finders/composing-semantics-matchers/SKILL.md`.
- **PREFERRED:** prefer tag-anchored lookups before assertions. Skydoves hot take #1.

## Verification

- [ ] No assertion reads `node.config[...]` directly. All property checks go through `assertIs*` / `assertTextEquals` / `assert(matcher)`.
- [ ] `assertIsDisplayed` is used for user-visible checks; `assertExists` only when on-screen visibility is irrelevant.
- [ ] `waitUntil { … }` blocks use `isDisplayed()` / `isNotDisplayed()` / `fetchSemanticsNodes().size` — no throwing `assertIs*` calls inside.
- [ ] `assertAny` / `assertAll` choice matches the intended empty-collection semantics.
- [ ] `./gradlew :app:connectedDebugAndroidTest` (device) or `:app:testDebugUnitTest` (host) passes.
- [ ] Failure messages mention the offending semantic property by name (proof the typed assertion is in use).

## References

- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- Semantics in Compose: https://developer.android.com/develop/ui/compose/accessibility/semantics
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Assertions.kt` — every `assertIs*` / `assertTextEquals` / `assertContentDescriptionEquals` / `assertHasClickAction` / `assertCountEquals` / `assertAll` / `assertAny` / `isDisplayed` / `isNotDisplayed`.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/SemanticsNodeInteraction.kt` — `assertExists`, `assertDoesNotExist`, `assertIsDeactivated`, `fetchSemanticsNode`.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Filters.kt` — `isOn`, `isEnabled`, `hasText`, `hasClickAction` underlying matchers used by typed assertions.
