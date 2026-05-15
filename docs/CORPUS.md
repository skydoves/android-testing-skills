# android-testing-skills — Research Corpus

This file is the distilled knowledge base for `android-testing-skills`. Every fact, API signature,
and behavioral note here is grounded in a file path inside `androidx/`, an authoritative external
doc, or one of the deep research reports in `tasks/research/`. Skill authors **MUST** pull facts
and citations from this corpus so vocabulary and claims stay consistent across skills.

The repository organises skills into **five sets**:

- **`compose/`** — Jetpack Compose UI testing (22 skills). Authoritative source: `androidx/compose/ui/ui-test*`.
- **`fundamentals/`** — testing pyramid, what to test, test doubles, strategies, source sets. Authoritative source: developer.android.com/training/testing/fundamentals + `tasks/research/R8-android-fundamentals.md`.
- **`jvm-tests/`** — JVM unit tests: JUnit4, Mockito, MockK, kotlinx-coroutines-test, Turbine, Robolectric. See `tasks/research/R1`, `R5`, `R6`, `R7`.
- **`instrumentation/`** — on-device tests: AndroidJUnit4, ActivityScenario, FragmentScenario, Espresso, UiAutomator. See `tasks/research/R1`, `R2`, `R3`, `R4`.
- **`adb/`** — ADB-driven device E2E. See `tasks/research/A1`, `A2`, `A3`.

Per-set deep research reports under `tasks/research/` are the long form. This CORPUS file is the
short form — what every skill author needs at their fingertips.

---

## A. Canonical external references

Cite these directly in SKILL.md "References" sections.

### Android Developers (official testing docs)

- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
- Compose testing setup: https://developer.android.com/develop/ui/compose/testing#setup
- Compose testing cheat sheet: https://developer.android.com/develop/ui/compose/testing-cheatsheet
- Semantics in Compose: https://developer.android.com/develop/ui/compose/accessibility/semantics
- Testing animations: https://developer.android.com/develop/ui/compose/animation/testing
- Compose Multiplatform testing: https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-test.html

### Library coordinates (released API surface)

- `androidx.compose.ui:ui-test` — common test API (finders, matchers, assertions, actions, `MainTestClock`, `IdlingResource`, `ComposeUiTest`).
- `androidx.compose.ui:ui-test-junit4` — JUnit4 surface (`ComposeTestRule`, `createComposeRule`, `StateRestorationTester`).
- `androidx.compose.ui:ui-test-manifest` — `debugImplementation` only; merges an `<activity>` entry for `androidx.activity.ComponentActivity` so `createComposeRule()` can launch it.
- `androidx.compose.ui:ui-test-accessibility` — `enableAccessibilityChecks` for `ComposeUiTest` (`@RequiresApi(34)`).
- `androidx.compose.ui:ui-test-junit4-accessibility` — same for `ComposeTestRule` / `AndroidComposeTestRule`.
- Release notes: https://developer.android.com/jetpack/androidx/releases/compose-ui

### Background reading (community)

- Adam Powell — "How does compose-runtime test work?": https://issuetracker.google.com (search "compose ui test")
- Chris Banes — Compose testing posts: https://chrisbanes.me/tags/jetpack-compose/
- Android Developers blog — Testing posts tag: https://android-developers.googleblog.com/search/label/testing

### skydoves OSS context

- compose-performance-skills (sibling repo): https://github.com/skydoves/compose-performance-skills
- compose-effects: https://github.com/skydoves/compose-effects
- compose-stability-analyzer: https://github.com/skydoves/compose-stability-analyzer

---

## B. Primary source — androidx file map

Every skill MUST ground API claims in one of these files. Paths are relative to
`/Users/jaewoong/Desktop/projects/study/androidx/`. Subfolder shorthand `ui-test-cm` =
`compose/ui/ui-test/src/commonMain/kotlin/androidx/compose/ui/test/`.

### `ui-test` (commonMain — multiplatform API surface)

