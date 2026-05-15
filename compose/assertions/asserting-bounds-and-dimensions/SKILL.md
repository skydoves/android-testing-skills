---
name: asserting-bounds-and-dimensions
description: Use this skill to verify Compose layout measurements from a UI test using `assertWidthIsEqualTo`, `assertHeightIsEqualTo`, `assertWidthIsAtLeast`, `assertHeightIsAtLeast`, `assertTouchWidthIsEqualTo`, `assertTouchHeightIsEqualTo`, `assertPositionInRootIsEqualTo`, `assertTopPositionInRootIsEqualTo`, `assertLeftPositionInRootIsEqualTo`, plus read helpers `getUnclippedBoundsInRoot`, `getBoundsInRoot`, `getAlignmentLinePosition`, `getFirstLinkBounds`, and the underlying `Dp.assertIsEqualTo(expected, subject, tolerance = Dp(.5f))`. Covers the half-dp default tolerance, the unclipped vs clipped distinction, the canonical "compute padding from two unclipped rects" pattern, and minimum-touch-target assertions like `assertHeightIsAtLeast(MinHeight + 1.dp)`. Use when the developer wants to assert sizes, padding, alignment, position in dp, or asks about `getUnclippedBoundsInRoot`, `DpRect`, touch-target size, or compares widths in pixels. If the developer is comparing layout dimensions from a test, use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - assertWidthIsEqualTo
  - assertHeightIsAtLeast
  - assertPositionInRootIsEqualTo
  - getUnclippedBoundsInRoot
  - DpRect
  - touch-target
  - dp-tolerance
  - layout-assertions
---

# Asserting Bounds and Dimensions — Layout Math in Dp, Not Pixels

Layout assertions belong in dp, run with a half-dp tolerance, and most of the interesting checks (padding, gap, alignment) are subtractions between two `getUnclippedBoundsInRoot()` rectangles. This skill picks the right size/position assertion, explains clipped vs unclipped, and shows the canonical "compute padding from two rects" pattern lifted directly from `material3/ButtonTest.kt`.

## When to use this skill

- The developer wants to verify a Button is 48 dp tall, a Spacer is 16 dp wide, an Icon is at position `(24.dp, 12.dp)`.
- The developer asks how to assert the padding between two composables.
- The developer asks about minimum touch target sizes (`ChipDefaults.MinHeight + 1.dp`).
- The developer is comparing layout values in pixels and wants the dp-typed equivalent.
- The developer mentions `assertWidthIsEqualTo`, `getUnclippedBoundsInRoot`, `DpRect`, `getAlignmentLinePosition`, `getFirstLinkBounds`.

## When NOT to use this skill

- The check is about state (enabled, on, selected) — see `./asserting-node-state-and-text/SKILL.md`.
- The check is "is the node on screen at all" — `assertIsDisplayed()` is enough; bounds math adds friction without value.
- The composable's bounds depend on an animation in flight — pause the clock first; see `../../synchronization/testing-animations-deterministically/SKILL.md`.
- The bounds are relative to a screenshot — use a screenshot test instead.

## Prerequisites

- A working `ComposeTestRule` / `ComposeUiTest`. See `../../setup/configuring-test-dependencies/SKILL.md`.
- The target composable has finished measuring and placing. If it animates in, advance the test clock first — see `../../synchronization/controlling-the-test-clock/SKILL.md`.
- For touch-target assertions, the target node has a click action so `touchBoundsInRoot` is meaningful.

## Workflow

- [ ] **1. Pick the assertion by question type.** All APIs live in `commonMain/.../BoundsAssertions.kt`.

  | Question | API | File:line |
  |---|---|---|
  | Is the layout exactly W dp wide? | `assertWidthIsEqualTo(expectedWidth: Dp)` | `BoundsAssertions.kt:44-46` |
  | Is the layout exactly H dp tall? | `assertHeightIsEqualTo(expectedHeight: Dp)` | `BoundsAssertions.kt:53-55` |
  | At least W wide? | `assertWidthIsAtLeast(expectedMinWidth: Dp)` | `BoundsAssertions.kt:85-87` |
  | At least H tall? | `assertHeightIsAtLeast(expectedMinHeight: Dp)` | `BoundsAssertions.kt:95-99` |
  | Touch-target width? | `assertTouchWidthIsEqualTo(expectedWidth: Dp)` | `BoundsAssertions.kt:62-66` |
  | Touch-target height? | `assertTouchHeightIsEqualTo(expectedHeight: Dp)` | `BoundsAssertions.kt:73-77` |
  | Exact position in root? | `assertPositionInRootIsEqualTo(left: Dp, top: Dp)` | `BoundsAssertions.kt:109-117` |
  | Top position only? | `assertTopPositionInRootIsEqualTo(top: Dp)` | `BoundsAssertions.kt:126-130` |
  | Left position only? | `assertLeftPositionInRootIsEqualTo(left: Dp)` | `BoundsAssertions.kt:139-143` |
  | Read full unclipped bounds | `getUnclippedBoundsInRoot(): DpRect` | `BoundsAssertions.kt:148-152` |
  | Read clipped bounds | `getBoundsInRoot(): DpRect` | `BoundsAssertions.kt:158-165` |
  | Alignment line in dp | `getAlignmentLinePosition(line: AlignmentLine): Dp` | `BoundsAssertions.kt:171-180` |
  | Bounds of a `LinkAnnotation` in a Text | `getFirstLinkBounds(predicate)` | `BoundsAssertions.kt:196-248` |
  | Compare any two `Dp` values | `Dp.assertIsEqualTo(expected, subject, tolerance = Dp(.5f))` | `BoundsAssertions.kt:319-324` |

