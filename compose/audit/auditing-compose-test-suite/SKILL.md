---
name: auditing-compose-test-suite
description: Use this skill to perform an end-to-end review of an existing Jetpack Compose UI test file or test suite. Sequences six audit phases (setup correctness, finder discipline, assertion strength, action correctness, time/idle correctness, debug output) and routes each finding to the precise sibling skill that fixes it. Produces a prioritized issue list — does NOT mutate code. Use when the user asks "review my Compose tests", "audit this test class", "is this test flaky", "why is this test slow on CI", "which Compose tests should I rewrite", or pastes a `*Test.kt` file and asks for feedback.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - test-audit
  - test-review
  - flaky-test
  - test-smell
  - testTag
  - performGesture
  - thread-sleep
  - autoAdvance
  - merged-tree
---

# Auditing a Compose Test Suite — Six-Phase Review, Then Route

This is an index-style skill. It orchestrates the rest of `compose-test-skills/` into a fixed-order audit pass over an existing test file or test suite. Each phase produces findings; each finding points at the sibling skill that owns the fix. The audit itself never mutates code — that is the job of the routed skills.

## When to use this skill

- The user pastes a `*Test.kt` (or several) and asks "what's wrong with this", "review this", "audit this".
- The user reports systemic flakes, slow CI, or "the test passes locally but fails on CI".
- The user is preparing a Compose feature for release and wants the test suite vetted.
- The user is migrating from v1 (`UnconfinedTestDispatcher`) to v2 (`StandardTestDispatcher`) and wants a checklist of likely-broken tests.
- The user inherited a test file from another team and is unsure what is correct vs accidentally working.

## When NOT to use this skill

- The user has a specific failing assertion and wants it fixed — jump directly to the relevant sibling (most often `../../debug/printing-the-semantics-tree/SKILL.md` for missed matchers, or `../../synchronization/testing-animations-deterministically/SKILL.md` for animation flakes).
- The user is starting a brand-new test from scratch — start with `../../setup/configuring-test-dependencies/SKILL.md`, then `../../setup/choosing-test-rule-vs-runtest/SKILL.md`, then `../../setup/setting-up-host-vs-device-tests/SKILL.md`.
- The user wants performance review of production composables — that is `compose-performance-skills`, not this repo.

## Prerequisites

- Read access to the full test file(s) under review. A single test method is too narrow to audit — patterns emerge across a class.
- The module's `build.gradle.kts` (for Phase 1 dep checks) and any `src/androidTest/AndroidManifest.xml` if the test uses a custom Activity.
- Familiarity with the skydoves hot takes encoded in `../docs/SPEC.md` §5 (testTag from production, default `useUnmergedTree = false`, `mainClock.autoAdvance = false` for animations, `waitUntil` is wall clock, `runOnIdle` for state mutations, prefer v2 entry points, `Thread.sleep` is a smell).

## Audit pipeline

Run the six phases in order. Stop and route to the matching sibling skill the moment a phase produces a finding — fixes in earlier phases sometimes invalidate findings in later ones (e.g. switching from v1 to v2 in Phase 1 can change Phase 5 idle behavior).

### Phase 1 — Setup correctness

What to check on the build/`@Rule`/source-set layer:

- Gradle: is `androidx.compose.ui:ui-test-manifest` on `debugImplementation` (instrumentation) / `testImplementation` (host)? Is `ui-test-junit4` on the matching configuration as the test class?
- Lint: is `TestManifestGradleConfiguration` (severity WARNING) being suppressed?
- Rule choice: `createComposeRule()` vs `createAndroidComposeRule<A>()` vs `createEmptyComposeRule()` — does the rule match the host? Is the custom Activity declared in `src/androidTest/AndroidManifest.xml`?
- v1 vs v2 imports: any `androidx.compose.ui.test.junit4.createComposeRule` or `androidx.compose.ui.test.runComposeUiTest` (deprecated WARNING)?
- Mixing surfaces: does the same test class declare both `@get:Rule val rule = createComposeRule()` AND a `runComposeUiTest { }` body?
- Source set: is the file in `src/test/` (host) or `src/androidTest/` (device)? Does its capability set match (no `captureToImage`, no `enableAccessibilityChecks`, no ripple-dependent assertions in host)?

Failure modes route to:

- Missing/misconfigured deps, `ActivityNotFoundException`, lint warning → `../../setup/configuring-test-dependencies/SKILL.md`.
- Wrong rule constructor, v1 imports, mixed surfaces, double `setContent` → `../../setup/choosing-test-rule-vs-runtest/SKILL.md`.
- Wrong source set, screenshot test on host, accessibility check on host → `../../setup/setting-up-host-vs-device-tests/SKILL.md`.

### Phase 2 — Finder discipline

What to check on each `onNode*` / `onAllNodes*` call:

- Finder choice: count `onNodeWithTag` vs `onNodeWithText` vs `onNodeWithContentDescription`. Heavy `onNodeWithText` usage on non-i18n strings is a fragility signal. Skydoves hot take #1: `Modifier.testTag("…")` belongs in production under a constant; tests find by tag, not text.
- Production tag presence: is the matched tag actually applied with `Modifier.testTag(...)` in the production composable? If absent, the finder will never match.
- `useUnmergedTree`: defaults to `false`. Is the developer flipping it without a stated reason? Is the test relying on a tag inside a merged subtree (e.g. tag on the `Text` inside a `Button`) and matching the wrong (parent) node?
- Selector chains: `onChild`/`onParent` propagate `useUnmergedTree`. Any chain that flips the flag mid-traversal is suspicious.
- `onAllNodes(...).filter(...)` vs custom `SemanticsMatcher`: is the test re-implementing a prebuilt matcher (`hasText`, `hasClickAction`, `hasContentDescription`)?

Failure modes route to:

- Tag absent in production, text-finder fragility → `../../finders/finding-nodes-by-tag-text-content/SKILL.md`.
- Wrong merged/unmerged choice or fragile tree navigation → `../../finders/traversing-the-semantics-tree/SKILL.md`.
- Hand-rolled matcher when a prebuilt exists → `../../finders/composing-semantics-matchers/SKILL.md`.
- Finder produces zero matches in CI → `../../debug/printing-the-semantics-tree/SKILL.md`.

### Phase 3 — Assertion strength

What to check on each `assert*` call:

- `assertExists()` where the test author meant `assertIsDisplayed()`: existence in the semantics tree is weaker than on-screen visibility.
- `assertCountEquals(n)` vs `assertAny(matcher)`: is the test brittle on collection size when it really means "at least one matches"?
- Bounds vs state: `assertWidthIsEqualTo` / `getUnclippedBoundsInRoot` are state checks against layout. Mixing them with screenshot assertions duplicates work.
- Text assertions: `assertTextEquals("Save")` on user-visible strings drifts across i18n; `assertTextEquals(context.getString(R.string.save))` is sturdier.
- `assertIsOn` / `assertIsOff` / `assertIsSelected`: these query the toggle/selectable semantics state. Are they being skipped in favor of weaker `assertExists`?
- `Dp.assertIsEqualTo` tolerance: defaults to ½ dp. Is the test asserting exact pixel equality where a tolerance is more honest?

Failure modes route to:

- `assertExists` overuse, brittle count vs intent, toggle/selectable state, text assertions → `../../assertions/asserting-node-state-and-text/SKILL.md`.
- Bounds checks → `../../assertions/asserting-bounds-and-dimensions/SKILL.md`.

### Phase 4 — Action correctness

What to check on each `perform*` call:

- `performGesture { }`: `@Deprecated` everywhere — replace with `performTouchInput { }`. This is non-negotiable; the SPEC forbids recommending `performGesture`.
- Coordinate system: `performTouchInput { down(Offset(x, y)) }` uses node-local coordinates. Tests that compute global coordinates and pass them to a node-relative DSL produce off-screen taps that silently miss.
- Touch DSL extensions: is the test re-implementing `click()` / `longClick()` / `swipeUp()` / `pinch(...)` from primitive `down`/`moveBy`/`up` calls? Use the high-level extensions when available.
- Text input: `performTextInput("foo")` appends; `performTextReplacement("foo")` replaces; `performTextClearance()` empties. Wrong choice produces "fooSave" instead of "foo".
- IME action: `performImeAction()` triggers the action set by `KeyboardOptions.imeAction`. Is the test using `Espresso.pressBack()` or a key event when `performImeAction()` is correct?
- Multi-modal: any test mixing `performTouchInput` and `performMouseInput` should consider `performMultiModalInput { touch { … }; mouse { … } }` for a single-pointer-event timeline.
- Scroll: `performScrollToNode`, `performScrollToIndex`, `performScrollToKey` are LazyList-aware — is the test using a brute-force `swipeUp` instead?

