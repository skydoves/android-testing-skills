---
name: testing-lazy-lists
description: Use this skill to test `LazyColumn`, `LazyRow`, and `LazyVerticalGrid` correctly — tag the container with `Modifier.testTag(...)`, tag each item by its key, scroll via the semantic action (`hasScrollAction()`, `performScrollToIndex`, `performScrollToKey`) or via `Modifier.testTag` plus `performTouchInput { swipeUp() }`, and verify visibility with `assertIsDisplayed` (NOT `assertExists`, since off-screen lazy items still exist as semantic nodes in some configurations). Covers `LazyListState.layoutInfo.visibleItemsInfo` as the highest-signal probe, `mainClock.autoAdvance = false` for animated item placement, and the parameterized vertical/horizontal base-class pattern. Use when the developer asks "how do I scroll a LazyColumn in a test", "assertExists vs assertIsDisplayed", "scroll to a key", "test item placement animation", or sees a test pass when an item is off-screen.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - lazycolumn
  - lazyrow
  - lazylayout
  - performScrollToIndex
  - performScrollToKey
  - hasScrollAction
  - layoutInfo
  - visibleItemsInfo
  - assertIsDisplayed
---

# Testing Lazy Lists — Tag the Container, Scroll by Semantics, Probe `layoutInfo`

`LazyColumn` / `LazyRow` / `LazyVerticalGrid` only compose items that are or were near the viewport. That breaks the naive "find by text and assert" flow because off-screen items may not exist, may exist as detached semantic nodes, or may exist with wrong bounds. This skill encodes the patterns androidx itself uses — tag the container, scroll via the semantic action or via `testTag` + a swipe gesture, verify with `assertIsDisplayed`, and read `LazyListState.layoutInfo` for ground truth.

## When to use this skill

- The developer is writing a test for any `Lazy*` composable (`LazyColumn`, `LazyRow`, `LazyVerticalGrid`, `LazyHorizontalGrid`, `LazyVerticalStaggeredGrid`).
- A test asserts an item is "present" but the developer cannot tell whether it is on-screen.
- The developer wants to programmatically scroll to index 7, key `"key_10"`, or by an offset.
- A test asserts on a per-item index/offset and needs the canonical `state.layoutInfo` shape.
- The developer is testing `Modifier.animateItem()` or any item placement animation.
- A reviewer asks why `assertExists` is wrong for a lazy item.

## When NOT to use this skill

- The list under test is a non-lazy `Column`/`Row`/`FlowRow`. Use `../structuring-a-compose-test/SKILL.md` plus `../../assertions/asserting-node-state-and-text/SKILL.md`.
- The class skeleton itself is wrong; fix `../structuring-a-compose-test/SKILL.md` first.
- The scroll action is a custom `Modifier.scrollable` (not `LazyListState`-backed); use `../../actions/injecting-touch-gestures/SKILL.md`.
- The test is for animation timing; layer `../../synchronization/testing-animations-deterministically/SKILL.md`.

## Prerequisites