- [ ] **2. Use unclipped bounds for layout math; clipped bounds for "what the user sees".** `getUnclippedBoundsInRoot()` returns the laid-out rectangle ignoring viewport clipping (`BoundsAssertions.kt:148-152` → `unclippedBoundsInRoot` private, `BoundsAssertions.kt:284-291`). `getBoundsInRoot()` clips to the viewport (`BoundsAssertions.kt:158-165`). Padding/spacing math uses unclipped; partial-visibility checks use clipped.

- [ ] **3. Tolerance is half a dp by default.** `Dp.assertIsEqualTo(expected, subject, tolerance = Dp(.5f))` (`BoundsAssertions.kt:319`) accepts deviations up to 0.5 dp because layout rounding introduces sub-dp drift. Override only when stricter precision is justified by an explicit measurement contract.

- [ ] **4. Compute padding by subtracting two unclipped rects, then call `Dp.assertIsEqualTo`.** This is the canonical material3 pattern (`material3/.../ButtonTest.kt:213-225`):

  ```kotlin
  val buttonBounds = rule.onNodeWithTag(ButtonTestTag).getUnclippedBoundsInRoot()
  val textBounds   = rule.onNodeWithTag(TextTestTag).getUnclippedBoundsInRoot()

  (textBounds.left - buttonBounds.left).assertIsEqualTo(
      24.dp,
      "padding between the start of the button and the start of the text.",
  )

  (buttonBounds.right - textBounds.right).assertIsEqualTo(
      24.dp,
      "padding between the end of the text and the end of the button.",
  )
  buttonBounds.height.assertIsEqualTo(ButtonDefaults.MinHeight, "height of button.")
  ```

  The `subject` string lands in the failure message: `"Actual padding between the start of the button and the start of the text. is 22.dp, expected 24.dp (tolerance: .5.dp)"`.

- [ ] **5. For minimum-size contracts, use `assertHeightIsAtLeast` / `assertWidthIsAtLeast`.** Useful when a composable should never be smaller than a constant, even at large font scale. Example from `material/ChipTest.kt:226-233`:

  ```kotlin
  rule.setMaterialContent { Chip(onClick = {}) { Text(text = "Test chip", fontSize = 50.sp) } }
  rule.onNode(hasClickAction()).assertHeightIsAtLeast(ChipDefaults.MinHeight + 1.dp)
  ```

- [ ] **6. Use touch bounds when verifying tap-target accessibility.** `assertTouchWidthIsEqualTo` / `assertTouchHeightIsEqualTo` reads `node.touchBoundsInRoot` (`BoundsAssertions.kt:270-282`), which can extend past the visual bounds when the composable applies `Modifier.minimumInteractiveComponentSize()` or similar. Visual bounds use `getUnclippedBoundsInRoot`; touch bounds answer "where will a click land".

## Patterns

### Pattern: dp typed assertions over pixel reads

```kotlin
// WRONG
@Test
fun submit_isMin48dpTall() {
    rule.setContent { CheckoutScreen() }
    val node = rule.onNodeWithTag(SubmitTag).fetchSemanticsNode()
    val heightPx = node.size.height
    assert(heightPx >= 48 * Resources.getSystem().displayMetrics.density)
}
// WRONG because: pixel-typed and density-dependent. Reads outside the framework's tolerance
// model. Failure prints raw integers, no node dump, no subject label.
```

```kotlin
// RIGHT
@Test
fun submit_isMin48dpTall() {
    rule.setContent { CheckoutScreen() }
    rule.onNodeWithTag(SubmitTag).assertHeightIsAtLeast(48.dp)
}
```

### Pattern: padding by subtracting two unclipped rects

```kotlin
@Test
fun button_text_has24dpPadding() {
    rule.setContent {
        Button(onClick = {}, modifier = Modifier.testTag(ButtonTestTag)) {
            Text("Submit", modifier = Modifier.testTag(TextTestTag).semantics(mergeDescendants = true) {})
        }
    }

    val buttonBounds = rule.onNodeWithTag(ButtonTestTag).getUnclippedBoundsInRoot()
    val textBounds   = rule.onNodeWithTag(TextTestTag).getUnclippedBoundsInRoot()

    (textBounds.left - buttonBounds.left).assertIsEqualTo(24.dp, "start padding")
    (buttonBounds.right - textBounds.right).assertIsEqualTo(24.dp, "end padding")
}
```

