---
name: clicking-and-scrolling
description: Use this skill to drive Jetpack Compose UI from tests with the high-level action APIs that do not go through a gesture builder — performClick, performScrollTo, performScrollToIndex, performScrollToKey, performScrollToNode, requestFocus, performSemanticsAction, and performFirstLinkClick. Covers picking the correct receiver node (the scrollable container vs an item), the matchers used to find a scroll parent (hasScrollAction, hasScrollToIndexAction, hasScrollToKeyAction, hasScrollToNodeAction), and how the lazy vs non-lazy scrollable cases differ. Use when the developer asks "how do I tap a Compose node", "scroll to an item in a LazyColumn", "click a link inside Text", "trigger a custom semantics action", or reports "Action performScrollTo failed" / "node has no parent layout with a Scroll SemanticsAction" / "ScrollToIndex not defined".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - performClick
  - performScrollTo
  - performScrollToIndex
  - performScrollToKey
  - performScrollToNode
  - performSemanticsAction
  - lazy-column
  - link-annotation
---

# Clicking and Scrolling — Drive the UI Without a Gesture Builder

Compose's high-level actions resolve to platform-appropriate primitives or to semantics actions that the composable already exposes. Use them whenever the test does not need pixel-level control over the gesture. Reach for `performTouchInput { … }` (see `../injecting-touch-gestures/SKILL.md`) only when the high-level action cannot express the intent.

## When to use this skill

- The developer asks "how do I click a Button / IconButton / clickable Box from a test".
- The test must reveal a node before asserting on it (e.g. an item that is currently below the fold).
- A test must scroll a `LazyColumn` / `LazyRow` to a specific index, key, or matched item.
- The test must trigger a focus change or invoke an `AccessibilityAction<T>` declared via `Modifier.semantics { … }`.
- The test must click a `LinkAnnotation` inside a `Text`.

## When NOT to use this skill

- The test needs partial gestures, multi-touch, velocity-controlled flings, or split touch sequences across recomposition. Use `../injecting-touch-gestures/SKILL.md`.
- The test exercises a hover / scroll wheel / right-click flow. Use `../injecting-mouse-and-keyboard/SKILL.md`.
- The test enters text or fires an IME action. Use `../entering-text/SKILL.md`.
- A node lookup keeps failing. Verify the matcher first via `../../finders/finding-nodes-by-tag-text-content/SKILL.md` and `../../debug/printing-the-semantics-tree/SKILL.md`.

## Prerequisites

- `androidx.compose.ui:ui-test-junit4` (or `androidx.compose.ui:ui-test` for `runComposeUiTest`) configured per `../../setup/configuring-test-dependencies/SKILL.md`.
- The target nodes carry `Modifier.testTag("…")` from production source — skydoves hot take #1.
- For `performScrollToIndex` / `performScrollToKey`: the container must implement the relevant semantics actions. `LazyColumn` / `LazyRow` do; a plain `Modifier.verticalScroll` does not.
- For `performFirstLinkClick`: the `Text` must use an `AnnotatedString` carrying a `LinkAnnotation`.

## Workflow

1. **Pick the right receiver node.** This is the most common failure mode. `performClick` / `performScrollTo` / `performFirstLinkClick` run on a content node; `performScrollToIndex` / `performScrollToKey` / `performScrollToNode` run on the scrollable container. Confirm with `../../debug/printing-the-semantics-tree/SKILL.md` if unsure.

2. **Click a node** — `performClick()`. On Android (both device tests and Robolectric host tests) it delegates to `performTouchInput { click() }` via the Android `actual` (Actions.android.kt:22-24). The mouse-click path lives in JetBrains' Compose Multiplatform desktop fork. Common API at Actions.kt:49-62.

```kotlin
rule.onNodeWithTag(SubmitButtonTag).performClick()
```

3. **Reveal a content node before asserting** — `performScrollTo()`. Walks up the tree to the closest parent carrying `SemanticsActions.ScrollBy` and scrolls by the smallest delta needed to put the node fully in the viewport (Actions.kt:78-87). Throws `AssertionError` if no scroll parent exists.

```kotlin
rule.onNodeWithTag(LastItemTag).performScrollTo().assertIsDisplayed()
```

4. **Scroll a lazy container to an index** — `performScrollToIndex(index)`. Requires `SemanticsActions.ScrollToIndex` on the receiver node (Actions.kt:159-172). The receiver MUST be the container (the `LazyColumn` itself), not an item.

```kotlin
rule.onNode(hasScrollToIndexAction()).performScrollToIndex(42)
```

5. **Scroll to a keyed item** — `performScrollToKey(key)`. Requires both `IndexForKey` and `ScrollToIndex` (Actions.kt:188-203). Matches the `key = { … }` parameter of `LazyColumn` / `LazyRow` items.

