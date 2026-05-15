---
name: traversing-the-semantics-tree
description: Use this skill to navigate from one Compose semantics node to its relatives via `onParent`, `onChildren`, `onChild`, `onChildAt`, `onSibling`, `onSiblings`, `onAncestors`, plus the collection helpers `onFirst`, `onLast`, `filter`, `filterToOne`, and the `[index]` operator. Covers when to traverse vs when to add a stable `testTag`, the LazyColumn/LazyRow caveat (only currently composed children appear), the absence of a singular `onAncestor`, and the sticky `useUnmergedTree` flag across navigation. Use when the developer mentions `onChildren`, `onChild`, `onParent`, `onSiblings`, `onAncestors`, `filterToOne`, `onFirst`, `onLast`, brittle child-index chains, or asks how to find the second child of a Row, the parent of a Text, or any sibling of a node. If the developer is dot-chaining navigation through a layout, use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - onChildren
  - onParent
  - onAncestors
  - filterToOne
  - onFirst
  - onLast
  - semantics-traversal
  - lazy-column-test
---

# Traversing the Semantics Tree — When a Single Finder Won't Reach the Node

The single-finder shortcuts (`onNodeWithTag`, etc.) cover ~95% of test queries. The remainder need tree navigation: "the parent of this Text", "the third child of this Row", "any sibling that is enabled". This skill maps those navigation operators, calls out the `LazyColumn` snapshot caveat, and shows when traversal is the wrong tool.

## When to use this skill

- The target node has no stable `testTag` and one cannot be added to production (third-party composable, dynamically generated children).
- The test verifies structural relationships ("the second child of the row is the icon").
- The developer chains `.onChildren()[i].onChildAt(j)` and wants to know whether that is the right shape.
- The developer mentions `onChildren`, `onChild`, `onParent`, `onSibling`, `onSiblings`, `onAncestors`, `filterToOne`, `onFirst`, `onLast`, or `[index]`.
- A `LazyColumn` test misses items because they are off-screen.

## When NOT to use this skill

- A `Modifier.testTag(...)` could be added to the target node — adding a tag is almost always cleaner than a traversal chain. See `../finding-nodes-by-tag-text-content/SKILL.md`.
- The relationship is "anywhere above" / "anywhere below" — prefer the matcher-based `hasAnyAncestor` / `hasAnyDescendant` (see `../composing-semantics-matchers/SKILL.md`).
- The query is about a LazyColumn item by key — use `performScrollToKey` instead (see `../../actions/clicking-and-scrolling/SKILL.md`, `../../patterns/testing-lazy-lists/SKILL.md`).

## Prerequisites

- A working `ComposeTestRule` / `ComposeUiTest`. See `../../setup/configuring-test-dependencies/SKILL.md`.
- Familiarity with the merged vs unmerged tree distinction. See `../finding-nodes-by-tag-text-content/SKILL.md`.

## Workflow

- [ ] **1. Pick the navigator by relationship type.** Each operator returns either a `SemanticsNodeInteraction` (single — fails on 0 or >1) or a `SemanticsNodeInteractionCollection` (plural — never fails on 0).

  | From a single node, go to … | API | Returns | File:line |
  |---|---|---|---|
  | parent | `onParent()` | single | `Selectors.kt:36-42` |
  | exactly one child | `onChild()` | single | `Selectors.kt:71-77` |
  | child at an index | `onChildAt(index)` | single | `Selectors.kt:85` |
  | all currently-composed children | `onChildren()` | collection | `Selectors.kt:53-59` |
  | exactly one sibling | `onSibling()` | single | `Selectors.kt:119-125` |
  | all siblings | `onSiblings()` | collection | `Selectors.kt:101-107` |
  | every ancestor up to root | `onAncestors()` | collection | `Selectors.kt:140-146` |

  | From a collection, narrow to … | API | Returns | File:line |
  |---|---|---|---|
  | first | `onFirst()` (= `[0]`) | single | `Selectors.kt:156-158` |
  | last | `onLast()` | single | `Selectors.kt:168-170` |
  | nth | `[index]` | single | `SemanticsNodeInteraction.kt:261-267` |
  | filter to a sub-collection | `filter(matcher)` | collection | `Selectors.kt:178-186` |
  | filter to exactly one | `filterToOne(matcher)` | single | `Selectors.kt:198-206` |