| File | Owns |
|---|---|
| `ui-test-cm/Finders.kt` | `onNodeWithTag`/`Text`/`ContentDescription`, `onAllNodesWith*`, `onRoot` (all `@CheckResult` thin wrappers over `SemanticsNodeInteractionsProvider.onNode`/`onAllNodes`) |
| `ui-test-cm/SemanticsNodeInteractionsProvider.kt` | `onNode(matcher, useUnmergedTree=false)`, `onAllNodes(matcher, useUnmergedTree=false)` |
| `ui-test-cm/SemanticsNodeInteraction.kt` | Lazy single-node and collection handles, `assertExists`, `assertDoesNotExist`, `assertIsDeactivated`, `fetchSemanticsNode`, collection `[index]` |
| `ui-test-cm/Selectors.kt` | `onParent`, `onChildren`, `onChild`, `onChildAt`, `onSibling(s)`, `onAncestors`, `onFirst`, `onLast`, `filter`, `filterToOne` |
| `ui-test-cm/SemanticsMatcher.kt` | `class SemanticsMatcher`, `expectValue`/`keyIsDefined`/`keyNotDefined` factories, `infix and`, `infix or`, `operator not` |
| `ui-test-cm/Filters.kt` | Every prebuilt matcher: `isEnabled`, `isOn`, `hasText`, `hasContentDescription`, `hasTestTag`, `hasClickAction`, `hasScrollAction`, `isRoot`, `isDialog`, `isPopup`, `hasParent`, `hasAnyAncestor`, `hasAnyChild`, `hasAnySibling`, `hasAnyDescendant`, `hasImeAction`, `hasScrollToIndexAction`, `hasScrollToKeyAction`, `hasScrollToNodeAction` |
| `ui-test-cm/Assertions.kt` | Every `assert*` extension: `assertIsDisplayed`, `assertIsNotDisplayed`, `assertIsEnabled`, `assertIsOn`/`Off`, `assertIsSelected`, `assertHasClickAction`, `assertTextEquals`, `assertTextContains`, `assertContentDescriptionEquals`, `assertCountEquals`, `assertAll`, `assertAny`, `assert(matcher)` |
| `ui-test-cm/BoundsAssertions.kt` | `assertWidthIsEqualTo`, `assertHeightIsAtLeast`, `assertPositionInRootIsEqualTo`, `assertLeftPositionInRootIsEqualTo`, `getUnclippedBoundsInRoot`, `getBoundsInRoot`, `getAlignmentLinePosition`, `Dp.assertIsEqualTo` (default tolerance ½ dp) |
| `ui-test-cm/Actions.kt` | `performClick`, `performScrollTo`, `performScrollToIndex`/`Key`/`Node`, `performTouchInput`, `performMouseInput`, `performKeyInput`, `performMultiModalInput`, `performTrackpadInput`, `performRotaryScrollInput`, `requestFocus`, `performSemanticsAction`, `performCustomAccessibilityActionWithLabel*` (experimental), `performFirstLinkClick`, `performIndirectPointerInput`, `tryPerformAccessibilityChecks`. `performGesture` is `@Deprecated` |
| `ui-test-cm/TextActions.kt` | `performTextInput`, `performTextReplacement`, `performTextClearance`, `performTextInputSelection`, `performImeAction` |
| `ui-test-cm/KeyInputHelpers.kt` | `performKeyPress(KeyEvent): Boolean` (low-level, sends to focused root) |
| `ui-test-cm/InjectionScope.kt` | `interface InjectionScope : Density`. Geometry helpers: `width`, `height`, `left/top/right/bottom`, `centerX/Y`, `topLeft/topCenter/topRight/centerLeft/center/centerRight/bottomLeft/bottomCenter/bottomRight`, `percentOffset`, `eventPeriodMillis`, `advanceEventTime`, `viewConfiguration`, `visibleSize` |
| `ui-test-cm/TouchInjectionScope.kt` | Touch DSL: `down`/`up`/`moveBy`/`moveTo`/`cancel`. Extensions: `click`, `longClick`, `doubleClick`, `swipe`, `swipeUp`/`Down`/`Left`/`Right`, `pinch`, `swipeWithVelocity`, `multiTouchSwipe` |
| `ui-test-cm/MouseInjectionScope.kt` | `press`/`release`/`scroll` + extensions `click`, `rightClick`, `doubleClick`, `tripleClick`, `longClick`, `animateMoveTo`/`By`/`Along`, `dragAndDrop`, `smoothScroll` |
| `ui-test-cm/KeyInjectionScope.kt` | `keyDown`/`keyUp`/`isKeyDown`, modifier-state vals (`isCtrlDown`/`isShiftDown`/etc.), helpers: `pressKey`, `withKeyDown`, `withKeysDown`, `withKeyToggled`, `withKeysToggled` |
| `ui-test-cm/RotaryInjectionScope.kt` | `rotateToScrollVertically`, `rotateToScrollHorizontally` (Wear OS) |
| `ui-test-cm/MultiModalInjectionScope.kt` | Aggregator: `touch`, `mouse`, `key`, `rotary`, `trackpad`, `indirectPointer` |
| `ui-test-cm/Mouse.kt` | `value class ScrollWheel { Horizontal, Vertical }`, `expect value class MouseButton { Primary, Secondary, Tertiary }` |
| `ui-test-cm/Output.kt` | `printToLog(tag, maxDepth)`, `printToString(maxDepth)` for both single and collection. Default depth is `Int.MAX_VALUE` (single) / `0` (collection) |
| `ui-test-cm/MainTestClock.kt` | Public clock interface, `ComposeTimeoutException` |
| `ui-test-cm/AbstractMainTestClock.kt` | Abstract impl wrapping `TestCoroutineScheduler` |
| `ui-test-cm/ComposeUiTest.kt` | `expect sealed interface ComposeUiTest`, free `waitUntilNodeCount`/`AtLeastOneExists`/`ExactlyOneExists`/`DoesNotExist` extensions, `runComposeUiTest` (expect) |
| `ui-test/src/commonMain/kotlin/androidx/compose/ui/test/v2/ComposeUiTest.kt` | v2 `runComposeUiTest` (StandardTestDispatcher default) |
| `ui-test-cm/ExperimentalTestApi.kt` | `@ExperimentalTestApi`, `@InternalTestApi` |
| `ui-test/src/jvmAndAndroidMain/kotlin/androidx/compose/ui/test/IdlingResource.kt` | `interface IdlingResource { val isIdleNow: Boolean; fun getDiagnosticMessageIfBusy(): String? = null }` |

### `ui-test` (androidMain — Android-specific actuals)

| File | Owns |
|---|---|
| `ui-test/src/androidMain/kotlin/androidx/compose/ui/test/MainTestClockImpl.android.kt` | Android clock impl |
| `ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeUiTest.android.kt` | `runComposeUiTest`, `runAndroidComposeUiTest`, `runEmptyComposeUiTest`, `AndroidComposeUiTestEnvironment`, `sealed interface AndroidComposeUiTest<A> : ComposeUiTest` (adds `val activity: A?`) |
| `ui-test/src/androidMain/kotlin/androidx/compose/ui/test/v2/ComposeUiTest.android.kt` | v2 actuals |
| `ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeIdlingResource.android.kt` | Aggregates recomposer + snapshot + frame-clock awaiters into one `IdlingResource`. Caps internal advance loop at **100 frames per call** |
| `ui-test/src/androidMain/kotlin/androidx/compose/ui/test/IdlingStrategy.android.kt`, `EspressoLink.android.kt`, `RobolectricIdlingStrategy.android.kt` | Strategy split: Espresso for instrumentation, Robolectric for host. `EspressoLink` is the `androidx.test.espresso.IdlingResource` bridge |
| `ui-test/src/androidMain/kotlin/androidx/compose/ui/test/AndroidSynchronization.android.kt` | `runOnUiThread` posts a `FutureTask` via `Instrumentation.runOnMainSync`; if already on UI thread, runs in-place |