```kotlin
rule.onNode(hasScrollToKeyAction()).performScrollToKey("user-7")
```

6. **Scroll until a matcher matches** — `performScrollToNode(matcher)`. Walks the lazy container viewport-by-viewport from start to end. For non-lazy scrollables, falls back to `performScrollTo` once the node materializes (Actions.kt:233-268). Throws when end-of-content is reached without a match.

```kotlin
rule.onNode(hasScrollToNodeAction()).performScrollToNode(hasText("End of feed"))
```

7. **Move focus to a node** — `requestFocus()`. Sugar for `performSemanticsAction(SemanticsActions.RequestFocus)` (Actions.kt:600-601). Required before `performIndirectPointerInput`, and useful for `BasicTextField` setups that bypass `performTextInput`'s auto-focus path.

```kotlin
rule.onNodeWithTag(InputTag).requestFocus()
```

8. **Invoke a custom `AccessibilityAction<T>`** — `performSemanticsAction(key, invocation)` for parameterized actions, `performSemanticsAction(key)` for nullary ones (Actions.kt:631-672). The action MUST be declared on the node via `Modifier.semantics { … }`; otherwise an `AssertionError` is thrown.

```kotlin
val MyAction = SemanticsPropertyKey<AccessibilityAction<(Int) -> Boolean>>("MyAction")
// production:
Modifier.semantics { this[MyAction] = AccessibilityAction("Bump") { delta -> bump(delta); true } }
// test:
rule.onNodeWithTag(Tag).performSemanticsAction(MyAction) { it(3) }
```

9. **Click a `LinkAnnotation` inside `Text`** — `performFirstLinkClick(predicate)` (Actions.kt:777-804). The default predicate `{ true }` clicks the first link. Throws when the receiver has no text or when no link satisfies the predicate.

```kotlin
rule.onNodeWithTag(BodyTag).performFirstLinkClick { it.item is LinkAnnotation.Url }
```

## Patterns

### Pattern: Scrolling a LazyColumn — picking the right receiver

```kotlin
// WRONG
rule.onNodeWithTag("item-7").performScrollToIndex(7)
// WRONG because: performScrollToIndex requires the ScrollToIndex semantics action,
// which lives on the LazyColumn container, not on its items. AssertionError:
// "Failed to scroll to index 7, the node is missing [ScrollToIndex]".
```

```kotlin
// RIGHT
rule.onNode(hasScrollToIndexAction()).performScrollToIndex(7)
rule.onNodeWithTag("item-7").assertIsDisplayed()
```

The matcher `hasScrollToIndexAction()` is defined in `Filters.kt` and identifies any container exposing `SemanticsActions.ScrollToIndex`. Tag the container too if multiple lazy lists exist on screen.

### Pattern: Revealing a non-lazy item before asserting

```kotlin
// WRONG
rule.onNodeWithTag(BottomBannerTag).assertIsDisplayed()
// WRONG because: a Column inside Modifier.verticalScroll renders all children, but
// children outside the viewport are clipped — assertIsDisplayed fails because
// the visible bounds intersect the viewport at zero pixels.
```

```kotlin
// RIGHT
rule.onNodeWithTag(BottomBannerTag).performScrollTo().assertIsDisplayed()
```

`performScrollTo` scans up to the closest `hasScrollAction()` parent and scrolls by the smallest delta needed (Actions.kt:95-141). For lazy lists, prefer `performScrollToNode` since the target item may not yet be composed.

### Pattern: performScrollToNode for arbitrary content

```kotlin
// RIGHT
rule.onNode(hasScrollToNodeAction()).performScrollToNode(
    hasText("Privacy Policy", substring = true)
)
```

The matcher `hasScrollToNodeAction()` accepts both lazy and non-lazy scrollables (`Filters.kt`). For lazy containers, `performScrollToNode` rewinds to index 0 first and walks viewport-sized steps until the matcher hits.

### Pattern: Triggering a custom semantics action

Production:

```kotlin
val Bump = SemanticsPropertyKey<AccessibilityAction<(Int) -> Boolean>>("Bump")

@Composable
fun Counter(value: Int, onBump: (Int) -> Unit) {
    Box(
        Modifier
            .testTag("counter")
            .semantics {
                this[Bump] = AccessibilityAction("Bump") { delta ->
                    onBump(delta); true
                }
            }
    ) { Text(value.toString()) }
}
```

Test:

```kotlin
@Test fun bumpAction_increments() {
    var value by mutableIntStateOf(0)
    rule.setContent { Counter(value) { value += it } }

    rule.onNodeWithTag("counter").performSemanticsAction(Bump) { it(5) }

    rule.runOnIdle { assertEquals(5, value) }
}
```

This is the cleanest way to test logic that does not have a built-in action like `OnClick` — no need to fabricate touch coordinates or to depend on the layout being clickable.

### Pattern: Clicking the first link in a Text

