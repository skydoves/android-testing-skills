---
name: finding-nodes-by-tag-text-content
description: Use this skill to locate Compose semantics nodes from a UI test using `onNodeWithTag`, `onNodeWithText`, `onNodeWithContentDescription`, `onAllNodes*`, and `onRoot`. Covers the count contract (single-node finders fail on 0 or >1 matches; collection finders never throw on 0), the merged-vs-unmerged tree distinction, and why tagging in production beats text-matching from tests. Use when the developer reports "no node matched", "multiple nodes matched", "tag not found", "useUnmergedTree", "Modifier.testTag", or asks how to find a Button/Text/Icon in a Compose test. If the developer mentions onNodeWithText, onNodeWithTag, onNodeWithContentDescription, onAllNodesWithTag, onRoot, or merged tree vs unmerged tree, use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - onNodeWithTag
  - onNodeWithText
  - onNodeWithContentDescription
  - useUnmergedTree
  - testTag
  - semantics-tree
  - compose-finders
---

# Finding Nodes by Tag, Text, or Content Description — Tag From Production, Find by Tag From Tests

Every Compose UI test starts by locating a semantics node. This skill picks the right finder, reasons about the count contract, and decides when to flip `useUnmergedTree`. Use it whenever the developer is staring at a "Reason: Expected exactly '1' node but could not find any node that satisfies …" error.

## When to use this skill

- The developer reports "no node matched" / "multiple nodes matched" / "tag not found" / "ambiguous match" from `SemanticsNodeInteraction`.
- The developer asks how to locate a `Button`, `Text`, `Icon`, `IconButton`, `Switch`, or any other composable inside a test.
- The developer mentions `onNodeWithTag`, `onNodeWithText`, `onNodeWithContentDescription`, `onAllNodesWith*`, or `onRoot`.
- A test passes locally but is brittle to copy edits or i18n rotation of the UI strings.
- The developer asks about `useUnmergedTree`, the merged tree, or "why does my inner Text not exist".

## When NOT to use this skill

- The right node is found but assertions fail — see `../../assertions/asserting-node-state-and-text/SKILL.md` (state) or `../../assertions/asserting-bounds-and-dimensions/SKILL.md` (geometry).
- The finder returns one node but custom predicates are needed — see `../composing-semantics-matchers/SKILL.md`.
- The right node exists but only as a child/sibling/descendant — see `../traversing-the-semantics-tree/SKILL.md`.
- The semantics tree itself needs inspection — see `../../debug/printing-the-semantics-tree/SKILL.md`.

## Prerequisites

- `androidx.compose.ui:ui-test-junit4` (or v2) wired up. See `../../setup/configuring-test-dependencies/SKILL.md` if `createComposeRule` is unresolved.
- A `ComposeTestRule` or `ComposeUiTest` is in scope. See `../../setup/choosing-test-rule-vs-runtest/SKILL.md`.
- The composable under test calls `setContent { … }` before the finder runs. Finders are lazy and re-evaluate on every operation.

## Workflow

- [ ] **1. Default to `onNodeWithTag(SomeTag)`.** Add a `Modifier.testTag(SomeTag)` in the production source under a `const val SomeTag = "Some"`. Tests find by tag, not by text. Skydoves hot take #1: in `androidx/material3` itself, `onNodeWithTag` outnumbers `onNodeWithText` roughly 4:1 (1825 vs 424) and `onNodeWithContentDescription` 40:1 (1825 vs 46). Text finders churn with copy edits and break on locale rotation.

- [ ] **2. Pick the finder by signal type.** Mapping (all from `commonMain/.../Finders.kt`):

  | Need to match by … | Finder | File:line |
  |---|---|---|
  | `Modifier.testTag("…")` | `onNodeWithTag(testTag, useUnmergedTree=false)` | `Finders.kt:31-35` |
  | Visible/editable/input text | `onNodeWithText(text, substring=false, ignoreCase=false, useUnmergedTree=false)` | `Finders.kt:110-116` |
  | Accessibility content description | `onNodeWithContentDescription(label, substring=false, ignoreCase=false, useUnmergedTree=false)` | `Finders.kt:66-73` |
  | Custom predicate | `onNode(matcher, useUnmergedTree=false)` from `SemanticsNodeInteractionsProvider` | `SemanticsNodeInteractionsProvider.kt` |
  | Whole tree | `onRoot(useUnmergedTree=false)` | `Finders.kt:150-153` |

  Each `onNodeWith*` is a thin `@CheckResult` wrapper over `onNode(hasTestTag(...))`/`onNode(hasText(...))`/`onNode(hasContentDescription(...))`. The behavior is identical to the matcher form.

