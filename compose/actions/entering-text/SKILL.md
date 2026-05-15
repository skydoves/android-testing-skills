---
name: entering-text
description: Use this skill to drive Jetpack Compose text fields from tests with the text-specific actions — performTextInput (insert at cursor via the InsertTextAtCursor semantics action), performTextReplacement (clear + replace via SetText), performTextClearance (replace with empty string), performTextInputSelection (set TextRange selection), and performImeAction (fire the configured IME action such as Next, Done, Search). Covers the auto-focus path, the enabled / editable / focusable preconditions, the append vs replace gotcha that produces "helloworld" instead of "world", and the IME-action focus chain (Next advances focus to the next focusable). Use when the developer asks "how do I type into a TextField in a test", "test an IME Next button", "test a clear-text action", "BasicTextField won't accept input", "text input duplicated in test", or reports "Failed to perform text input" / "Failed to perform IME action" / "Default ImeAction" errors.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - performTextInput
  - performTextReplacement
  - performTextClearance
  - performTextInputSelection
  - performImeAction
  - text-field-testing
  - basic-text-field
  - ime-action
---

# Entering Text — performTextInput, performTextReplacement, performImeAction

Compose text fields expose semantics actions for IME-style edits. The test API wraps those actions so a single call types text, replaces text, clears text, sets a selection, or fires the IME action. Every entry point auto-focuses the node when needed and asserts that the node is enabled and editable, surfacing precise errors when one of those preconditions fails.

## When to use this skill

- The test types into `TextField`, `OutlinedTextField`, `BasicTextField`, or any composable that exposes the `InsertTextAtCursor` / `SetText` semantics actions.
- The test fires an IME action — `ImeAction.Next` to advance focus, `ImeAction.Done` to dismiss the keyboard, `ImeAction.Search` / `Send` / `Go` to trigger a form submit.
- The test must clear an existing value, or replace the full value with new text.
- The test asserts cursor or selection state with `performTextInputSelection(TextRange(...))`.

## When NOT to use this skill

- The test sends an arbitrary key shortcut into a focused field (e.g. `Ctrl+A`, `Tab`, arrow keys). Use `../injecting-mouse-and-keyboard/SKILL.md`.
- The test only checks the displayed text without typing — assert via `../../assertions/asserting-node-state-and-text/SKILL.md`.
- A finder cannot locate the text field. Verify the matcher first via `../../finders/finding-nodes-by-tag-text-content/SKILL.md`.
- The test asserts state restoration of the typed text after rotation. See `../../setup/configuring-test-dependencies/SKILL.md` and `StateRestorationTester`.

## Prerequisites

- `androidx.compose.ui:ui-test-junit4` (or `ui-test` for `runComposeUiTest`) configured per `../../setup/configuring-test-dependencies/SKILL.md`.
- The `TextField` carries `Modifier.testTag("…")` from production code so the test selects it deterministically.
- The field is `enabled = true`. `performTextInput` calls `assert(isEnabled())` before doing anything (TextActions.kt:125).
- For `performImeAction`: the field declares both `ImeAction != ImeAction.Default` (via `KeyboardOptions(imeAction = …)`) and an `OnImeAction` handler (via `KeyboardActions(...)` or default keyboard actions). The action assertion checks both (TextActions.kt:99-100).

## Workflow

1. **Type text at the cursor** — `performTextInput(text)` (TextActions.kt:37-41). Internally invokes `SemanticsActions.InsertTextAtCursor` with `AnnotatedString(text)`. Auto-focuses the node first when it is not focused (`getNodeAndFocus` — TextActions.kt:119-138). Throws `AssertionError("Failed to perform text input.")` if the node is not enabled, not focusable, or does not advertise `InsertTextAtCursor`.

```kotlin
rule.onNodeWithTag(Tag).performTextInput("Hello ")
```

2. **Replace the entire text** — `performTextReplacement(text)` (TextActions.kt:81-84). Invokes `SemanticsActions.SetText`, which clears existing text and inserts new text. Use this when the field already has a value the test wants to overwrite. **MUST NOT** be used to "type more characters" — see the WRONG/RIGHT pair below.