- Class skeleton from `../structuring-a-compose-test/SKILL.md` (`@MediumTest` + `@RunWith(AndroidJUnit4::class)` + `createComposeRule(StandardTestDispatcher())`).
- A const tag for the container in production source (skydoves directive #1):

```kotlin
const val LazyListTag = "LazyListTag"
```

- Each item carries `Modifier.testTag(it)` where `it` is the stable item key (matches `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/LazyListTest.kt:285-296`).

## Workflow

- [ ] **1. Tag the container, tag each item by key.** Tags belong in production source.

```kotlin
// production
LazyColumn(modifier = Modifier.testTag(LazyListTag), state = state) {
    items(snacks, key = { it.id }) { snack ->
        Row(Modifier.testTag(snack.id.toString())) { /* … */ }
    }
}
```

- [ ] **2. Scroll to an index via the semantic action.** PREFERRED for tests that do not care which `Lazy*` composable is under test.

```kotlin
rule.onNode(hasScrollAction()).performScrollToIndex(7)
```

- [ ] **3. Scroll via the container tag.** Use this when the test must disambiguate among multiple scrollable nodes.

```kotlin
rule.onNodeWithTag(LazyListTag).performScrollToIndex(7)
```

- [ ] **4. Scroll to a key.** This is the most stable scrolling primitive — items can be added or reordered without breaking the test.

```kotlin
rule.onNode(hasScrollToKeyAction()).performScrollToKey("key_10")
```

Cited from `compose/ui/ui-test/src/androidDeviceTest/kotlin/androidx/compose/ui/test/actions/ScrollToKeyTest.kt:70-80`.

- [ ] **5. Scroll by raw gesture when the container has no scroll semantics.** Fall back to a touch swipe.

```kotlin
rule.onRoot().performTouchInput { swipeUp() }
// or, scoped to the list container:
rule.onNodeWithTag(LazyListTag).performTouchInput { swipeUp() }
```

- [ ] **6. Verify visibility with `assertIsDisplayed`, not `assertExists`.** Lazy items can be composed but off-screen, or removed from composition entirely depending on `beyondBoundsItemCount`. `assertExists` only proves the node is in the semantics tree.

- [ ] **7. Count visible children with `.onChildren().assertCountEquals(N)`.** This counts what the merged tree currently surfaces, NOT every backed item.

```kotlin
rule.onNodeWithTag(LazyListTag).onChildren().assertCountEquals(3)
```

- [ ] **8. For ground truth, read `state.layoutInfo` inside `runOnIdle`.** This is the highest-signal probe.

```kotlin
rule.runOnIdle {
    val keys = state.layoutInfo.visibleItemsInfo.map { it.key }
    assertThat(keys).isEqualTo(listOf(0))
}
```

Cited from `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/LazyListRequestScrollTest.kt:155-184` (the `requestScrollToItem` test reads `state.firstVisibleItemIndex`, `state.firstVisibleItemScrollOffset`, and `state.visibleKeys`).

- [ ] **9. For animated item placement, set `mainClock.autoAdvance = false` and step frames manually.** See skydoves directive #3.

```kotlin
@Before
fun before() {
    rule.mainClock.autoAdvance = false
}

@Test
fun reorderTwoItems() {
    var list by mutableStateOf(listOf(0, 1))
    rule.setContent { LazyList { items(list, key = { it }) { Item(it) } } }

    assertPositions(0 to 0f, 1 to itemSize)
    rule.runOnUiThread { list = listOf(1, 0) }

    onAnimationFrame { fraction ->
        assertPositions(
            0 to 0 + itemSize * fraction,
            1 to itemSize - itemSize * fraction,
            fraction = fraction,
        )
    }
}

private fun onAnimationFrame(duration: Long = Duration, onFrame: (fraction: Float) -> Unit) {
    require(duration.mod(FrameDuration) == 0L)
    rule.waitForIdle()
    rule.mainClock.advanceTimeByFrame()
    var expectedTime = rule.mainClock.currentTime
    for (i in 0..duration step FrameDuration) {
        val fraction = i / duration.toFloat()
        onFrame(fraction)
        if (i < duration) {
            rule.mainClock.advanceTimeBy(FrameDuration)
            expectedTime += FrameDuration
            assertThat(expectedTime).isEqualTo(rule.mainClock.currentTime)
        }
    }
}
```

Cited from `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/LazyListItemPlacementAnimationTest.kt:120-148, 1724-1738`. Detail in `../../synchronization/testing-animations-deterministically/SKILL.md`.

- [ ] **10. Parameterize vertical vs horizontal with a base class.** Mirror `BaseLazyListTestWithOrientation` to share scrolling helpers between `LazyColumn` and `LazyRow` tests.

```kotlin
@RunWith(Parameterized::class)
class MyLazyTest(orientation: Orientation) : BaseLazyListTestWithOrientation(orientation) {
    companion object {
        @JvmStatic
        @Parameterized.Parameters(name = "{0}")
        fun params() = arrayOf(Orientation.Vertical, Orientation.Horizontal)
    }
}
```

Cited from `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/BaseLazyListTestWithOrientation.kt:55-114`.

## Patterns

### Pattern: `assertExists` vs `assertIsDisplayed`

```kotlin
// WRONG
@Test
fun item7Visible() {
    rule.setContent { LazyColumn(Modifier.testTag(LazyListTag)) { items(100) { Item("item-$it") } } }
    rule.onNodeWithTag(LazyListTag).performScrollToIndex(7)
    rule.onNodeWithTag("item-7").assertExists()
}
// WRONG because: assertExists only proves the node is in the semantics tree. Lazy lists can
// keep prefetched / detached items in semantics; an item composed out of viewport will pass
// assertExists yet not be on-screen.
```

```kotlin
// RIGHT
@Test
fun item7Visible() {
    rule.setContent { LazyColumn(Modifier.testTag(LazyListTag)) { items(100) { Item("item-$it") } } }
    rule.onNodeWithTag(LazyListTag).performScrollToIndex(7)
    rule.onNodeWithTag("item-7").assertIsDisplayed()
}
```

### Pattern: counting items — `onChildren()` vs `state.layoutInfo`

```kotlin
// WRONG
@Test
fun has100Items() {
    rule.setContent { LazyColumn(Modifier.testTag(LazyListTag)) { items(100) { … } } }
    rule.onNodeWithTag(LazyListTag).onChildren().assertCountEquals(100)
}
// WRONG because: only the items currently composed appear as children. The viewport may
// contain three. The total item count lives on LazyListState, not the semantics tree.
```

```kotlin
// RIGHT
@Test
fun has100Items() {
    lateinit var state: LazyListState
    rule.setContent {
        state = rememberLazyListState()
        LazyColumn(Modifier.testTag(LazyListTag), state = state) { items(100) { Item(it) } }
    }
    rule.runOnIdle {
        assertThat(state.layoutInfo.totalItemsCount).isEqualTo(100)
        assertThat(state.layoutInfo.visibleItemsInfo).isNotEmpty()
    }
}
```

### Pattern: scroll by tag for disambiguation

```kotlin
// WRONG
@Test
fun scrollList() {
    rule.setContent {
        Column {
            LazyColumn { items(20) { … } }                          // list A
            LazyColumn(Modifier.testTag(LazyListTag)) { items(20) { … } } // list B
        }
    }
    rule.onNode(hasScrollAction()).performScrollToIndex(5)
}
// WRONG because: hasScrollAction matches both lists, so onNode throws "found 2 nodes". Use the
// container's tag to disambiguate.
```

```kotlin
// RIGHT
rule.onNodeWithTag(LazyListTag).performScrollToIndex(5)
```

### Pattern: find by tag, never by item text

```kotlin
// WRONG
rule.onNodeWithText("Snack #7").assertIsDisplayed()
// WRONG because: copy edits, plurals, and i18n break every test. Skydoves directive #1.
```

```kotlin
// RIGHT — tag each item by its stable key in production
items(snacks, key = { it.id }) { snack ->
    Row(Modifier.testTag(snack.id.toString())) { Text(snack.name) }
}
// then, in the test:
rule.onNodeWithTag(snack.id.toString()).assertIsDisplayed()
```

## Mandatory rules

- **MUST** tag the container with `Modifier.testTag(...)` declared as a `const val` in production source (skydoves directive #1). **MUST NOT** find lazy items by their visible text.
- **MUST** verify visibility with `assertIsDisplayed` / `assertIsNotDisplayed`. **MUST NOT** use `assertExists` / `assertDoesNotExist` to prove that an item is on-screen.
- **MUST** use the semantic scroll action for index- or key-based scrolls: `performScrollToIndex(...)`, `performScrollToKey(...)`. The semantic action is what `LazyListState.scrollToItem` exposes via `Modifier.semantics`.
- **MUST** read `state.layoutInfo` inside `rule.runOnIdle { … }`, never directly from the test thread.
- **MUST** set `rule.mainClock.autoAdvance = false` before any test that asserts on intermediate frames of an item-placement animation. See skydoves directive #3 and `../../synchronization/testing-animations-deterministically/SKILL.md`.
- **MUST NOT** call `rule.onNodeWithTag(LazyListTag).onChildren().assertCountEquals(totalItems)` to verify the total count — only composed items appear as children.
- **PREFERRED:** scroll by **key** (`performScrollToKey`) when testing reorderable / paginated content; the test survives data churn.
- **PREFERRED:** parameterize `Orientation.Vertical` / `Orientation.Horizontal` with a base class (`BaseLazyListTestWithOrientation`-style) instead of duplicating tests.

## Verification

- [ ] The container has `Modifier.testTag(LazyListTag)`; each item has `Modifier.testTag(<stable key>)`.
- [ ] Every visibility assertion uses `assertIsDisplayed` / `assertIsNotDisplayed`. No `assertExists` is used as a stand-in for "on-screen".
- [ ] Scrolling uses `performScrollToIndex` / `performScrollToKey` (preferred) or `performTouchInput { swipeUp() }` (fallback when no semantics).
- [ ] Per-item index / offset assertions read `state.layoutInfo.visibleItemsInfo` inside `runOnIdle`.
- [ ] Tests for item-placement animations set `rule.mainClock.autoAdvance = false` (typically in `@Before`) and step frames manually.
- [ ] No `onNodeWithText("…")` is used to find a list item.

## References

- Compose testing cheat sheet — finders & lazy lists: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- Lists and grids: https://developer.android.com/develop/ui/compose/lists
- Canonical scroll-to-key test: `compose/ui/ui-test/src/androidDeviceTest/kotlin/androidx/compose/ui/test/actions/ScrollToKeyTest.kt:50-95`
- Canonical layoutInfo probe: `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/LazyListRequestScrollTest.kt:155-184`
- Per-item test tags + scroll: `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/LazyListTest.kt:141, 285-296`
- Item-placement animation harness: `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/LazyListItemPlacementAnimationTest.kt:120-148, 1724-1738`
- Vertical/horizontal base class: `compose/foundation/foundation/integration-tests/lazy-tests/src/androidTest/kotlin/androidx/compose/foundation/lazy/list/BaseLazyListTestWithOrientation.kt:55-114`
- `hasScrollToKeyAction` matcher: `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Filters.kt`