### `ui-test-junit4`

| File | Owns |
|---|---|
| `ui-test-junit4/src/jvmAndAndroidMain/kotlin/androidx/compose/ui/test/junit4/ComposeTestRule.jvmAndAndroid.kt` | `interface ComposeTestRule : TestRule, SemanticsNodeInteractionsProvider`, `interface ComposeContentTestRule : ComposeTestRule { fun setContent(...) }`, `expect createComposeRule` |
| `ui-test-junit4/src/androidMain/kotlin/androidx/compose/ui/test/junit4/AndroidComposeTestRule.android.kt` | `class AndroidComposeTestRule<R: TestRule, A: ComponentActivity>`, `actual createComposeRule()`, `createAndroidComposeRule<A>()` (reified + non-reified), `createEmptyComposeRule()` |
| `ui-test-junit4/src/androidMain/kotlin/androidx/compose/ui/test/junit4/StateRestorationTester.android.kt` | `class StateRestorationTester(rule: ComposeContentTestRule)`, `setContent`, `emulateSavedInstanceStateRestore()` |
| `ui-test-junit4/src/jvmAndAndroidMain/kotlin/androidx/compose/ui/test/junit4/v2/ComposeTestRule.jvmAndAndroid.kt` | v2 `expect createComposeRule` |
| `ui-test-junit4/src/androidMain/kotlin/androidx/compose/ui/test/junit4/v2/AndroidComposeTestRule.android.kt` | v2 `createComposeRule` / `createAndroidComposeRule` / `createEmptyComposeRule` (StandardTestDispatcher default) |

### `ui-test-manifest`

| File | Owns |
|---|---|
| `ui-test-manifest/src/main/AndroidManifest.xml` | The whole artifact: declares `<activity android:theme="@android:style/Theme.Material.Light.NoActionBar" android:name="androidx.activity.ComponentActivity" android:exported="true" />` |
| `ui-test-manifest/build.gradle` | `api("androidx.activity:activity:1.2.1")`, `lintPublish(:compose:ui:ui-test-manifest-lint)` |

### `ui-test-manifest-lint`

| File | Owns |
|---|---|
| `ui-test-manifest-lint/src/main/java/androidx/compose/ui/test/manifest/lint/GradleDebugConfigurationDetector.kt` | The `TestManifestGradleConfiguration` lint check: WARNING when `androidx.compose.ui:ui-test-manifest` is added on any config other than `debugImplementation`. Quick fix replaces it with `debugImplementation` |

### `ui-test-accessibility` / `ui-test-junit4-accessibility`

| File | Owns |
|---|---|
| `ui-test-accessibility/src/androidMain/kotlin/androidx/compose/ui/test/accessibility/ComposeUiTestExt.android.kt` | `@RequiresApi(34) @ExperimentalTestApi fun ComposeUiTest.enableAccessibilityChecks(validator)`, `disableAccessibilityChecks` |
| `ui-test-junit4-accessibility/src/androidMain/kotlin/androidx/compose/ui/test/junit4/accessibility/AndroidComposeTestRuleExt.android.kt` | Same on `AndroidComposeTestRule<R, A>` (NOT experimental — stable) |

### Real test patterns to mine when writing skills

| File | Why it's a canonical example |
|---|---|
| `compose/material3/material3/src/androidDeviceTest/.../SwitchTest.kt` | Class skeleton, hoisted state, position assertions |
| `compose/foundation/foundation/integration-tests/lazy-tests/.../LazyListTest.kt` | LazyList scrolling, `scrollMainAxisBy`, per-item tags |
| `compose/foundation/foundation/integration-tests/lazy-tests/.../LazyListItemPlacementAnimationTest.kt` | `mainClock.autoAdvance = false` + `onAnimationFrame` helper |
| `compose/animation/animation/src/androidDeviceTest/.../CrossfadeTest.kt` | `advanceTimeBy` + `advanceTimeUntil`, `runOnUiThread { state = … }` |
| `compose/material3/material3/src/androidDeviceTest/.../ButtonTest.kt` | `getUnclippedBoundsInRoot` math for padding |
| `compose/foundation/foundation/src/androidDeviceTest/.../DraggableTest.kt` | High-level `swipe(start, end, durationMillis)` |
| `compose/foundation/foundation/src/androidDeviceTest/.../ScrollableAreaTest.kt` | Low-level `down(); moveBy()` |
| `compose/foundation/foundation/src/androidDeviceTest/.../CombinedClickableTest.kt` | `longClick`, mouse hover via `enter`/`exit` |
| `compose/foundation/foundation/src/androidDeviceTest/.../TransformableTest.kt` | `pinch(...)` four-point gesture |
| `compose/foundation/foundation/src/androidDeviceTest/.../BasicTextFieldTest.kt` | `performTextInput` + `assertTextEquals`, `performTextReplacement` |
| `compose/foundation/foundation/src/androidDeviceTest/.../DefaultKeyboardActionsTest.kt` | `performImeAction` + IME-action focus chain |
| `compose/foundation/foundation/src/androidDeviceTest/.../TextFieldFocusCustomDialogTest.kt` | Compose ↔ Espresso interop: `Espresso.onView(...).perform(ViewActions.click())` after `rule.setContent` |
| `compose/material3/material3/src/androidDeviceTest/.../SnackbarHostTest.kt` | `rule.waitUntil { job.isCompleted }` over a coroutine |
| `compose/material3/material3/src/androidDeviceTest/.../ExposedDropdownMenuTest.kt` | `rule.waitUntil { matcher.matches(node) }` |
| `compose/material3/material3/src/androidDeviceTest/.../ToggleButtonScreenshotTest.kt` | The ONE place `Thread.sleep` is correct: waiting on RenderThread for ripples |
| `compose/ui/ui-test/src/androidHostTest/.../RobolectricComposeTest.kt` | Host (Robolectric) entry-point, `@Config(minSdk = RobolectricMinSdk = 23)` |