- [ ] **3. Respect the count contract.** `SemanticsNodeInteraction` (single) throws `AssertionError` when fetched against 0 or >1 matching nodes (`SemanticsNodeInteraction.kt:150-191`). `SemanticsNodeInteractionCollection` (plural) never throws on 0 — it returns an empty collection that `assertCountEquals(0)` will accept (`Assertions.kt:276-292`). Pick the variant that matches the expected cardinality.

- [ ] **4. Flip `useUnmergedTree = true` only for inner elements.** The merged tree collapses descendants whose semantics roll up into the parent — `Text` and `Icon` inside a `Button` appear as one node. To target the inner `Text`, pass `useUnmergedTree = true`. The flag is sticky: once a finder is created with `useUnmergedTree = true`, every subsequent `onChild`/`onParent`/`filter` keeps it on (`SemanticsNodeInteraction.kt:42-49`). Skydoves hot take #2: default to merged; flip only when targeting composition detail.

- [ ] **5. Read the auto-suggested unmerged hint on count mismatch.** When a finder targets the merged tree and finds zero, the framework re-runs the same query against the unmerged tree and embeds the matches in the error message (`SemanticsNodeInteraction.kt:184-205`, `getNodesInUnmergedTree`). If the error shows nodes that "would have matched" with `useUnmergedTree = true`, flip the flag.

- [ ] **6. For substring or case-insensitive text matches, pass the boolean flags explicitly.** `onNodeWithText("Sub", substring = true)` matches `"Submit"` and `"Subscribe"`. Combine with `onAllNodesWithText` plus `assertCountEquals` to express intent (see Pattern: ambiguous text below).

## Patterns

### Pattern: tag from production, find by tag from tests

```kotlin
// WRONG
@Test
fun submit_isEnabled_afterFormFilled() {
    rule.setContent { CheckoutScreen() }
    rule.onNodeWithText("Place order").assertIsEnabled()
}
// WRONG because: the button label changes with copy edits, and the test breaks the moment QA
// asks for "Buy now". Also fails under any locale that is not the test default.
```

```kotlin
// RIGHT
// production: ui/checkout/CheckoutScreen.kt
const val PlaceOrderButtonTag = "CheckoutScreen.PlaceOrderButton"

@Composable
fun CheckoutScreen() {
    Button(modifier = Modifier.testTag(PlaceOrderButtonTag), onClick = { /* … */ }) {
        Text(stringResource(R.string.place_order))
    }
}

// test:
@Test
fun submit_isEnabled_afterFormFilled() {
    rule.setContent { CheckoutScreen() }
    rule.onNodeWithTag(PlaceOrderButtonTag).assertIsEnabled()
}
```

### Pattern: collection finder when the count is the assertion

```kotlin
// WRONG
@Test
fun greeting_appearsTwice() {
    rule.setContent { GreetingList(listOf("Hello", "Hello")) }
    // Returns the first match silently; never proves "twice".
    rule.onNodeWithText("Hello").assertIsDisplayed()
}
// WRONG because: onNode* throws on >1 matches; if the duplicate count drifts to 1 the test
// passes for the wrong reason, and if it drifts to 3 the test fails with an opaque count error.
```

```kotlin
// RIGHT
@Test
fun greeting_appearsTwice() {
    rule.setContent { GreetingList(listOf("Hello", "Hello")) }
    rule.onAllNodesWithText("Hello").assertCountEquals(2)
}
```

### Pattern: matching an inner Text inside a Button (merge collapse)

```kotlin
// WRONG
@Test
fun button_text_isShown() {
    rule.setContent {
        Button(onClick = {}, modifier = Modifier.testTag("submit")) {
            Text("Submit", modifier = Modifier.testTag("submitLabel"))
        }
    }
    rule.onNodeWithTag("submitLabel").assertIsDisplayed()
}
// WRONG because: in the merged tree the Text is rolled into the Button. The finder sees zero
// nodes and the auto-attached unmerged hint will surface in the error message — flip the flag.
```

