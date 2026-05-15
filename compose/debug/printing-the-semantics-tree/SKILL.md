---
name: printing-the-semantics-tree
description: Use this skill to diagnose "no node matched", "found N nodes", and "useUnmergedTree" failures by dumping the actual semantics tree with `printToLog(tag, maxDepth)` and `printToString(maxDepth)`. Covers the output grammar (`Node #<id> at (l, t, r, b)px, Tag: '...'`, alphabetically sorted config, `Actions = [...]`, `[FlagKey]` for `Unit`-valued semantics, `MergeDescendants`, `ClearAndSetSemantics`), the framework hint that suggests `useUnmergedTree = true` when matches are found only in the unmerged tree, and `fetchSemanticsNode` / `fetchSemanticsNodes` for custom matchers. Use when the developer reports "tag not found", "multiple nodes matched", "merged tree vs unmerged tree", "test fails with assertion error and a long error message", or wants to debug a finder without adding `Thread.sleep`.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - printToLog
  - printToString
  - semantics-tree
  - useUnmergedTree
  - fetchSemanticsNode
  - debug-finder
  - tag-not-found
  - multiple-nodes-matched
---

# Printing the Semantics Tree — When a Finder Fails, Dump the Tree

`onNodeWithTag` failures are structural — the tag is wrong, the matcher is on the merged tree but the tag is on a leaf, or the node is genuinely not composed yet. None of those are timing problems, so `Thread.sleep` does nothing. This skill teaches the agent to diagnose finder failures by dumping the semantics tree with `printToLog` / `printToString` and reading the framework's built-in "found in the unmerged tree" hint.

## When to use this skill

- The test fails with `"Failed: assertExists. Reason: Expected exactly '1' node but could not find any node that satisfies: …"`.
- The test fails with `"Reason: Expected exactly '1' node but found 'N' nodes that satisfy: …"`.
- The error message ends with `"…were found in the unmerged tree. If you really wanted to match against merged tree, use useUnmergedTree = true."` and the developer is confused.
- The developer is writing a custom matcher and needs direct access to a `SemanticsNode`.
- A test was fixed by adding `Thread.sleep` — the underlying problem is a wrong tag, not a timing issue. Replace with this skill's pattern. See skydoves directive #7.

## When NOT to use this skill