```kotlin
// RIGHT
@Test fun privacyLink_navigates() {
    var clicked = false
    rule.setContent {
        Text(
            buildAnnotatedString {
                append("Read our ")
                withLink(LinkAnnotation.Url("https://example.com/privacy") {
                    clicked = true
                }) { append("Privacy Policy") }
            },
            modifier = Modifier.testTag(BodyTag),
        )
    }

    rule.onNodeWithTag(BodyTag).performFirstLinkClick { it.item is LinkAnnotation.Url }

    rule.runOnIdle { assertTrue(clicked) }
}
```

`performFirstLinkClick` first asserts the node has text, collects every `LinkAnnotation` in the `AnnotatedString`, picks the first that satisfies the predicate, then dispatches `OnClick` on the corresponding link child (Actions.kt:777-804).

### Pattern: requestFocus before indirect pointer input

```kotlin
// RIGHT
rule.onNodeWithTag(SurfaceTag).requestFocus()
rule.performIndirectPointerInput(                  // extension on SemanticsNodeInteractionsProvider, not on a node
    indirectPointerEventPrimaryDirectionalMotionAxis = Vertical,
    inputDeviceSize = IntSize(1000, 1000),
) {
    // events go to the focused tree
}
```

The public `performIndirectPointerInput` extension hangs off `SemanticsNodeInteractionsProvider` (Actions.kt:862) — i.e. the rule (or `ComposeUiTest`) itself. The same-name overload on `SemanticsNodeInteraction` is `internal` (Actions.kt:942), so `rule.onRoot().performIndirectPointerInput(...)` does NOT compile from consumer code.

Indirect pointer input dispatches through the focus path, so an explicit `requestFocus` is mandatory (Actions.kt:807-875). For ordinary touch tests, focus is not required.

## Mandatory rules

- **MUST** call `performScrollToIndex`, `performScrollToKey`, and `performScrollToNode` on the **scrollable container**, not on an item — the semantics actions live on the container. Otherwise `AssertionError: the node is missing [ScrollToIndex]`.
- **MUST** call `performScrollTo` on a content node, not on the container — `performScrollTo` walks **up** to find the scroll parent.
- **MUST** match nodes by `Modifier.testTag("…")` whose value is a constant defined in production source — skydoves hot take #1. Text and content-description finders are i18n-fragile.
- **MUST** route any state mutation that follows an action through `runOnIdle { … }` (skydoves hot take #5). Reading `state` directly from the test thread races with the recomposer.
- **MUST NOT** call `performScrollTo` on a `LazyColumn` item — the item probably is not even composed. Use `performScrollToIndex` / `performScrollToKey` / `performScrollToNode` on the container.
- **MUST NOT** rely on `performClick` for hover, right-click, or wheel scroll. Use the modality-specific scope from `../injecting-mouse-and-keyboard/SKILL.md`.
- **PREFERRED:** select the scroll container via `hasScrollAction()` / `hasScrollToIndexAction()` / `hasScrollToKeyAction()` / `hasScrollToNodeAction()` (Filters.kt) when there is a single such container on screen, instead of adding a redundant test tag to it.

## Verification

- [ ] Every `performScrollToIndex` / `performScrollToKey` / `performScrollToNode` call targets the container — not an item — and a tag or `hasScrollToIndexAction()` matcher selects it.
- [ ] No `performClick` is used to simulate hover, right-click, or scroll wheel.
- [ ] Every `performSemanticsAction` call references a `SemanticsPropertyKey` that the production code installs via `Modifier.semantics { … }`.
- [ ] State assertions after an action read state inside `rule.runOnIdle { … }`.
- [ ] `./gradlew :app:connectedDebugAndroidTest` (or `:app:testDebugUnitTest` for Robolectric) passes for the test under change.

## References

- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- Semantics in Compose: https://developer.android.com/develop/ui/compose/accessibility/semantics
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Actions.kt` — `performClick` (Actions.kt:58), `performScrollTo` (Actions.kt:78), `performScrollToIndex` (Actions.kt:159), `performScrollToKey` (Actions.kt:188), `performScrollToNode` (Actions.kt:233), `requestFocus` (Actions.kt:600), `performSemanticsAction` (Actions.kt:631 / Actions.kt:668), `performFirstLinkClick` (Actions.kt:777).
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Filters.kt` — `hasScrollAction`, `hasScrollToIndexAction`, `hasScrollToKeyAction`, `hasScrollToNodeAction`.
- `compose/foundation/foundation/integration-tests/lazy-tests/.../LazyListTest.kt` — canonical `performScrollToIndex` / `performScrollToKey` patterns.
- `compose/foundation/foundation/src/androidDeviceTest/.../CombinedClickableTest.kt` — `performClick`, `performTouchInput { longClick() }`.
- skydoves — compose-performance-skills: https://github.com/skydoves/compose-performance-skills