Failure modes route to:

- Deprecated `performGesture`, coordinate confusion, swipe primitives → `../../actions/injecting-touch-gestures/SKILL.md`.
- Text input/replacement/IME action → `../../actions/entering-text/SKILL.md`.
- Mouse, key, multi-modal → `../../actions/injecting-mouse-and-keyboard/SKILL.md`.
- Scroll containers (`performScrollToIndex`/`Key`/`Node`), `performClick` → `../../actions/clicking-and-scrolling/SKILL.md`.
- Higher-level LazyList patterns (visibility, `state.layoutInfo`, animated item placement) → `../../patterns/testing-lazy-lists/SKILL.md`.

### Phase 5 — Time and idle correctness

The flakiest phase. Check each test method end-to-end:

- `Thread.sleep(N)`: forbidden anywhere except screenshot tests waiting on the RenderThread for ripple completion. Skydoves hot take #7. Anywhere else, replace with `mainClock.advanceTimeBy(durationMs)` (test clock) or `rule.waitUntil(timeoutMillis) { … }` (wall clock).
- Animation tests: any test that triggers an animation MUST set `mainClock.autoAdvance = false` before `setContent`. Otherwise the installed `InfiniteAnimationPolicy` cancels infinite animations the moment they start with a `CancellationException`. Skydoves hot take #3.
- Clock vs wall clock: `mainClock.advanceTimeUntil { … }` uses the test clock; `rule.waitUntil(timeoutMillis) { … }` uses wall clock. Mixing them in the same test produces flakes — `waitUntil` will time out while `advanceTimeUntil` waits for a clock tick that never comes. Skydoves hot take #4.
- State mutations off-thread: any `state.value = X` from the test thread (not inside `runOnIdle`/`runOnUiThread`) races with the recomposer. Skydoves hot take #5.
- v2 queued work: after migrating to `androidx.compose.ui.test.v2.*`, `LaunchedEffect`s are queued under `StandardTestDispatcher`. Tests that previously relied on `UnconfinedTestDispatcher` eager dispatch may need an explicit `mainClock.advanceTimeBy(0)` or `runCurrent()` to drain.
- `waitUntil` timeout: defaults to 1000 ms — is the test using a longer timeout to mask a missing `IdlingResource` or to hide a wrong-clock issue?

Failure modes route to:

- `Thread.sleep`, missing `IdlingResource`, `waitUntil` confusion, off-thread state mutation, custom `IdlingResource` for external work → `../../synchronization/synchronizing-with-idle/SKILL.md`.
- Frame-rounding, `advanceTimeBy` granularity, v1/v2 dispatcher migration → `../../synchronization/controlling-the-test-clock/SKILL.md`.
- Animation flakes, `autoAdvance` missing, `InfiniteAnimationPolicy` `CancellationException` → `../../synchronization/testing-animations-deterministically/SKILL.md`.

### Phase 6 — Debug output

Sanity-check what the test reveals when it fails:

- `printToLog` / `printToString`: when a finder misses, does the test print the semantics tree to make the failure actionable, or does it just throw `AssertionError: failed: assertExists`?
- `fetchSemanticsNode`: any direct calls? They're an escape hatch — fine when used sparingly, but a sea of them indicates the test is bypassing the public matcher API.
- Failure messages: are matcher assertions on collections using the default error (lists all matched/unmatched nodes) or are they swallowed by a custom `try/catch`?
- Test names: do failing test names tell the developer what was supposed to happen, or do they read like `testButton1`?

Failure modes route to:

- Non-actionable failures, missing tree dump on miss, `fetchSemanticsNode` overuse → `../../debug/printing-the-semantics-tree/SKILL.md`.
- Accessibility check coverage gap, Robolectric no-op surprise → `../../debug/enabling-accessibility-checks/SKILL.md`.

## Audit report template

Produce the audit as a Markdown table the user can paste into a PR description. One row per finding. Phases run in order, but findings within a phase can be reordered by severity.