- The symptom is timing (animation has not finished, an idling resource is busy). Use `../../synchronization/synchronizing-with-idle/SKILL.md` or `../../synchronization/testing-animations-deterministically/SKILL.md`.
- The finder is wrong because the test searches by text instead of by tag. Fix with `../../finders/finding-nodes-by-tag-text-content/SKILL.md` (skydoves directive #1).
- The diagnosis involves accessibility violations rather than node lookup. Use `../enabling-accessibility-checks/SKILL.md`.

## Prerequisites

- `androidx.compose.ui:ui-test` on `androidTestImplementation` or `testImplementation` (the API lives in `commonMain` so it is available in both source sets).
- An active `ComposeTestRule` or `ComposeUiTest` (no extra setup beyond the standard skeleton).

## Workflow

- [ ] **1. When a finder fails, dump the merged tree first.** Add the line right after `rule.setContent { … }` (or right before the failing finder).

```kotlin
rule.onRoot().printToLog("DEBUG")
```

The signature is `fun SemanticsNodeInteraction.printToLog(tag: String, maxDepth: Int = Int.MAX_VALUE)` — full subtree by default for a single-node receiver. Cited at `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Output.kt:62-69`. View the output via `adb logcat -s DEBUG`.

- [ ] **2. If the merged dump does not show the expected node, dump the unmerged tree.**

```kotlin
rule.onRoot(useUnmergedTree = true).printToLog("DEBUG")
```

Inner descendants of merging composables (e.g. an `Icon` inside a `Button`) only appear in the unmerged tree.

- [ ] **3. Read the output grammar.** The format is fixed by `Output.kt:135-296`:

```
Node #<id> at (l=<L>, t=<T>, r=<R>, b=<B>)px, Tag: '<tag>'
  <Sorted ConfigKey> = '<value>'
  ...
  [<FlagKey1>, <FlagKey2>]            // keys whose value is Unit
  Actions = [<ActionKey1>, ...]        // AccessibilityActions and Function values
  MergeDescendants = 'true'            // when isMergingSemanticsOfDescendants
  ClearAndSetSemantics = 'true'        // when isClearingSemantics
   |-Node #<childId> at (...)px        // children indented with " |-"
```

- Top line: `Node #` + id + bounds + (optionally) `Tag: '<testTag>'`.
- Config entries are alphabetically sorted by key name.
- `AccessibilityAction` and `Function<*>` values are summarized as `Actions = [keyName, …]`.
- `Unit` values (e.g. `Disabled`, `IsTraversalGroup`) are summarized as `[Disabled, IsTraversalGroup, …]`.
- `MergeDescendants = 'true'` and `ClearAndSetSemantics = 'true'` flags appear at the bottom of a node's block.

- [ ] **4. Read the framework's built-in hint.** When `onNode(...)` finds zero matches in the merged tree but the same matcher would have matched in the unmerged tree, the error message embeds a "These nodes were found in the unmerged tree" block. Cited at `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/SemanticsNodeInteraction.kt:184-194`. When the developer sees that, the fix is to flip `useUnmergedTree = true` on the failing finder. See `../../finders/finding-nodes-by-tag-text-content/SKILL.md` for the merged/unmerged decision.

- [ ] **5. For custom matchers, use `fetchSemanticsNode` / `fetchSemanticsNodes` directly.** These return the underlying `SemanticsNode`(s) for direct introspection.

```kotlin
val node = rule.onNodeWithTag("title").fetchSemanticsNode()
assertThat(node.config[SemanticsProperties.Text]).isNotEmpty()

val all = rule.onAllNodesWithTag("row").fetchSemanticsNodes(
    atLeastOneRootRequired = true,
    errorMessageOnFail = "expected at least one row",
)
```

Single-node signature: `fun fetchSemanticsNode(errorMessageOnFail: String? = null): SemanticsNode`. Collection signature: `fun fetchSemanticsNodes(atLeastOneRootRequired: Boolean = true, errorMessageOnFail: String? = null): List<SemanticsNode>`.

- [ ] **6. For collection finders, default `maxDepth` is 0 (no children).** Override when the developer wants the full subtree per match.

```kotlin
rule.onAllNodesWithTag("row").printToLog("DEBUG", maxDepth = Int.MAX_VALUE)
```

Cited at `Output.kt:83-116`.

- [ ] **7. Capture the dump as a String for assertion-based debugging.** Useful in CI logs.

```kotlin
val tree = rule.onRoot(useUnmergedTree = true).printToString()
println(tree) // or attach to a test report
```

## Patterns

### Pattern: `Thread.sleep` to "wait for the node" instead of dumping the tree

```kotlin
// WRONG
@Test
fun appears() {
    rule.setContent { MyScreen() }
    Thread.sleep(2000)                                    // hide the symptom
    rule.onNodeWithTag("submit").assertIsDisplayed()      // still fails
}
// WRONG because: Thread.sleep desyncs from MainTestClock and does not advance composition.
// If the node is missing, more wall time will not produce it. The cause is structural.
// Skydoves directive #7: Thread.sleep is a smell.
```

```kotlin
// RIGHT
@Test
fun appears() {
    rule.setContent { MyScreen() }
    rule.onRoot(useUnmergedTree = true).printToLog("DEBUG")   // see what's actually present
    rule.onNodeWithTag("submit").assertIsDisplayed()
}
```

### Pattern: "found in the unmerged tree" hint

```kotlin
// WRONG — the test ignores the framework hint
@Test
fun iconExists() {
    rule.setContent { Button(onClick = {}) { Icon(Icons.Default.Add, modifier = Modifier.testTag("add-icon")) } }
    rule.onNodeWithTag("add-icon").assertExists()
    // AssertionError: ... 0 matches.
    // These nodes were found in the unmerged tree:
    //   Node #4 ... Tag: 'add-icon'
    // If you really wanted to match against merged tree, use useUnmergedTree = true.
}
// WRONG because: a Button merges descendants. The Icon's tag is collapsed into the Button
// node in the merged tree. The error message names the fix in plain English.
```

```kotlin
// RIGHT
@Test
fun iconExists() {
    rule.setContent { Button(onClick = {}) { Icon(Icons.Default.Add, modifier = Modifier.testTag("add-icon")) } }
    rule.onNodeWithTag("add-icon", useUnmergedTree = true).assertExists()
}
```

### Pattern: ambiguous "found 3 nodes" — disambiguate via dump

```kotlin
// WRONG
rule.onNodeWithTag("row").assertIsDisplayed()
// AssertionError: Reason: Expected exactly '1' node but found '3' nodes that satisfy:
// (TestTag = 'row')
```

```kotlin
// RIGHT — dump first, then narrow the matcher
rule.onAllNodesWithTag("row").printToLog("DEBUG", maxDepth = Int.MAX_VALUE)
// Read the dump to find a stable disambiguator (e.g. parent tag, contentDescription),
// then narrow:
rule.onAllNodesWithTag("row")
    .filterToOne(hasAnyAncestor(hasTestTag("section-completed")))
    .assertIsDisplayed()
```

### Pattern: `fetchSemanticsNode` for custom assertions

```kotlin
// RIGHT
@Test
fun rowConfigContainsCustomKey() {
    rule.setContent { /* … */ }
    val node = rule.onNodeWithTag("row").fetchSemanticsNode("missing 'row'")
    val custom: String? = node.config.getOrNull(MyCustomKey)
    assertThat(custom).isEqualTo("expected")
}
```

## Mandatory rules

- **MUST** dump the semantics tree with `rule.onRoot().printToLog("DEBUG")` (or the unmerged variant) the moment a finder fails. The default `maxDepth` for a single-node receiver is `Int.MAX_VALUE`, so the full subtree is printed without further configuration. Cited at `Output.kt:43, 65`.
- **MUST** read the framework hint `"…were found in the unmerged tree. If you really wanted to match against merged tree, use useUnmergedTree = true."` as a directive — flip `useUnmergedTree = true` on the failing finder. Cited at `SemanticsNodeInteraction.kt:184-194`.
- **MUST NOT** add `Thread.sleep(…)` to "give the tree time to load". A finder failure is structural; more wall time does not produce the node. Skydoves directive #7. The single legitimate use is screenshot/RenderThread waits.
- **MUST** override `maxDepth` to a non-zero value when calling `printToLog` on a `SemanticsNodeInteractionCollection` — the collection default is `0` (no children). Cited at `Output.kt:85, 112`.
- **PREFERRED:** start the diagnosis with the merged tree, then escalate to `useUnmergedTree = true` only if the expected node is collapsed. Skydoves directive #2: keep `useUnmergedTree = false` by default.
- **PREFERRED:** for custom matchers, prefer `fetchSemanticsNode(errorMessageOnFail = "…")` so the failure message identifies the test, not just the assertion line.

## Verification

- [ ] When a finder fails, the test (or the agent's interactive debugging) emits at least one `printToLog("DEBUG")` line for the relevant subtree.
- [ ] No `Thread.sleep` was added to fix a finder problem.
- [ ] If the dump shows the node only under `useUnmergedTree = true`, the finder has been updated to pass that flag.
- [ ] If multiple matches were found, the finder has been narrowed (via `filterToOne`, `hasAnyAncestor`, a parent `testTag`, etc.) — not by adding sleep or assert-attempts.
- [ ] Custom matchers use `fetchSemanticsNode(errorMessageOnFail = "…")` with a meaningful diagnostic message.
- [ ] In CI, the test logs include the dump output (developer collects logcat for the test process).

## References

- `printToLog` / `printToString` source: `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Output.kt:41-116`
- Output grammar (rect, sorted config, actions, flags): `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Output.kt:135-296`
- "Found in the unmerged tree" hint: `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/SemanticsNodeInteraction.kt:178-194`
- `fetchSemanticsNode(s)` signature: `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/SemanticsNodeInteraction.kt`
- Semantics in Compose: https://developer.android.com/develop/ui/compose/accessibility/semantics
- Compose testing — finders: https://developer.android.com/develop/ui/compose/testing#finders
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
