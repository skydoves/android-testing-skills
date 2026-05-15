---
name: composing-semantics-matchers
description: Use this skill to build precise Compose UI test queries by composing `SemanticsMatcher` predicates with `infix and`, `infix or`, and `operator not`, plus the prebuilt filter library (`hasText`, `hasClickAction`, `isEnabled`, `isOn`, `hasTestTag`, `hasContentDescription`, `hasParent`, `hasAnyAncestor`, `hasAnyChild`, `hasAnySibling`, `hasAnyDescendant`, `hasImeAction`, `hasScrollToKeyAction`, `hasScrollToNodeAction`, `isDialog`, `isPopup`, `isRoot`, `isFocused`, `isEditable`, `isHeading`). Covers `SemanticsMatcher.expectValue` / `keyIsDefined` / `keyNotDefined` for custom semantics keys. Use when the developer asks how to find "an enabled button with text Submit", how to combine matchers, how to filter by a custom `SemanticsPropertyKey`, how to write a hierarchical predicate, or mentions `hasParent`, `hasAnyAncestor`, `expectValue`. If the developer wants one matcher instead of three chained assertions, use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - SemanticsMatcher
  - hasText
  - hasClickAction
  - hasParent
  - hasAnyAncestor
  - expectValue
  - matcher-algebra
---

# Composing Semantics Matchers — One Matcher, One Error

A `SemanticsMatcher` is a description string plus a `(SemanticsNode) -> Boolean` predicate (`SemanticsMatcher.kt:26`). Compose ships with a rich filter library and three combinators (`and`, `or`, `not`) that produce a single matcher whose error message lists every clause. This skill picks the right factory, composes precise predicates, and replaces brittle assertion chains.

## When to use this skill

- The developer wants to find "an enabled Button with text Submit" or any node matching multiple constraints.
- The developer asks how to combine `hasText` and `hasClickAction`, or how to negate a matcher with `not`.
- The developer needs to filter by a custom `SemanticsPropertyKey` (e.g. a domain-specific role) using `expectValue`/`keyIsDefined`.
- The developer asks about hierarchical matchers like `hasParent`, `hasAnyAncestor`, `hasAnyChild`, `hasAnySibling`, `hasAnyDescendant`.
- A test chains multiple `assertIs*` calls and the developer wants a single, declarative predicate.

## When NOT to use this skill

- The query is "find by tag/text/content description only" — use `../finding-nodes-by-tag-text-content/SKILL.md`.
- The right node exists but only as a relative — use `../traversing-the-semantics-tree/SKILL.md`.
- The matcher is fine; only the assertions need rework — see `../../assertions/asserting-node-state-and-text/SKILL.md`.

## Prerequisites

- `androidx.compose.ui:ui-test` on the test classpath. See `../../setup/configuring-test-dependencies/SKILL.md` if unresolved.
- Comfort with `Modifier.semantics { … }` in production code. Custom keys must be set via `Modifier.semantics { customKey = value }` to be matchable from tests.

## Workflow

- [ ] **1. Build matchers from the prebuilt filter library.** Every prebuilt matcher lives in `commonMain/.../Filters.kt`. Group by signal:

  | Signal | Matchers | File:line |
  |---|---|---|
  | Enabled / disabled | `isEnabled()`, `isNotEnabled()` | `Filters.kt:38-47` |
  | Toggle (Switch, Checkbox) | `isToggleable()`, `isOn()`, `isOff()` | `Filters.kt:54-70` |
  | Selection (Tab, RadioButton) | `isSelectable()`, `isSelected()`, `isNotSelected()` | `Filters.kt:77-93` |
  | Focus | `isFocusable()`, `isNotFocusable()`, `isFocused()`, `isNotFocused()` | `Filters.kt:100-122` |
  | Click action | `hasClickAction()`, `hasNoClickAction()` | `Filters.kt:129-136` |
  | Scroll action | `hasScrollAction()`, `hasNoScrollAction()`, `hasScrollToIndexAction()`, `hasScrollToKeyAction()`, `hasScrollToNodeAction()` | `Filters.kt:143-425` |
  | Text actions (TextField) | `hasSetTextAction()`, `hasInsertTextAtCursorAction()`, `hasPerformImeAction()`, `hasImeAction(ImeAction)` | `Filters.kt:365-392` |
  | Focus action | `hasRequestFocusAction()` | `Filters.kt:399` |
  | Content description | `hasContentDescription(value, substring, ignoreCase)`, `hasContentDescriptionExactly(vararg)` | `Filters.kt:165-212` |
  | Text | `hasText(text, substring, ignoreCase)`, `hasTextExactly(vararg, includeEditableText=true)` | `Filters.kt:229-293` |
  | A11y / state metadata | `hasStateDescription`, `hasProgressBarRangeInfo`, `isHeading`, `isDialog`, `isPopup`, `isHiddenFromAccessibility`, `isEditable`, `isRoot` | `Filters.kt:301-439` |
  | Test tag | `hasTestTag(testTag)` | `Filters.kt:326-327` |
  | Hierarchical | `hasParent(matcher)`, `hasAnyChild(matcher)`, `hasAnySibling(matcher)`, `hasAnyAncestor(matcher)`, `hasAnyDescendant(matcher)` | `Filters.kt:446-530` |

  `hasText` searches `SemanticsProperties.Text`, `SemanticsProperties.EditableText`, AND `SemanticsProperties.InputText` simultaneously (`Filters.kt:229-258`). `hasContentDescription` matches against the merged list of descriptions (`Filters.kt:165-188`).