---

## C. Concept cheatsheet

### Source set conventions (KMP)

```
commonTest
└── androidCommonTest          (helpers shared by host + device)
    ├── androidDeviceTest      (instrumentation APK on emulator/device)
    └── androidHostTest        (Robolectric on JVM)
```

- `androidDeviceTest` requires its own `AndroidManifest.xml` for any non-`ComponentActivity` test activity.
- `androidHostTest` uses `org.robolectric:robolectric` + `@Config(minSdk = …)`. `enableRobolectric()` is an internal androidx Gradle helper; consumers add Robolectric themselves.
- Same `runComposeUiTest { setContent { … } }` block runs unchanged in both — only the underlying Looper/Choreographer differs.

### Frame model

- `DefaultFrameDelay = 16_000_000L` ns (16 ms) — `TestMonotonicFrameClock.jvmAndAndroid.kt:33`.
- `MainTestClock` and `Recomposer` share one `kotlinx.coroutines.test.TestCoroutineScheduler`.
- `advanceTimeBy(ms)` rounds **up** to nearest multiple of 16 ms unless `ignoreFrameDuration = true`.
- Per-frame order inside the recomposer: animations awaiting `withFrameNanos` resume **before** recomposition runs (`MainTestClock.kt:49-60` KDoc).

### `autoAdvance` truth table

| Mode | What advances time? | When to use |
|---|---|---|
| `true` (default) | Framework auto-advances during `waitForIdle`/`waitUntil` (up to 100 frames per call inside `ComposeIdlingResource.isIdleNow`). | Normal "drive UI to final state" tests. |
| `false` | Only explicit `mainClock.advanceTimeBy*` / `advanceTimeUntil` advance. `waitForIdle` does NOT advance. Pending recomposition / animations are NOT treated as non-idle. | Animation tests, frame-by-frame snapshots, infinite animations (which would otherwise be cancelled by `InfiniteAnimationPolicy`). |

### `runOnIdle` vs `runOnUiThread` vs `runWhenIdle`

- `runOnUiThread { … }` — posts to UI thread, blocks until done. Does NOT wait for idle.
- `runOnIdle { … }` = `waitForIdle()` then `runOnUiThread`. Default for state mutations.
- `runWhenIdle { … }` — same as `runOnIdle` but suppresses the implicit `waitForIdle` triggered by node queries inside the block. Faster for assert-only blocks; **MUST NOT** mutate state inside.
- `awaitAndRunWhenIdle { … }` — suspending variant.

### Timeout sources (do not mix)

| API | Timeout source |
|---|---|
| `waitForIdle` / `awaitIdle` | Wall clock — Espresso `IdlingPolicies.getMasterIdlingPolicy()` (override with `IdlingPolicies.setMasterPolicyTimeout`) |
| `waitUntil(...)` family | Wall clock — per-call `timeoutMillis = 1_000L` default |
| `MainTestClock.advanceTimeUntil(...)` | **Test clock** — per-call `timeoutMillis = 1_000` default |
| `runComposeUiTest(testTimeout = …)` | Wall clock — default `60.seconds`. Throws `AndroidComposeUiTestTimeoutException` |

### Default merged tree behavior

- `useUnmergedTree` defaults to `false` everywhere.
- Merged tree collapses descendants whose semantics roll up (e.g. `Text` and `Icon` inside a `Button` show as one node).
- The flag is sticky — `onChild`/`onParent`/`filter` propagate it (`SemanticsNodeInteraction.kt:42-49`).
- Error messages on count mismatch automatically run the same query against the unmerged tree and embed the unmerged matches as a hint to flip the flag.

### v1 vs v2 test entry points

- v1 (`androidx.compose.ui.test.junit4.createComposeRule`, `androidx.compose.ui.test.runComposeUiTest`) — `UnconfinedTestDispatcher` for composition. Deprecated `WARNING`. Eager dispatch.
- v2 (`androidx.compose.ui.test.junit4.v2.createComposeRule`, `androidx.compose.ui.test.v2.runComposeUiTest`) — `StandardTestDispatcher` for composition. Recommended. Matches `kotlinx.coroutines.test.runTest` semantics. Tests may need explicit `mainClock.runCurrent()` / `advanceTimeBy(0)` after migrating.

### Common gotchas

1. **`createComposeRule()` requires `ui-test-manifest`** as `debugImplementation`. Without it: `ActivityNotFoundException` for `androidx.activity.ComponentActivity`. Lint flag `TestManifestGradleConfiguration` warns when this dep is on the wrong config.
2. **Custom Activity needs its own manifest entry** in `src/androidTest/AndroidManifest.xml` (or `src/debug/AndroidManifest.xml`).
3. **`createEmptyComposeRule()` returns `ComposeTestRule`, NOT `ComposeContentTestRule`** — no `setContent` exposed; you must launch your own scenario.
4. **`StateRestorationTester.emulateSavedInstanceStateRestore()` does NOT exercise activity lifecycle** — only restores `SaveableStateRegistry`. Plain `remember` state is lost. Bundle size cap: 1 MB.
5. **Cannot mix `runComposeUiTest { }` and `ComposeTestRule`** in the same test — both manage independent test environments.
6. **`runOnUiThread` from the UI thread does not deadlock** (runs in-place via `isOnUiThread()` check), but `Espresso.runUntilIdle` from the UI thread DOES throw with a clear message.
7. **`Thread.sleep` desyncs from `MainTestClock`** — only legitimate use is waiting for the RenderThread (ripples, screenshot tests) where Compose has no idle observation.
8. **Indefinite animations + `autoAdvance = true`** — `InfiniteAnimationPolicy.onInfiniteOperation` throws `CancellationException` to prevent hangs. Set `autoAdvance = false` first.
9. **Accessibility checks require API 34 + are no-op on Robolectric** — both `enableAccessibilityChecks` extensions check `Build.FINGERPRINT == "robolectric"` and warn.
10. **`performGesture`** is `@Deprecated` — always `performTouchInput`.