- [ ] **2. Do not look for `onAncestor` (singular) — it does not exist.** The API surface ships only `onAncestors()` plural (`Selectors.kt:140-146`). It returns `[parent, grandparent, …, root]` in that order. To assert "the immediate parent satisfies X", use `onParent().assert(matcherX)` or the `hasParent(matcherX)` predicate.

- [ ] **3. Remember the `useUnmergedTree` flag is sticky.** Every navigator carries the same `useUnmergedTree` value as the source `SemanticsNodeInteraction` (`SemanticsNodeInteraction.kt:42-49`; each `Selectors.kt` constructor passes `useUnmergedTree` through unchanged). A finder created with `onNodeWithTag(tag, useUnmergedTree = true)` keeps the flag on across `onChild`/`onParent`/`filter`. Skydoves hot take #2: default merged, flip to unmerged only when targeting composition detail.

- [ ] **4. `onChildren()` is a snapshot at invocation time.** It returns nodes "currently present in the semantic tree" (`Selectors.kt:46-52`). For a `LazyColumn` or `LazyRow` only the on-screen items appear. To reach an off-screen item, scroll first with `performScrollToIndex` / `performScrollToKey` / `performScrollToNode` (see `../../actions/clicking-and-scrolling/SKILL.md`).

- [ ] **5. Re-finding by tag usually beats navigating.** If the tree shape might change between Compose versions or under translation rotation, a stable tag on the target node is more durable than a `[2].onChildAt(0)` chain. Add the tag in production. Skydoves hot take #1.

- [ ] **6. Use `filterToOne(matcher)` when the count assertion is implicit.** `filterToOne` throws on 0 or >1 matches at fetch time, the same way `onNode(matcher)` does (`Selectors.kt:198-206`). It is the collection-narrowing analogue of `onNode(matcher)`.

## Patterns

### Pattern: prefer a tag over a deep `[index]` chain

```kotlin
// WRONG
@Test
fun second_avatar_isVisible() {
    rule.setContent { ProfileGrid(profiles = profiles) }
    rule.onNodeWithTag("ProfileGrid")
        .onChildren()[2]
        .onChildren()[0]
        .assertIsDisplayed()
}
// WRONG because: layout shuffling (a header inserted, an extra wrapper, a Spacer added)
// silently changes which node is being asserted. The test passes for the wrong reason.
```

```kotlin
// RIGHT
// production:
@Composable
fun ProfileGrid(profiles: List<Profile>) {
    LazyColumn(modifier = Modifier.testTag(ProfileGridTag)) {
        itemsIndexed(profiles) { index, profile ->
            Row(modifier = Modifier.testTag("$AvatarTagPrefix$index")) {
                Avatar(profile, modifier = Modifier.testTag("$AvatarImageTagPrefix$index"))
            }
        }
    }
}

// test:
rule.onNodeWithTag("$AvatarImageTagPrefix${1}").assertIsDisplayed()
```

### Pattern: traversal when no tag is available

```kotlin
@Test
fun row_third_child_isIcon() {
    rule.setContent {
        Row(modifier = Modifier.testTag("toolbar")) {
            Text("Title")
            Spacer(Modifier.weight(1f))
            Icon(Icons.Default.Share, contentDescription = "Share")
        }
    }

    rule.onNodeWithTag("toolbar")
        .onChildren()
        .assertCountEquals(3)

    rule.onNodeWithTag("toolbar")
        .onChildAt(2)
        .assertContentDescriptionEquals("Share")
}
```

`onChildAt(index)` is exactly `onChildren()[index]` (`Selectors.kt:85`). Both fail if the index is out of range or if the resolved node count is not exactly 1 at the leaf.

### Pattern: `filterToOne` instead of `[i]`

```kotlin
// WRONG
rule.onAllNodesWithTag(RowTag).onChildren().filter(hasClickAction())[0]
    .assertHasClickAction()
// WRONG because: indexing into a filtered collection silently passes when the filter returns
// many. The test asserts only "at least one is clickable", not "exactly one".
```

```kotlin
// RIGHT
rule.onAllNodesWithTag(RowTag).onChildren()
    .filterToOne(hasClickAction())
    .assertHasClickAction()
```

### Pattern: assert the parent role from a known child

```kotlin
@Test
fun submitText_isInsideEnabledButton() {
    rule.setContent {
        Button(onClick = {}, modifier = Modifier.testTag("submit"), enabled = true) {
            Text("Submit", modifier = Modifier.testTag("submitLabel"))
        }
    }

    rule.onNodeWithTag("submitLabel", useUnmergedTree = true)
        .onParent()                       // sticky: stays unmerged
        .assertIsEnabled()
        .assertHasClickAction()
}
```