- [ ] **2. Compose with `and` / `or` / `not`.** Combinators come from `SemanticsMatcher.kt:60-74`. Each produces a new matcher whose `description` reads `(left) && (right)` / `(left) || (right)` / `NOT (inner)`. The framework prints this description verbatim on assertion failure, so the failure message shows exactly which clause(s) failed.

- [ ] **3. Use `SemanticsMatcher.expectValue` for custom keys.** When the production code declares a custom `SemanticsPropertyKey<T>` and writes it via `Modifier.semantics { customKey = value }`, match it with `SemanticsMatcher.expectValue(customKey, expectedValue)` (`SemanticsMatcher.kt:33-37`). For "this key is set at all", use `SemanticsMatcher.keyIsDefined(customKey)` (`SemanticsMatcher.kt:40-42`); for "this key is NOT set", use `keyNotDefined` (`SemanticsMatcher.kt:45-47`). Built-in matchers like `isOn()` are themselves thin wrappers over `expectValue(SemanticsProperties.ToggleableState, On)` (`Filters.kt:61-62`).

- [ ] **4. Pass the composed matcher to `onNode`/`onAllNodes`.** `onNode(matcher, useUnmergedTree=false)` and `onAllNodes(matcher, useUnmergedTree=false)` are the canonical entry points (`SemanticsNodeInteractionsProvider.kt`). Both `onNodeWithTag` and friends are convenience wrappers — drop them when the predicate gets richer.

- [ ] **5. For hierarchical assertions, prefer `hasAnyAncestor`/`hasAnyDescendant` over multiple traversal steps.** Example: "find the Text inside the dialog" → `onNode(hasText("Discard?") and hasAnyAncestor(isDialog()))`. Cleaner than `onAllNodes(isDialog()).onFirst().onChildren().filter(hasText(...)).onFirst()`.

## Patterns

### Pattern: replace assertion chains with a single matcher

```kotlin
// WRONG
@Test
fun submit_isEnabledClickable() {
    rule.setContent { CheckoutScreen() }
    rule.onNodeWithText("Submit")
        .assertHasClickAction()
        .assertIsEnabled()
}
// WRONG because: each assertion is a separate fetch + error path. Failure prints "no click
// action" without telling the developer the matched node is the *enabled-but-wrong* button on
// a different screen.
```

```kotlin
// RIGHT
@Test
fun submit_isEnabledClickable() {
    rule.setContent { CheckoutScreen() }
    rule.onNode(hasText("Submit") and hasClickAction() and isEnabled())
        .assertExists()
}
```

The composed matcher's description prints `(Text + InputText + EditableText contains 'Submit' (ignoreCase: false)) && (OnClick is defined) && (is enabled)` on failure, naming every unsatisfied clause.

### Pattern: hierarchical match — "the Text inside the dialog"

```kotlin
// WRONG
rule.onAllNodes(isDialog())
    .onFirst()
    .onChildren()
    .filterToOne(hasText("Discard?"))
    .assertIsDisplayed()
// WRONG because: it implicitly assumes the Text is a direct child of the dialog root.
// AlertDialog wraps its body in Column/Surface, so the Text is a descendant, not a child.
```

```kotlin
// RIGHT
rule.onNode(hasText("Discard?") and hasAnyAncestor(isDialog()))
    .assertIsDisplayed()
```

### Pattern: negation and "exactly one"

```kotlin
// "An enabled, non-toggleable Button (so: not a Switch or Checkbox) with text Submit"
rule.onNode(hasText("Submit") and isEnabled() and !isToggleable())
    .assertHasClickAction()
```