The merge bypass on the inner `Text` is the same trick used by `material3/ButtonTest.kt:202-226` to keep the inner Text addressable from the merged tree.

### Pattern: `assertPositionInRootIsEqualTo` for absolute placement

```kotlin
// "the close button sits at (320.dp, 0.dp) in the root"
rule.onNodeWithTag(CloseButtonTag)
    .assertPositionInRootIsEqualTo(expectedLeft = 320.dp, expectedTop = 0.dp)
```

If only one axis matters, use `assertLeftPositionInRootIsEqualTo` / `assertTopPositionInRootIsEqualTo` to avoid coupling the test to layout decisions on the other axis.

### Pattern: alignment line for baseline math

```kotlin
val baselineDp = rule.onNodeWithTag(LabelTag)
    .getAlignmentLinePosition(FirstBaseline)
require(!baselineDp.isUnspecified) { "Label has no first baseline" }
baselineDp.assertIsEqualTo(20.dp, "first baseline of label")
```

`getAlignmentLinePosition` returns `Dp.Unspecified` when the alignment line is not provided (`BoundsAssertions.kt:172-179`). Always check `isUnspecified` before comparing.

### Pattern: tolerance override for sub-dp precision

```kotlin
// Most tests want the default ½ dp tolerance:
rule.onNodeWithTag(IconTag).getUnclippedBoundsInRoot().width.assertIsEqualTo(24.dp, "icon width")

// Stricter tolerance when measuring a hand-aligned constant:
val width = rule.onNodeWithTag(IconTag).getUnclippedBoundsInRoot().width
width.assertIsEqualTo(expected = 24.dp, subject = "icon width", tolerance = 0.1.dp)
```

### Pattern: clipped vs unclipped — partial visibility

```kotlin
val unclipped = rule.onNodeWithTag(BannerTag).getUnclippedBoundsInRoot()
val clipped   = rule.onNodeWithTag(BannerTag).getBoundsInRoot()

// Banner laid out 200 dp tall but only 80 dp visible (rest clipped by parent):
unclipped.height.assertIsEqualTo(200.dp, "banner intrinsic height")
clipped.height.assertIsEqualTo(80.dp, "banner visible height")
```

For "is any of it visible", prefer `assertIsDisplayed()` — see `./asserting-node-state-and-text/SKILL.md`.

## Mandatory rules

- **MUST** assert in dp using the typed `assertWidthIsEqualTo` / `assertHeightIsEqualTo` / `assertPositionInRootIsEqualTo`. **MUST NOT** read `fetchSemanticsNode().size.width` and compare pixels.
- **MUST** use `getUnclippedBoundsInRoot()` for padding / gap / alignment math; **MUST** use `getBoundsInRoot()` only when the contract is "what the user sees after clipping".
- **MUST** pass a meaningful `subject` string to `Dp.assertIsEqualTo` so the failure message identifies which measurement failed.
- **MUST** prefer `assertHeightIsAtLeast(MinHeight + 1.dp)` over `assertHeightIsEqualTo(MinHeight + N.dp)` when the goal is "the layout grows past the minimum at large font scales".
- **MUST NOT** assume zero tolerance. Layout rounding produces sub-dp drift; rely on the half-dp default and override only when justified.
- **PREFERRED:** when an animation is in flight, pause `mainClock.autoAdvance = false` and step deterministically before reading bounds. Skydoves hot take #3.

## Verification

- [ ] No `node.size.width` / `node.size.height` reads remain. All dimension checks use the typed `assert*IsEqualTo` / `Dp.assertIsEqualTo`.
- [ ] Padding / gap math uses `getUnclippedBoundsInRoot`; clipped reads only appear with a comment explaining why.
- [ ] Every `Dp.assertIsEqualTo(...)` passes a non-empty `subject` string.
- [ ] Touch-target assertions use `assertTouchWidthIsEqualTo` / `assertTouchHeightIsEqualTo` rather than visual bounds when verifying accessibility constraints.
- [ ] `./gradlew :app:connectedDebugAndroidTest` passes; failure messages identify the specific failed measurement by `subject`.

## References

- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- Layout in Compose: https://developer.android.com/develop/ui/compose/layouts
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/BoundsAssertions.kt` — `assertWidthIsEqualTo`, `assertHeightIsAtLeast`, `assertPositionInRootIsEqualTo`, `getUnclippedBoundsInRoot`, `getBoundsInRoot`, `getAlignmentLinePosition`, `getFirstLinkBounds`, `Dp.assertIsEqualTo` (default tolerance ½ dp).
- `compose/material3/material3/src/androidDeviceTest/kotlin/androidx/compose/material3/ButtonTest.kt:202-226` — canonical "subtract two unclipped rects" padding test.
- `compose/material/material/src/androidDeviceTest/kotlin/androidx/compose/material/ChipTest.kt:226-233` — `assertHeightIsAtLeast(MinHeight + 1.dp)` for minimum-touch contracts.