```kotlin
// RIGHT
@Test
fun button_text_isShown() {
    rule.setContent { /* same content */ }
    rule.onNodeWithTag("submitLabel", useUnmergedTree = true).assertIsDisplayed()
}
```

### Pattern: substring/ignoreCase for partial copy

```kotlin
// WRONG
rule.onNodeWithText("hello").assertIsDisplayed()
// WRONG because: matches only the exact string. UI showing "Hello, world" or "HELLO" produces
// a "no node matched" error.
```

```kotlin
// RIGHT — when intent really is "contains 'Hello'":
rule.onNodeWithText("Hello", substring = true, ignoreCase = true).assertIsDisplayed()

// RIGHTER — prefer a tag and assert the text once:
rule.onNodeWithTag(GreetingTag).assertTextContains("Hello", substring = true, ignoreCase = true)
```

### Pattern: content description for icon-only controls

```kotlin
// production:
IconButton(onClick = onShare, modifier = Modifier.testTag(ShareButtonTag)) {
    Icon(Icons.Default.Share, contentDescription = stringResource(R.string.share))
}

// test:
rule.onNodeWithTag(ShareButtonTag).assertIsDisplayed()                  // PREFERRED
rule.onNodeWithContentDescription("Share").assertIsDisplayed()          // acceptable for accessibility-focused tests
```

`onNodeWithContentDescription` defaults to merged-tree exact match, so the merged Icon's
content description rolls up to the parent IconButton. To match a specific Icon's description
in a button that contains many icons, flip `useUnmergedTree = true`.

### Pattern: `onRoot()` as the screenshot/print starting point

```kotlin
rule.onRoot().printToLog("tree")                  // dump merged
rule.onRoot(useUnmergedTree = true).printToLog("unmerged")
```

See `../../debug/printing-the-semantics-tree/SKILL.md` for `printToLog` / `printToString` depth controls.

## Mandatory rules

- **MUST** prefer `onNodeWithTag` over `onNodeWithText` and `onNodeWithContentDescription`. Skydoves hot take #1.
- **MUST** put the `Modifier.testTag(...)` in production code under a constant. **MUST NOT** redefine the same string literal in both the production source and the test.
- **MUST** default to `useUnmergedTree = false`. Flip to `true` only when targeting an inner element collapsed by merge or verifying composition detail. Skydoves hot take #2.
- **MUST** use `onAllNodesWith*` + `assertCountEquals` when the count itself is part of the test contract. **MUST NOT** rely on `onNodeWith*` to imply "exactly one" by side effect.
- **MUST NOT** chain finders inside a single statement past the point of clarity (e.g. `onNode(...).onChild().onChildAt(2).onChild()`). Add a tag instead — see `../traversing-the-semantics-tree/SKILL.md`.
- **PREFERRED:** when a count-mismatch error shows unmerged-tree matches, flip `useUnmergedTree = true` rather than rewriting the matcher.

## Verification

- [ ] Every finder in the test file uses `onNodeWithTag` unless the assertion is explicitly about user-visible text or accessibility content description.
- [ ] Every test tag is defined as a `const val` in the production source, not inlined in the test.
- [ ] `useUnmergedTree = true` appears only at sites that target an inner element of a merged composable (Button, IconButton, ListItem, etc.).
- [ ] No raw string-literal text comparison appears where a tag would do.
- [ ] `./gradlew :app:connectedDebugAndroidTest` (device) or `:app:testDebugUnitTest` (Robolectric host) passes locally.
- [ ] No "Reason: Expected exactly '1' node" errors remain.

## References

- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- Semantics in Compose: https://developer.android.com/develop/ui/compose/accessibility/semantics
- Compose Multiplatform testing: https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-test.html
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Finders.kt` — `onNodeWithTag`/`Text`/`ContentDescription`/`onRoot` and their `onAllNodesWith*` siblings.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/SemanticsNodeInteractionsProvider.kt` — base `onNode(matcher, useUnmergedTree)` / `onAllNodes(matcher, useUnmergedTree)`.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/SemanticsNodeInteraction.kt` — count contract (`fetchOneOrThrow`) and the auto-attached unmerged-tree hint (`getNodesInUnmergedTree`).
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Filters.kt` — `hasTestTag`, `hasText`, `hasContentDescription` underlying matchers.