`!matcher` invokes `operator fun not()` (`SemanticsMatcher.kt:72-74`) — works on every matcher in the library.

### Pattern: custom semantics key

```kotlin
// production:
val PriorityKey = SemanticsPropertyKey<Int>("Priority")
var SemanticsPropertyReceiver.priority by PriorityKey

@Composable
fun TaskRow(task: Task) {
    Row(modifier = Modifier
        .testTag(TaskRowTag)
        .semantics { priority = task.priority }
    ) { /* … */ }
}

// test:
rule.onNode(hasTestTag(TaskRowTag) and SemanticsMatcher.expectValue(PriorityKey, 1))
    .assertIsDisplayed()

// "this row has a priority assigned at all":
rule.onNode(hasTestTag(TaskRowTag) and SemanticsMatcher.keyIsDefined(PriorityKey))
    .assertIsDisplayed()
```

### Pattern: dropdown / IME-action / scroll-to-key — narrow a TextField query

```kotlin
// "the editable TextField that handles Done as IME action"
rule.onNode(hasSetTextAction() and hasImeAction(ImeAction.Done))
    .performTextInput("hello")

// "any LazyColumn child reachable by key — i.e. has IndexForKey on top of ScrollToIndex"
rule.onNode(hasScrollToKeyAction()).performScrollToKey(itemKey)
```

`hasScrollToKeyAction` is the conjunction `hasKey(ScrollToIndex) and hasKey(IndexForKey)` (`Filters.kt:415-416`). `hasScrollToNodeAction` adds an axis-range check (`Filters.kt:419-425`).

### Pattern: matcher description appears in error messages

```kotlin
val matcher = hasText("Submit") and hasClickAction() and isEnabled()
println(matcher.description)
// (Text + InputText + EditableText contains 'Submit' (ignoreCase: false)) &&
// (OnClick is defined) && (is enabled)
```

The description is human-readable and used by `assert(matcher)` (`Assertions.kt:254-267`) when the matcher does not hold. Clauses MUST be readable in isolation — that is why named matchers are preferred over inline lambdas.

## Mandatory rules

- **MUST** use `infix and` / `infix or` / `operator not` to compose matchers; **MUST NOT** chain `assertIs*` calls when a single composed matcher captures the same intent.
- **MUST** wrap custom semantics keys with `SemanticsMatcher.expectValue(...)` or `SemanticsMatcher.keyIsDefined(...)`; **MUST NOT** access `node.config[CustomKey]` from inside an ad-hoc lambda. The factory methods produce readable error descriptions.
- **MUST** prefer `hasAnyAncestor(isDialog())` / `hasAnyDescendant(...)` over multi-step traversal when the relationship is "somewhere above/below". Cross-reference: `../traversing-the-semantics-tree/SKILL.md`.
- **MUST** keep the `useUnmergedTree` flag at its default `false` unless the matcher targets an inner element collapsed by merge. Skydoves hot take #2.
- **MUST NOT** redefine an existing prebuilt matcher (e.g. writing a custom `SemanticsMatcher("is enabled") { … }`) — use `isEnabled()`. Custom matchers are reserved for keys that don't have a prebuilt filter.
- **PREFERRED:** prefer tag-anchored matchers — `hasTestTag(SubmitTag) and isEnabled()` — over text-anchored. Skydoves hot take #1.

## Verification

- [ ] Every multi-condition query uses one `onNode(matcher)` rather than chained finders + assertions.
- [ ] Custom key checks use `SemanticsMatcher.expectValue` / `keyIsDefined` — `node.config[...]` does not appear in matcher predicates.
- [ ] `./gradlew :app:connectedDebugAndroidTest` or `:app:testDebugUnitTest` passes.
- [ ] Failure messages from intentionally-broken matchers print the composed `description` so the failing clause is identifiable.
- [ ] No matcher is silently shadowed by a re-declared local lambda with the same name.

## References

- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- Semantics in Compose: https://developer.android.com/develop/ui/compose/accessibility/semantics
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/SemanticsMatcher.kt` — `class SemanticsMatcher`, `expectValue` / `keyIsDefined` / `keyNotDefined`, `infix and`, `infix or`, `operator not`.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Filters.kt` — every prebuilt matcher (state, action, text, hierarchical).
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Assertions.kt` — `fun assert(matcher, messagePrefixOnError)` and how the matcher description surfaces in errors.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/SemanticsNodeInteractionsProvider.kt` — `onNode(matcher, useUnmergedTree)` / `onAllNodes(matcher, useUnmergedTree)`.