```kotlin
rule.onNodeWithTag(Tag).performTextReplacement("New value")
```

3. **Clear the text** — `performTextClearance()` (TextActions.kt:28-30). Sugar for `performTextReplacement("")`. The field must be focusable + have `SetText`.

```kotlin
rule.onNodeWithTag(Tag).performTextClearance()
```

4. **Set the selection / cursor** — `performTextInputSelection(selection: TextRange, relativeToOriginalText: Boolean = true)` (TextActions.kt:61-72). Maps to `SemanticsActions.SetSelection`. When `relativeToOriginalText = true`, indices refer to the original (untransformed) text; `false` makes them refer to the post-`VisualTransformation` text.

```kotlin
rule.onNodeWithTag(Tag).performTextInput("Hello World")
rule.onNodeWithTag(Tag).performTextInputSelection(TextRange(0, 5)) // selects "Hello"
```

5. **Fire the IME action** — `performImeAction()` (TextActions.kt:97-117). Pre-conditions, in order:
   - `assert(hasPerformImeAction())` — the field declares `OnImeAction`.
   - `assert(!hasImeAction(ImeAction.Default))` — the action is something other than the default.
   - `tryPerformAccessibilityChecks()`.
   - Auto-focus via `getNodeAndFocus(requireEditable = false)`.
   - Invoke `OnImeAction` and require it returned `true`; otherwise throw `AssertionError("Failed to perform IME action, handler returned false.")`.

```kotlin
rule.onNodeWithTag(InitialFieldTag).performImeAction()
```

6. **Pair with `assertTextEquals`** to confirm the expected post-state. The merged-tree default applies to text assertions exactly as to finders — see `../../assertions/asserting-node-state-and-text/SKILL.md`.

## Patterns

### Pattern: Append (`performTextInput`) vs replace (`performTextReplacement`)

The most common gotcha — the two APIs look interchangeable but compose differently.

```kotlin
// WRONG
rule.onNodeWithTag(Tag).performTextInput("hello")
rule.onNodeWithTag(Tag).performTextInput("world")
rule.onNodeWithTag(Tag).assertTextEquals("world")
// WRONG because: performTextInput maps to InsertTextAtCursor, not SetText. The cursor
// remains at the end of "hello" after the first call, so the second call inserts
// "world" there. Final value is "helloworld" — assertion fails.
```

```kotlin
// RIGHT
rule.onNodeWithTag(Tag).performTextInput("hello")
rule.onNodeWithTag(Tag).performTextReplacement("world")
rule.onNodeWithTag(Tag).assertTextEquals("world")
```

`performTextReplacement` calls `SemanticsActions.SetText` (TextActions.kt:83), which clears existing characters before inserting the new text — equivalent to the IME's "set composing region to whole text and replace" path. Use it any time the second call should overwrite, not append.

### Pattern: Canonical insert + assert

From `BasicTextFieldTest.kt:171-194`:

```kotlin
@Test
fun textFieldState_textChange_updatesState() {
    val state = TextFieldState("Hello ", TextRange(Int.MAX_VALUE))
    inputMethodInterceptor.setTextFieldTestContent {
        BasicTextField(state = state, modifier = Modifier.fillMaxSize().testTag(Tag))
    }

    rule.onNodeWithTag(Tag).performTextInput("World!")

    rule.runOnIdle { assertThat(state.text.toString()).isEqualTo("Hello World!") }
}

@Test
fun textFieldState_textChange_updatesSemantics() {
    val state = TextFieldState("Hello ", TextRange(Int.MAX_VALUE))
    inputMethodInterceptor.setTextFieldTestContent {
        BasicTextField(state = state, modifier = Modifier.fillMaxSize().testTag(Tag))
    }

    rule.onNodeWithTag(Tag).performTextInput("World!")

    rule.onNodeWithTag(Tag).assertTextEquals("Hello World!")
}
```