```markdown
| # | Phase | File:line | Severity | Finding | Routed skill |
|---|---|---|---|---|---|
| 1 | 1 setup | build.gradle.kts:42 | WARN | ui-test-manifest on androidTestImplementation | ../../setup/configuring-test-dependencies/SKILL.md |
| 2 | 1 setup | FeedTest.kt:18 | WARN | v1 createComposeRule import (deprecated) | ../../setup/choosing-test-rule-vs-runtest/SKILL.md |
| 3 | 2 finder | FeedTest.kt:34 | ERROR | onNodeWithText("Save") — no testTag in production | ../../finders/finding-nodes-by-tag-text-content/SKILL.md |
| 4 | 4 action | FeedTest.kt:51 | ERROR | performGesture { } — deprecated | ../../actions/injecting-touch-gestures/SKILL.md |
| 5 | 5 idle | FeedTest.kt:66 | ERROR | Thread.sleep(2000) before assertion | ../../synchronization/synchronizing-with-idle/SKILL.md |
| 6 | 5 idle | FeedTest.kt:88 | ERROR | autoAdvance not disabled before animation | ../../synchronization/testing-animations-deterministically/SKILL.md |
| 7 | 6 debug | FeedTest.kt:104 | NIT | assertExists with no printToLog on miss | ../../debug/printing-the-semantics-tree/SKILL.md |
```

Severities (use exactly these three):

- **ERROR** — test is broken, flaky, or asserts the wrong thing. Fix before merge.
- **WARN** — test runs but encodes a smell that will produce flakes/regressions. Fix this iteration if cheap.
- **NIT** — style/maintainability. Fix when adjacent code is touched.

Do NOT autofix. Hand the report to the user; the user (or a follow-up skill invocation against the routed skill) applies the fix.

## Patterns

### Pattern: ranking findings by blast radius

When the same `*Test.kt` produces several findings, prioritize:

1. **Phase 1 setup blockers first.** A test on the wrong source set or with mixed surfaces produces undefined behavior; fixing Phase 5 idle issues on top of it wastes effort.
2. **Phase 5 idle issues before Phase 4 actions.** A `Thread.sleep` masking a missing `IdlingResource` makes Phase 4 action issues invisible — once idle is correct, the action timing problems surface clearly.
3. **Phase 2 finder issues before Phase 3 assertions.** A finder that targets the wrong node makes every assertion downstream meaningless.
4. **Phase 6 debug output is always the last fix.** It improves diagnosis but does not change correctness.

### Pattern: high-signal smells to grep for first

Before reading the test line by line, run these greps to surface the load-bearing smells:

```bash
# Phase 1 — v1 imports (deprecated)
grep -RIn 'import androidx.compose.ui.test.junit4.createComposeRule' .
grep -RIn 'import androidx.compose.ui.test.runComposeUiTest' .

# Phase 1 — manifest dep on the wrong configuration
grep -RIn 'ui-test-manifest' build.gradle.kts settings.gradle.kts buildSrc/

# Phase 2 — finder usage ratio (skydoves hot take #1: tag-heavy is correct)
grep -RIcn 'onNodeWithTag\|onNodeWithText\|onNodeWithContentDescription' src/

# Phase 4 — deprecated gesture API
grep -RIn 'performGesture' src/

# Phase 5 — Thread.sleep smell
grep -RIn 'Thread.sleep' src/

# Phase 5 — animation tests missing autoAdvance = false
grep -RIn 'animateFloatAsState\|animateDpAsState\|animateColorAsState\|rememberInfiniteTransition' src/test src/androidTest

# Phase 5 — off-thread mutation pattern (heuristic — review hits manually)
grep -RIn '\.value = ' src/test src/androidTest | grep -v 'runOnIdle\|runOnUiThread'
```

The output of these greps populates 60-80 percent of a typical audit report before the first careful read.

### Pattern: when to stop the audit and re-route

If Phase 1 finds the test is in the wrong source set (e.g. a screenshot test in `src/test/`), STOP. Do not run Phases 2-6. The whole test will be rewritten when moved to `src/androidTest/` — auditing the current incarnation wastes effort. Hand off to `../../setup/setting-up-host-vs-device-tests/SKILL.md` and rerun the audit after the move.

Similarly, if Phase 1 finds a `runComposeUiTest { }` block sharing a class with a `@get:Rule createComposeRule()`, halt and route to `../../setup/choosing-test-rule-vs-runtest/SKILL.md`. The mixed environment makes every later finding ambiguous.

### Pattern: presenting the audit to the user

End the audit with a short prose summary, then the table. The summary names the top three findings and the single sibling skill the user should run next. Example:

```text
The FeedTest.kt suite has 7 findings. The two ERRORs that block correctness are
(1) a Thread.sleep(2000) before the search assertion (FeedTest.kt:66) and
(2) an animation test without mainClock.autoAdvance = false (FeedTest.kt:88).
Run ../../synchronization/testing-animations-deterministically/SKILL.md first;
the autoAdvance fix typically resolves the Thread.sleep finding by exposing
a mainClock.advanceTimeBy(durationMs) replacement.
```

## Mandatory rules

- **MUST NOT** mutate code during the audit. The audit produces findings; the routed sibling skill applies the fix.
- **MUST** run the six phases in order. Earlier-phase fixes invalidate later-phase findings; the order minimizes wasted effort.
- **MUST** route every finding to a single sibling skill. Findings without a routed skill are not actionable and should be omitted from the report.
- **MUST** flag every `Thread.sleep` outside of a screenshot/RenderThread context as ERROR. Skydoves hot take #7.
- **MUST** flag every animation test without `mainClock.autoAdvance = false` as ERROR. Skydoves hot take #3.
- **MUST** flag every v1 entry point (`androidx.compose.ui.test.junit4.createComposeRule`, `androidx.compose.ui.test.runComposeUiTest`) as WARN. Skydoves hot take #6.
- **MUST** flag every `performGesture { }` usage as ERROR. The API is `@Deprecated`; the SPEC forbids recommending it.
- **MUST NOT** invent findings. If a phase produces nothing, write "Phase N: no findings" — false-positive findings degrade the audit's signal.
- **MUST** use the audit report table format (Phase, File:line, Severity, Finding, Routed skill) so subsequent skills can parse the output.
- **PREFERRED:** when several findings cluster around one symptom (e.g. "test flakes on CI"), name the symptom in the prose summary and order the table so the routed sibling skill shows up first.

## Verification

- [ ] Every audit phase ran and produced either findings or "no findings".
- [ ] Every finding has a Severity (ERROR | WARN | NIT) and a routed sibling skill path.
- [ ] No code in the test files under review was mutated by the audit itself.
- [ ] The Phase 1 grep set ran and its output is reflected in the table.
- [ ] The audit report table compiles to valid Markdown (the user can paste it into a PR).
- [ ] If a Phase 1 setup blocker fired (wrong source set, mixed surfaces), Phases 2-6 were skipped and the audit halted with a re-route note.

## References

- Compose testing overview (Android Developers): https://developer.android.com/develop/ui/compose/testing
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- Compose testing setup: https://developer.android.com/develop/ui/compose/testing#setup
- Testing animations: https://developer.android.com/develop/ui/compose/animation/testing
- Semantics in Compose: https://developer.android.com/develop/ui/compose/accessibility/semantics
- Compose UI release notes: https://developer.android.com/jetpack/androidx/releases/compose-ui
- Sibling setup skills: `../../setup/configuring-test-dependencies/SKILL.md`, `../../setup/choosing-test-rule-vs-runtest/SKILL.md`, `../../setup/setting-up-host-vs-device-tests/SKILL.md`.
- Sibling finder skills: `../../finders/finding-nodes-by-tag-text-content/SKILL.md`, `../../finders/composing-semantics-matchers/SKILL.md`, `../../finders/traversing-the-semantics-tree/SKILL.md`.
- Sibling assertion skills: `../../assertions/asserting-node-state-and-text/SKILL.md`, `../../assertions/asserting-bounds-and-dimensions/SKILL.md`.
- Sibling action skills: `../../actions/clicking-and-scrolling/SKILL.md`, `../../actions/injecting-touch-gestures/SKILL.md`, `../../actions/injecting-mouse-and-keyboard/SKILL.md`, `../../actions/entering-text/SKILL.md`.
- Sibling synchronization skills: `../../synchronization/controlling-the-test-clock/SKILL.md`, `../../synchronization/testing-animations-deterministically/SKILL.md`, `../../synchronization/synchronizing-with-idle/SKILL.md`.
- Sibling pattern skills: `../../patterns/structuring-a-compose-test/SKILL.md`, `../../patterns/testing-lazy-lists/SKILL.md`, `../../patterns/testing-state-restoration/SKILL.md`.
- Sibling interop skill: `../../interop/testing-with-espresso-interop/SKILL.md`.
- Sibling debug skills: `../../debug/printing-the-semantics-tree/SKILL.md`, `../../debug/enabling-accessibility-checks/SKILL.md`.
- skydoves — compose-performance-skills (sibling repo for production-side audits): https://github.com/skydoves/compose-performance-skills.