---

## D. Editorial directives (mirror in skills)

Five hot takes (see SPEC.md §5 for full text):

1. Tag from production, find by tag from tests.
2. Default to `useUnmergedTree = false`.
3. Animation tests require `mainClock.autoAdvance = false`.
4. `waitUntil` is wall clock; `advanceTimeUntil` is test clock — do not mix.
5. Funnel state mutations through `runOnIdle` / `runOnUiThread`.

Plus two recurring directives:

6. Prefer the v2 entry points (`StandardTestDispatcher` default).
7. `Thread.sleep` is a smell except in screenshot/RenderThread waits.

---

## E. Vocabulary the user will say (fold into `description`/`keywords`)

Symptoms (write skills that match these):

- "test is flaky / sometimes passes" → `synchronizing-with-idle`, `testing-animations-deterministically`
- "animation never finishes / test times out" → `testing-animations-deterministically`
- "no node matched / multiple nodes matched" → `printing-the-semantics-tree`, `finding-nodes-by-tag-text-content`
- "ActivityNotFoundException" / "ComponentActivity not found" → `configuring-test-dependencies`
- "Cannot find test rule, createComposeRule unresolved" → `configuring-test-dependencies`
- "test passes locally but fails on CI" → `synchronizing-with-idle`, `setting-up-host-vs-device-tests`
- "InfiniteAnimationPolicy CancellationException" → `testing-animations-deterministically`
- "rememberSaveable state lost on rotation" → `testing-state-restoration`
- "Espresso onView and Compose at the same time" → `testing-with-espresso-interop`
- "Thread.sleep in test" → `synchronizing-with-idle`
- "merged tree vs unmerged tree" → `finding-nodes-by-tag-text-content`
- "performGesture deprecated" → `injecting-touch-gestures`
- "lint warning ui-test-manifest" → `configuring-test-dependencies`
- "Robolectric vs instrumentation" / "host test" → `setting-up-host-vs-device-tests`
- "accessibility checks throw on Robolectric" → `enabling-accessibility-checks`

APIs (skill keyword sets):

- `createComposeRule`, `createAndroidComposeRule`, `createEmptyComposeRule`, `runComposeUiTest`, `runAndroidComposeUiTest`, `ComposeTestRule`, `ComposeUiTest`, `AndroidComposeTestRule`, `StateRestorationTester`
- `onNodeWithTag`, `onNodeWithText`, `onNodeWithContentDescription`, `onAllNodes`, `onRoot`, `useUnmergedTree`
- `SemanticsMatcher`, `hasText`, `hasClickAction`, `hasScrollToKeyAction`, `hasImeAction`, `isFocused`, `isEnabled`, `isOn`, `isPopup`, `isDialog`
- `assertExists`, `assertIsDisplayed`, `assertIsEnabled`, `assertIsOn`, `assertTextEquals`, `assertCountEquals`, `assertWidthIsEqualTo`, `getUnclippedBoundsInRoot`
- `performClick`, `performTouchInput`, `performMouseInput`, `performKeyInput`, `performMultiModalInput`, `performScrollToIndex`, `performScrollToKey`, `performScrollToNode`, `performTextInput`, `performTextReplacement`, `performImeAction`
- `MainTestClock`, `mainClock.autoAdvance`, `advanceTimeByFrame`, `advanceTimeBy`, `advanceTimeUntil`, `waitForIdle`, `waitUntil`, `waitUntilNodeCount`, `runOnIdle`, `runOnUiThread`, `runWhenIdle`
- `IdlingResource`, `IdlingResourceRegistry`, `ComposeIdlingResource`, `EspressoLink`, `IdlingPolicies.setMasterPolicyTimeout`
- `enableAccessibilityChecks`, `AccessibilityValidator`, `tryPerformAccessibilityChecks`
- `printToLog`, `printToString`, `fetchSemanticsNode`, `Modifier.testTag`

---

## F. Fundamentals — primary sources

Authoritative source: developer.android.com/training/testing/fundamentals + `tasks/research/R8-android-fundamentals.md`.

### F.1 External URLs

- https://developer.android.com/training/testing/fundamentals
- https://developer.android.com/training/testing/fundamentals/what-to-test
- https://developer.android.com/training/testing/fundamentals/test-doubles
- https://developer.android.com/training/testing/fundamentals/strategies
- https://developer.android.com/training/testing/local-tests
- https://developer.android.com/training/testing/instrumented-tests
- https://developer.android.com/training/testing/instrumented-tests/stability — determinism guidance (separate page)

### F.2 Test scope vocabulary (from /fundamentals)

Three sizes per Google: **small** (no Android stubs), **medium** (Android framework via Robolectric or in-process), **big** (instrumented). Plus an alternative 5-layer framing on `/strategies`: Unit / Component / Feature / Application / Release Candidate. Skill authors should pick ONE framing per skill and stick with it.

### F.3 Test-double terminology (verbatim definitions, /test-doubles)

- **Fake** — a working implementation with shortcuts (e.g. `FakeUserRepository` with in-memory map). **Google explicitly prefers fakes** over other doubles.
- **Mock** — verifies interactions (calls, args, order).
- **Stub** — returns canned answers, no behavior.
- **Spy** — wraps a real object to record calls.
- **Dummy** — placeholder, never used.
- **Shadow** — Robolectric-specific; replaces an Android framework class with a JVM impl.