The cursor is positioned at `TextRange(Int.MAX_VALUE)` (end of text) when constructing `TextFieldState`. `performTextInput("World!")` inserts at that cursor — final value `"Hello World!"`. State reads happen inside `runOnIdle` (skydoves hot take #5).

### Pattern: Replace then assert — `BasicTextFieldTest.kt:442-466`

```kotlin
@Test
fun textField_whenStateObjectChanges_restartsInput() {
    val state1 = TextFieldState("Hello")
    val state2 = TextFieldState("World")
    var toggleState by mutableStateOf(true)
    val state by derivedStateOf { if (toggleState) state1 else state2 }
    inputMethodInterceptor.setTextFieldTestContent {
        BasicTextField(
            state = state,
            enabled = true,
            modifier = Modifier.fillMaxSize().testTag(Tag),
        )
    }

    with(rule.onNodeWithTag(Tag)) {
        performTextReplacement("Compose")
        assertTextEquals("Compose")
    }
    toggleState = !toggleState
    with(rule.onNodeWithTag(Tag)) {
        performTextReplacement("Compose2")
        assertTextEquals("Compose2")
    }
    assertThat(state1.text.toString()).isEqualTo("Compose")
    assertThat(state2.text.toString()).isEqualTo("Compose2")
}
```

`performTextReplacement` overwrites whatever the field's current value happens to be — making it the right call when state is rotated underneath.

### Pattern: Clearing then refilling

```kotlin
rule.onNodeWithTag(Tag).performTextClearance()
rule.onNodeWithTag(Tag).performTextInput("Different")
rule.onNodeWithTag(Tag).assertTextEquals("Different")
```

`performTextClearance` is sugar for `performTextReplacement("")` (TextActions.kt:28-30). After it runs, the cursor is at position 0, so a follow-up `performTextInput` simply inserts at the start.

### Pattern: IME-action focus chain (`Next` advances focus)

From `DefaultKeyboardActionsTest.kt:130-149`:

```kotlin
// Show keyboard.
rule.onNodeWithTag(initialTextField).performClick()
inputMethodInterceptor.assertSessionActive()
keyboardController.show()

// Act.
rule.onNodeWithTag(initialTextField).performImeAction()

// Assert.
when (imeAction) {
    Next -> {
        // Focus moves to the next item.
        assertThat(focusState1).isFalse()
        assertThat(focusState2).isFalse()
        assertThat(focusState3).isTrue()
    }
    Previous -> {
        // Focus moves to the previous item.
        assertThat(focusState1).isTrue()
        assertThat(focusState2).isFalse()
        assertThat(focusState3).isFalse()
    }
    Done -> {
        // No change to focus state.
        assertThat(focusState1).isFalse()
        assertThat(focusState2).isTrue()
        assertThat(focusState3).isFalse()

        // Software keyboard is hidden.
        keyboardController.assertHidden()
    }
}
```

Production must declare a non-default `ImeAction` for `performImeAction` to be valid:

```kotlin
BasicTextField(
    state = state,
    keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
    modifier = Modifier.testTag(InitialFieldTag),
)
```

`performImeAction` asserts `!hasImeAction(ImeAction.Default)` first (TextActions.kt:100). Forgetting the `keyboardOptions` parameter yields `AssertionError("Failed to perform IME action.")`.

### Pattern: Set selection before replacing a substring

```kotlin
rule.onNodeWithTag(Tag).performTextInput("Hello World")
rule.onNodeWithTag(Tag).performTextInputSelection(TextRange(6, 11)) // select "World"
rule.onNodeWithTag(Tag).performTextInput("Compose")              // selection replaced
rule.onNodeWithTag(Tag).assertTextEquals("Hello Compose")
```

`SemanticsActions.SetSelection` honours the same path the IME uses, so a follow-up `InsertTextAtCursor` replaces the active selection with the new text. Pass `relativeToOriginalText = false` if the field installs a `VisualTransformation` (e.g. password mask, currency formatter) and the indices were computed against the displayed string.

### Pattern: Disabled or non-editable field — clear error

If the field is `enabled = false` or the matcher resolves to a `Text` instead of a `TextField`, `performTextInput` throws `AssertionError("Failed to perform text input.")`. The chain is:

1. `tryPerformAccessibilityChecks()`.
2. `assert(isEnabled())`.
3. `assert(hasRequestFocusAction())`.
4. `assert(hasSetTextAction())` and `assert(hasInsertTextAtCursorAction())` (TextActions.kt:127-129).

When the assertion fires, dump the semantics tree via `../../debug/printing-the-semantics-tree/SKILL.md` to confirm the matcher is not selecting a static `Text`.

## Mandatory rules

- **MUST** use `performTextReplacement` to overwrite an existing value. **MUST NOT** chain two `performTextInput` calls expecting the second to replace the first — they concatenate at the cursor.
- **MUST** declare a non-default `ImeAction` (e.g. `KeyboardOptions(imeAction = ImeAction.Next)`) before calling `performImeAction`. The API asserts `!hasImeAction(ImeAction.Default)` (TextActions.kt:100) and throws otherwise.
- **MUST** select the field by `Modifier.testTag(...)` whose value is a constant in production code — skydoves hot take #1.
- **MUST** route post-input state assertions through `rule.runOnIdle { … }` (skydoves hot take #5). The recomposer needs to settle before reading `TextFieldState.text`.
- **MUST NOT** call `Thread.sleep` to wait for IME events. Compose's idle observation includes the text-input pipeline; the assertion will be ready when `runOnIdle` returns.
- **MUST NOT** simulate text entry by injecting individual `Key` events through `performKeyInput { pressKey(Key.A) }` unless the test specifically exercises the hardware-keyboard path. The semantics actions are the canonical IME path.
- **PREFERRED:** keep `performTextInputSelection` calls explicit about `relativeToOriginalText`. The default (`true`) is correct for most cases; flip it only when the field installs a `VisualTransformation`.

## Verification

- [ ] Every test that overwrites field content uses `performTextReplacement(...)` or `performTextClearance()` followed by `performTextInput(...)`. No back-to-back `performTextInput` calls expecting replacement semantics.
- [ ] Every `performImeAction` call targets a field that declares `KeyboardOptions(imeAction = ImeAction.Next | Done | Search | …)` in production source.
- [ ] State or text assertions following an input action run inside `rule.runOnIdle { … }`.
- [ ] No `Thread.sleep` in the test method.
- [ ] `./gradlew :app:connectedDebugAndroidTest` (or `:app:testDebugUnitTest` for Robolectric) passes for the test under change.

## References

- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- BasicTextField docs: https://developer.android.com/reference/kotlin/androidx/compose/foundation/text/package-summary#BasicTextField(androidx.compose.foundation.text.input.TextFieldState,androidx.compose.ui.Modifier,kotlin.Boolean,kotlin.Boolean,androidx.compose.ui.text.TextStyle,androidx.compose.foundation.text.KeyboardOptions,androidx.compose.foundation.text.KeyboardActions,kotlin.Boolean,kotlin.Int,kotlin.Int,androidx.compose.foundation.text.input.InputTransformation,androidx.compose.ui.graphics.SolidColor,androidx.compose.foundation.text.input.OutputTransformation,androidx.compose.foundation.interaction.MutableInteractionSource,androidx.compose.foundation.text.input.TextFieldDecorator)
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/TextActions.kt` — `performTextClearance` (TextActions.kt:28), `performTextInput` (TextActions.kt:37), `performTextInputSelection` (TextActions.kt:61), `performTextReplacement` (TextActions.kt:81), `performImeAction` (TextActions.kt:97), `getNodeAndFocus` precondition chain (TextActions.kt:119-138).
- `compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/Filters.kt` — `hasPerformImeAction`, `hasImeAction`, `hasSetTextAction`, `hasInsertTextAtCursorAction`, `isEnabled`, `hasRequestFocusAction`.
- `compose/foundation/foundation/src/androidDeviceTest/.../BasicTextFieldTest.kt` — canonical `performTextInput` + `assertTextEquals` (BasicTextFieldTest.kt:171-194), `performTextReplacement` (BasicTextFieldTest.kt:442-466).
- `compose/foundation/foundation/src/androidDeviceTest/.../DefaultKeyboardActionsTest.kt` — `performImeAction` + IME-action focus chain (DefaultKeyboardActionsTest.kt:130-149).
- skydoves — compose-performance-skills: https://github.com/skydoves/compose-performance-skills