The `useUnmergedTree = true` set on the inner `Text` finder propagates to `onParent()` automatically — no need to repeat it.

### Pattern: `onAncestors()` for the chain to root

```kotlin
@Test
fun confirmButton_isInsideDialog() {
    rule.setContent { ConfirmDialog() }

    rule.onNodeWithTag(ConfirmButtonTag)
        .onAncestors()
        .filterToOne(isDialog())
        .assertExists()
}
```

PREFERRED: `rule.onNode(hasTestTag(ConfirmButtonTag) and hasAnyAncestor(isDialog())).assertExists()` — same intent in one matcher. See `../composing-semantics-matchers/SKILL.md`.

### Pattern: LazyColumn — only on-screen children appear

```kotlin
// WRONG
rule.onNodeWithTag(ListTag).onChildren().assertCountEquals(1000)
// WRONG because: onChildren() returns a snapshot of currently composed children. A LazyColumn
// only composes the visible viewport plus a small prefetch buffer, so the count is window-sized,
// not data-set-sized.
```

```kotlin
// RIGHT — assert the visible count, OR scroll first then assert by tag
rule.onNodeWithTag(ListTag).performScrollToIndex(999)
rule.onNodeWithTag("$ItemTagPrefix${999}").assertIsDisplayed()
```

See `../../patterns/testing-lazy-lists/SKILL.md` for the full LazyColumn workflow.

### Pattern: `onSibling()` to assert "the row's other half"

```kotlin
@Test
fun checkbox_label_isPresent() {
    rule.setContent {
        Row {
            Checkbox(checked = true, onCheckedChange = {},
                modifier = Modifier.testTag("agreeBox"))
            Text("I agree", modifier = Modifier.testTag("agreeLabel"))
        }
    }

    rule.onNodeWithTag("agreeBox")
        .onSibling()
        .assertTextEquals("I agree")
}
```

`onSibling()` requires exactly one sibling (`Selectors.kt:114-125`). For multiple siblings use `onSiblings()` plus `filterToOne(...)`.

## Mandatory rules

- **MUST** prefer adding a `Modifier.testTag(...)` to the target node over a multi-step traversal chain. Skydoves hot take #1.
- **MUST** use `filterToOne(matcher)` over `filter(matcher).onFirst()` when the contract is "exactly one match"; the former throws on >1, the latter silently picks the first.
- **MUST NOT** assume `onChildren()` returns the full data set for a `LazyColumn` / `LazyRow` — it returns the currently-composed snapshot only. Scroll first with `performScrollToIndex` / `performScrollToKey`.
- **MUST NOT** look up a singular `onAncestor` — only `onAncestors()` plural exists. Use `onParent()` for the immediate parent or `hasParent(matcher)` / `hasAnyAncestor(matcher)` as predicates.
- **MUST** remember the `useUnmergedTree` flag is sticky across `onChild`/`onParent`/`filter`. Skydoves hot take #2.
- **PREFERRED:** for "anywhere above/below" relationships, replace traversal chains with `hasAnyAncestor` / `hasAnyDescendant` from `../composing-semantics-matchers/SKILL.md`.

## Verification

- [ ] Every traversal chain has at most one navigation step, OR the deeper chain is justified by a comment naming the missing tag.
- [ ] No `onChildren()[i]` chain spans a `LazyColumn` / `LazyRow` boundary without a preceding `performScrollTo*`.
- [ ] No `onAncestor` (singular) usage — only `onParent`, `onAncestors`, or `hasAnyAncestor`.
- [ ] `filterToOne` is used wherever the contract is "exactly one match"; `filter().onFirst()` is replaced.
- [ ] `useUnmergedTree = true` on a chain root is intentional — there is a comment or matcher reason for it.
- [ ] `./gradlew :app:connectedDebugAndroidTest` or `:app:testDebugUnitTest` passes.

## References

- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- Semantics in Compose: https://developer.android.com/develop/ui/compose/accessibility/semantics
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Selectors.kt` — `onParent`, `onChildren`, `onChild`, `onChildAt`, `onSibling(s)`, `onAncestors`, `onFirst`, `onLast`, `filter`, `filterToOne`.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/SemanticsNodeInteraction.kt` — collection `[index]` operator and the sticky `useUnmergedTree` field.
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Filters.kt` — `hasParent`, `hasAnyChild`, `hasAnySibling`, `hasAnyAncestor`, `hasAnyDescendant` predicate alternatives to traversal.