Authoritative quotes: "fakes ... are preferred", "Fakes are preferred over stubs for simplicity", "Fakes or mocks are therefore preferred over spies" — all from /test-doubles. Cite verbatim in skills.

### F.4 What to test (from /what-to-test)

Categories:
1. **State on screen** — what's user-observable now.
2. **State held in memory** — `ViewModel` state, in-memory caches.
3. **Persisted state** — DB, DataStore, SharedPreferences, files.
4. **Other state** — system bars, system services, alarms.
5. **Errors and edge cases** — empty lists, network failures, malformed input.

What NOT to test: implementation details, framework internals, third-party library guarantees.

### F.5 Source sets

- `src/test/` — JVM unit tests. `testImplementation` deps. Run with `./gradlew test`.
- `src/androidTest/` — instrumented tests. `androidTestImplementation` deps. Run with `./gradlew connectedAndroidTest`.
- `src/sharedTest/` — community/Codelab convention (NOT Google-documented as such). Same source compiled into both `test` and `androidTest`. Broken on AGP 7.2+ in many projects; Compose internally moved to KMP `androidHostTest` / `androidDeviceTest`.

### F.6 Hilt for testing (from /strategies)

`@HiltAndroidTest`, `HiltAndroidRule`, `@TestInstallIn`, `@UninstallModules`, `@BindValue`, `@CustomTestApplication`. Rule ordering: HiltAndroidRule must be evaluated FIRST (`@Rule(order = 0)`).

### F.7 Honest gaps / cautions (per R8 report)

- The "70/20/10 ratio" famous in older Google guidance is **not** on the current /fundamentals or /strategies pages. Cite *Software Engineering at Google* if needed; do NOT attribute to developer.android.com.
- "Hermetic" is not formalized on /strategies — only mentioned in passing.
- Given-When-Then / AAA are never NAMED on the pages but are visibly used in code samples. Cite the samples.
- Determinism / seed guidance lives at /instrumented-tests/stability, not /strategies.
- The FakeUserRepository sample on /test-doubles has a typo (`val const UserAlice` is invalid Kotlin) — do not blindly reproduce.

---

## G. JVM tests — primary sources

### G.1 Library coordinates (released versions as of 2026-05-06)

```
testImplementation("junit:junit:4.13.2")
testImplementation("androidx.test:core:1.7.0")
testImplementation("androidx.test:runner:1.7.0")          // primarily androidTestImpl, but referenced from JVM
testImplementation("androidx.test:rules:1.7.0")
testImplementation("androidx.test.ext:junit:1.3.0")
testImplementation("androidx.test.ext:junit-ktx:1.3.0")
testImplementation("androidx.test.ext:truth:1.7.0")

// Mockito
testImplementation("org.mockito:mockito-core:5.x")          // 5+ uses inline by default
testImplementation("org.mockito.kotlin:mockito-kotlin:5.x") // Kotlin DSL

// MockK
testImplementation("io.mockk:mockk-jvm:1.14.x")
androidTestImplementation("io.mockk:mockk-android:1.14.x")
androidTestImplementation("io.mockk:mockk-agent:1.14.x")

// Coroutines test
testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.10.x")

// Turbine
testImplementation("app.cash.turbine:turbine:1.x")

// Robolectric
testImplementation("org.robolectric:robolectric:4.x")
```

### G.2 Critical deprecations

- `androidx.test.runner.AndroidJUnit4` is **`@Deprecated`** — use `androidx.test.ext.junit.runners.AndroidJUnit4`.
- `androidx.test.InstrumentationRegistry` (no `.platform.app.`) is **`@Deprecated`** — use `androidx.test.platform.app.InstrumentationRegistry`.
- `runBlockingTest` is deprecated with error since coroutines 1.7+ — use `runTest { }`.
- `ActivityTestRule` is deprecated — use `ActivityScenarioRule<A>`.

### G.3 androidx as evidence

Real usage signal from this checkout:
- **Mockito-only** — 400+ Kotlin files import `org.mockito`, **zero** import `io.mockk`. Compose-foundation, compose-material, room3-runtime ship `mockito-extensions/org.mockito.plugins.MockMaker` files containing `mock-maker-inline`.
- Canonical pattern: `mockito-kotlin` DSL (`mock<T>()`, `whenever`, `argumentCaptor<T>()` with `firstValue`/`lastValue`).
- Coroutines test: `MainDispatcherRule` JUnit4 wrapper at `testutils/testutils-ktx/src/jvmMain/kotlin/androidx/testutils/MainDispatcherRule.jvm.kt`.

### G.4 runTest signature

```kotlin
fun runTest(
    context: CoroutineContext = EmptyCoroutineContext,
    timeout: Duration = 60.seconds,
    testBody: suspend TestScope.() -> Unit
): TestResult
```

The older `dispatchTimeoutMs` overload is deprecated with error. `TestResult` is `Unit` on JVM/Native, `Promise<Unit>` on JS — KMP tests must use `fun foo() = runTest { … }` single-expression form.

### G.5 TestDispatcher truth table

| Type | Behavior | When |
|---|---|---|
| `StandardTestDispatcher` | queues continuations; explicit advance | default; matches `runTest` semantics |
| `UnconfinedTestDispatcher` | eager dispatch | tests where you don't want to manage time |

### G.6 Robolectric runner choice

- **`AndroidJUnit4`** (from `androidx.test.ext.junit.runners`) is preferred — dispatches to Robolectric on JVM and to instrumentation on device. Enables sharedTest pattern.
- **`RobolectricTestRunner`** only needed for Robolectric-specific subclasses (`ParameterizedRobolectricTestRunner`) or APIs that require it.
- `@Config(sdk = [Build.VERSION_CODES.UPSIDE_DOWN_CAKE])` for single SDK; matrix `@Config(sdk = [21, 28, 33])` for multi-SDK.
- `testOptions.unitTests.includeAndroidResources = true` is non-negotiable for any Robolectric test that touches `R.*`.

---

## H. Instrumentation — primary sources

### H.1 Library coordinates

```
androidTestImplementation("androidx.test:core:1.7.0")
androidTestImplementation("androidx.test:runner:1.7.0")
androidTestImplementation("androidx.test:rules:1.7.0")
androidTestImplementation("androidx.test.ext:junit:1.3.0")
androidTestImplementation("androidx.fragment:fragment-testing:1.8.x")
debugImplementation("androidx.fragment:fragment-testing-manifest:1.8.x")  // Android M+ manifest

// Espresso 3.7.0
androidTestImplementation("androidx.test.espresso:espresso-core:3.7.0")
androidTestImplementation("androidx.test.espresso:espresso-contrib:3.7.0")
androidTestImplementation("androidx.test.espresso:espresso-intents:3.7.0")
androidTestImplementation("androidx.test.espresso:espresso-web:3.7.0")
androidTestImplementation("androidx.test.espresso:espresso-accessibility:3.7.0")
implementation("androidx.test.espresso:espresso-idling-resource:3.7.0")  // for production code

// UiAutomator
androidTestImplementation("androidx.test.uiautomator:uiautomator:2.3.0")

// Test Orchestrator (note: androidTestUtil, not androidTestImplementation)
androidTestUtil("androidx.test:orchestrator:1.6.1")

testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
```

### H.2 Test runner Gradle config

```kotlin
android {
    defaultConfig {
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        // Optional: pass args
        testInstrumentationRunnerArguments["clearPackageData"] = "true"
    }
    testOptions {
        execution = "ANDROIDX_TEST_ORCHESTRATOR"   // for orchestrator
        animationsDisabled = true                  // hermetic
    }
}
```

### H.3 ActivityScenario API surface

```kotlin
ActivityScenario.launch<MyActivity>(): ActivityScenario<MyActivity>
ActivityScenario.launch(intent: Intent): ActivityScenario<*>
ActivityScenario.launchActivityForResult<MyActivity>(): ActivityScenario<MyActivity>

scenario.moveToState(Lifecycle.State.STARTED)    // CREATED, STARTED, RESUMED, DESTROYED
scenario.recreate()
scenario.onActivity { activity -> /* runs on UI thread */ }
scenario.state                                    // current Lifecycle.State
scenario.result                                   // Instrumentation.ActivityResult (only after finish)
scenario.close()                                  // AutoCloseable; idempotent

// JUnit4 wrapper (the SINGLE canonical location):
@get:Rule val activityScenarioRule = ActivityScenarioRule(MyActivity::class.java)
// FQN: androidx.test.ext.junit.rules.ActivityScenarioRule
// NOT: androidx.test.rule.ActivityScenarioRule (does not exist)
```

### H.4 FragmentScenario API surface

```kotlin
launchFragmentInContainer<MyFragment>(
    fragmentArgs: Bundle? = null,
    themeResId: Int = R.style.FragmentScenarioEmptyFragmentActivityTheme,
    factory: FragmentFactory? = null
): FragmentScenario<MyFragment>

launchFragment<HeadlessFragment>(...)              // no container; for headless

scenario.moveToState(Lifecycle.State.STARTED)
scenario.recreate()
scenario.onFragment { fragment -> ... }
```

The default theme `FragmentScenarioEmptyFragmentActivityTheme` extends `android:Theme.WithActionBar` (NOT AppCompat). Override with `themeResId` for AppCompat fragments.

### H.5 Espresso essentials

```kotlin
// Find
onView(withId(R.id.button))
onData(allOf(...))                                 // for AdapterView

// Match
onView(allOf(withText("Submit"), isEnabled()))

// Act
onView(...).perform(click())
onView(...).perform(typeText("hello"), closeSoftKeyboard())

// Assert
onView(...).check(matches(isDisplayed()))
onView(...).check(doesNotExist())
onView(...).check(matches(withText("Submit")))

// Idle
IdlingRegistry.getInstance().register(myIdlingResource)
IdlingPolicies.setMasterPolicyTimeout(60, TimeUnit.SECONDS)
```

`RootMatchers.DEFAULT` excludes dialogs, popups, toasts. Always use `.inRoot(isPlatformPopup())` for popup menus, `.inRoot(isDialog())` for `AlertDialog`.

### H.6 UiAutomator essentials

```kotlin
val device = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation())

device.pressHome()
device.findObject(By.text("Submit")).click()             // BySelector (modern)
device.findObject(By.res("com.example", "submit_btn"))   // by package + res id

device.wait(Until.hasObject(By.text("Loaded")), 5000)

// Cross-app: launch Settings, etc.
device.executeShellCommand("am start -n com.android.settings/.Settings")
```

`StaleObjectException` fires when a `UiObject2` reference becomes invalid after recompose/relayout. Re-find with `findObject(By...)`.

### H.7 Critical deprecations

- `androidx.test.runner.AndroidJUnit4` — deprecated; use `ext.junit.runners.AndroidJUnit4`.
- `ActivityTestRule` — deprecated; use `ActivityScenarioRule`.
- `IntentsTestRule` — deprecated; use `IntentsRule` (Espresso 3.5+) or call `Intents.init() / .release()` manually.
- `UiSelector` / `UiObject` — legacy; use `BySelector` / `UiObject2`.

---

## I. ADB — primary sources

Authoritative source: developer.android.com/tools/adb + developer.android.com/tools/logcat + `tasks/research/A1`, `A2`, `A3`.

### I.1 External URLs

- https://developer.android.com/tools/adb
- https://developer.android.com/tools/adb#wireless
- https://developer.android.com/tools/logcat
- https://developer.android.com/tools/releases/platform-tools
- https://developer.android.com/studio/test/command-line
- https://developer.android.com/studio/test/advanced-test-setup#use-gradle-managed-devices
- https://source.android.com/docs/setup/build/adb (AOSP)

### I.2 Three-piece architecture

- **Client** — the `adb` CLI binary at `$ANDROID_HOME/platform-tools/adb`.
- **Server** — background process on host, port **5037** by default.
- **Daemon (`adbd`)** — runs on the device; spawned by the system or by `init` on userdebug/eng builds.

`adb start-server`, `adb kill-server`, `adb reconnect [device|offline]`. Server logs at `$TMPDIR/adb.$UID.log` on macOS/Linux, `%TEMP%\adb.log` on Windows.

### I.3 Device states (truth set)

`device` (online), `offline`, `unauthorized`, `recovery`, `sideload`, `bootloader`, `rescue`, `connecting`, `no permissions` (Linux udev). The `[-state]` form requested in early plans does NOT exist; the canonical wait-for syntax is:

```
adb wait-for[-TRANSPORT]-<state>
TRANSPORT ∈ {usb, local, any}
state     ∈ {device, recovery, rescue, sideload, bootloader, disconnect}
```

### I.4 Wireless ADB (Android 11+)

```
adb pair <host:port>           # with pairing code from Settings → Developer options → Wireless debugging
adb connect <host:port>        # once paired
adb disconnect [<host:port>]
```

Legacy path (pre-Android 11): `adb tcpip <port>` after USB, then `adb connect`. mDNS service types: `_adb-tls-pairing._tcp`, `_adb-tls-connect._tcp`. ADB v34+ default mDNS backend is **Openscreen** (not Bonjour) on Linux/Windows; doc lags.

### I.5 `am instrument` invocation

```bash
adb shell am instrument -w -r \
  -e class com.example.MyTest#myMethod \
  -e package com.example \
  -e size small \
  -e numShards 4 -e shardIndex 0 \
  -e annotation com.example.SmokeTest \
  -e clearPackageData true \
  com.example.test/androidx.test.runner.AndroidJUnitRunner
```

- `-w` (wait) is **required** for exit codes to be meaningful.
- `-r` raw output.
- Output framing: `INSTRUMENTATION_STATUS` (per-test), `INSTRUMENTATION_STATUS_CODE` (1 start, 0 ok, -1 error, -2 failure, -3 ignored, -4 assumption-failure), `INSTRUMENTATION_RESULT` (final), `INSTRUMENTATION_CODE` (final).
- AndroidJUnitRunner adds `-3` IGNORED and `-4` ASSUMPTION_FAILURE on top of the framework's 4.
- For Test Orchestrator: target = `androidx.test.orchestrator/androidx.test.orchestrator.AndroidTestOrchestrator`, with `-e targetInstrumentation com.example.test/androidx.test.runner.AndroidJUnitRunner`.

### I.6 Hermetic test setup

```bash
adb shell settings put global window_animation_scale 0
adb shell settings put global transition_animation_scale 0
adb shell settings put global animator_duration_scale 0
adb shell pm clear <pkg>           # clean app state before each run
```

Or via Gradle: `testOptions.animationsDisabled = true`.

### I.7 logcat essentials

```bash
adb logcat -d                                      # dump and exit
adb logcat -c                                      # clear
adb logcat MyTag:D *:S                            # only MyTag at D+, silence others
adb logcat *:E                                     # errors and above
adb logcat -b crash                                # the crash buffer
adb logcat --pid $(adb shell pidof <pkg>)         # PID filter
adb logcat -v threadtime                           # default; -v json on Android 11+
adb logcat -t 100                                  # last 100 lines then exit
```

Buffers: `main` (default), `system`, `crash`, `events` (binary), `radio`, `kernel`, `all`.

### I.8 pull / push permission rules

| Path | Writable by adb shell? | Pull works? |
|---|---|---|
| `/data/local/tmp/` | ✓ | ✓ |
| `/sdcard/` (`/storage/emulated/0/`) | ✓ (within scoped storage rules on API 30+) | ✓ |
| `/data/data/<pkg>/` | only via `run-as <pkg>` (debuggable builds) | only via `run-as` |
| `/sdcard/Android/data/<pkg>/files/` | package-owned, scoped | ✓ for the owning package |

Binary-clean grab on debuggable build: `adb exec-out run-as <pkg> cat <path> > local-file`.

### I.9 Forward / reverse — argument-order trap

```
adb forward tcp:<host_port> tcp:<device_port>      # host → device. LOCAL REMOTE
adb reverse tcp:<device_port> tcp:<host_port>      # device → host. REMOTE LOCAL
```

The two have **opposite** argument order. Most common scripting bug.

### I.10 Critical findings (from research reports)

- `am instrument` exit codes only valid with `-w`. Without it, `$?` is meaningless.
- `force-stop` does NOT clear data — pair with `pm clear` for hermetic reset.
- `svc data`, `svc wifi`, `svc bluetooth` are **no-ops on API 30+**. Use `cmd wifi set-wifi-enabled`, `cmd connectivity airplane-mode` instead.
- `adb shell` exit-code propagation reliable only since API 24 / platform-tools 24.
- `adb -t <transport-id>` is transport ID, NOT a timeout flag. Wrap with shell `timeout` (or `gtimeout` on macOS) for timeouts.
- `adb logcat -f <file>` writes host-side via `adb logcat`, but device-side via `adb shell logcat -f`. Subtle.
- Test Orchestrator wires up via `androidTestUtil("androidx.test:orchestrator:...")`, NOT `androidTestImplementation`.
- Quoting through `adb shell` is double-shell (ssh-style) since Platform Tools 23 — load-bearing for `input text` and any string with spaces.
