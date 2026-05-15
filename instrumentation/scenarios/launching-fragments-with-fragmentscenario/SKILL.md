---
name: launching-fragments-with-fragmentscenario
description: Use this skill to test a `Fragment` in isolation using `FragmentScenario`, `launchFragmentInContainer<F>()`, and `launchFragment<F>()`. Covers the `androidx.fragment:fragment-testing` + `androidx.fragment:fragment-testing-manifest` artifact split, the `EmptyFragmentActivity` host (`android:Theme.WithActionBar` parent — not AppCompat), the `themeResId` override, the `FragmentFactory` survival across `recreate()` via the ViewModelStore-backed holder, the `containerViewId = android.R.id.content` vs `0` distinction (in-container vs headless), and the `findNavController()` limitation. Use when the user reports `You need to use a Theme.AppCompat theme (or descendant)`, `Fragment ... does not have a NavController set`, `Cannot set initial Lifecycle state to DESTROYED for FragmentScenario`, or asks "how do I test a fragment in isolation" / "headless fragment test".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-instrumentation-test
  - FragmentScenario
  - launchFragmentInContainer
  - launchFragment
  - EmptyFragmentActivity
  - fragment-testing-manifest
  - themeResId
  - FragmentFactory
  - AppCompat-theme
  - findNavController
  - debugImplementation
---

# Launching Fragments with FragmentScenario — Test Fragments in Isolation

`FragmentScenario` hosts a single fragment inside an internal `EmptyFragmentActivity` so the fragment's `onCreate` / `onViewCreated` / lifecycle can be exercised without a real screen. Two entry points exist: `launchFragmentInContainer<F>()` adds the fragment to `android.R.id.content` (full lifecycle including view), and `launchFragment<F>()` attaches it with no container (headless). The default theme extends `android:Theme.WithActionBar`, NOT AppCompat — overriding via `themeResId` is required for any AppCompat widget. This skill encodes the two artifacts, the theme trap, and the navigation limitation.

## When to use this skill

- The user wants to test a single Fragment without launching the full host Activity.
- A test crashes with `IllegalArgumentException: You need to use a Theme.AppCompat theme (or descendant) with the design library` and the fragment uses AppCompat / Material widgets.
- A test calls `findNavController()` and crashes with `Fragment ... does not have a NavController set`.
- The user wants to drive lifecycle (`STARTED` / `RESUMED` / `DESTROYED`) on a fragment for state-change testing.
- The user wants to inject a custom `FragmentFactory` for a fragment that doesn't have a no-arg constructor.
- The user mentions "headless fragment" or asks why `onViewCreated` doesn't fire.

## When NOT to use this skill

- The host is an Activity, not a Fragment — see `../launching-activities-with-activityscenario/SKILL.md`.
- The fragment requires a real navigation stack (`NavHostFragment`) — `FragmentScenario` does not provide one; use a custom test host activity that wires up `NavHostFragment` instead, or `TestNavHostController`.
- The runner / dependency stack is not yet set up — start with `../../runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.
- The fragment hosts Compose content and the test interacts via the Compose tree — see `../../../compose/setup/configuring-test-dependencies/SKILL.md`.

## Prerequisites

- `androidTestImplementation("androidx.fragment:fragment-testing:1.8.x")` — the API: `FragmentScenario`, `launchFragmentInContainer`, `launchFragment`, `withFragment`.
- `debugImplementation("androidx.fragment:fragment-testing-manifest:1.8.x")` — the manifest entry that declares `EmptyFragmentActivity`. Android M+ requires this artifact be on `debugImplementation` (or `testImplementation` for host tests) so the manifest merger picks up `EmptyFragmentActivity`.
- The runner stack from `../../runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.

## Workflow

- [ ] **1. Add both artifacts on the right configurations.** The two-artifact split is non-negotiable on Android M+:

```kotlin
dependencies {
    androidTestImplementation("androidx.fragment:fragment-testing:1.8.5")
    debugImplementation("androidx.fragment:fragment-testing-manifest:1.8.5")
}
```

`fragment-testing` ships the API; `fragment-testing-manifest` ships only the `<activity android:name="androidx.fragment.app.testing.EmptyFragmentActivity">` manifest entry plus the default theme. Without the manifest artifact, `launchFragmentInContainer` / `launchFragment` crash with `ActivityNotFoundException: EmptyFragmentActivity`. From `tasks/research/R2-scenario.md` lines 568-619.

- [ ] **2. Pick the launcher by view requirement.**

```kotlin
import androidx.fragment.app.testing.launchFragment
import androidx.fragment.app.testing.launchFragmentInContainer

// In-container — adds at android.R.id.content; full lifecycle including onCreateView/onViewCreated
val s1: FragmentScenario<MyFragment> = launchFragmentInContainer<MyFragment>()

// Headless — containerViewId = 0; onCreateView still runs but the view is not attached to a window
val s2: FragmentScenario<MyHeadlessFragment> = launchFragment<MyHeadlessFragment>()
```

`launchFragmentInContainer` uses `containerViewId = android.R.id.content` (`isViewAttachedToWindow == true`). `launchFragment` uses `containerViewId = 0` (headless; `isViewAttachedToWindow == false`). Source: R2 lines 263-389, 683-684.

- [ ] **3. Pass `fragmentArgs` for fragments that read `requireArguments()`.**

```kotlin
val args = bundleOf("user_id" to 42, "from_deep_link" to true)
val scenario = launchFragmentInContainer<UserDetailFragment>(fragmentArgs = args)
```

- [ ] **4. For AppCompat / Material fragments, override `themeResId` to an AppCompat theme.** The default theme `FragmentScenarioEmptyFragmentActivityTheme` extends `android:Theme.WithActionBar` (the platform Holo theme), NOT `Theme.AppCompat`. AppCompat widgets crash at inflation with `You need to use a Theme.AppCompat theme (or descendant) with the design library`. From R2 lines 466-501:

```kotlin
val scenario = launchFragmentInContainer<MyAppCompatFragment>(
    themeResId = R.style.Theme_MyApp,    // any AppCompat / Material theme
)
```

`EmptyFragmentActivity.setTheme` is called BEFORE `super.onCreate`, so the override applies before any fragment view inflation runs (R2 lines 490-501).

- [ ] **5. Inject a `FragmentFactory` for non-default-constructor fragments.** Both launchers accept a `factory: FragmentFactory?` parameter (R2 lines 263-323). The factory is stored in a `FragmentFactoryHolderViewModel` keyed off the `EmptyFragmentActivity`'s `ViewModelStore`, which means it survives `recreate()`:

```kotlin
class MyFragment(private val repo: UserRepository) : Fragment() { /* ... */ }

val factory = object : FragmentFactory() {
    override fun instantiate(classLoader: ClassLoader, className: String): Fragment {
        return when (loadFragmentClass(classLoader, className)) {
            MyFragment::class.java -> MyFragment(FakeUserRepository())
            else -> super.instantiate(classLoader, className)
        }
    }
}
val scenario = launchFragmentInContainer<MyFragment>(factory = factory)
```

Or use the no-factory inline overload that accepts a single-instance trailing lambda:

```kotlin
val scenario = launchFragmentInContainer { MyFragment(FakeUserRepository()) }
```

- [ ] **6. Drive lifecycle with `moveToState` and inspect with `onFragment`.** Same semantics as `ActivityScenario`:

```kotlin
import androidx.lifecycle.Lifecycle

scenario.moveToState(Lifecycle.State.STARTED)        // pause
scenario.onFragment { fragment ->                    // UI thread
    assertThat(fragment.someState).isEqualTo("ready")
}
scenario.recreate()                                  // configuration-change emulation
scenario.moveToState(Lifecycle.State.DESTROYED)      // terminal
```

`initialState = Lifecycle.State.DESTROYED` is **rejected** with `IllegalArgumentException: Cannot set initial Lifecycle state to DESTROYED for FragmentScenario` — the `require(...)` at `fragment-testing/src/main/.../FragmentScenario.kt:496-498` (R2 lines 699-718). Launch with `CREATED` and then `moveToState(DESTROYED)` instead.

- [ ] **7. Capture cross-thread state via `withFragment { }` instead of holder fields.** `withFragment` is the suspending/return-value variant of `onFragment` that propagates exceptions and return values cleanly (R2 lines 670-680):

```kotlin
val viewLifecycleOwner = scenario.withFragment { viewLifecycleOwner }
val isAdded = scenario.withFragment { isAdded }
```

- [ ] **8. Recognize the `findNavController()` limitation.** `EmptyFragmentActivity` does NOT install a `NavHostFragment`. `findNavController()` from a hosted fragment throws `IllegalStateException: Fragment ... does not have a NavController set` (R2 lines 852-874). Workarounds:

```kotlin
// Option A: TestNavHostController on the fragment view before onStart
launchFragmentInContainer {
    MyFragment().also { fragment ->
        fragment.viewLifecycleOwnerLiveData.observeForever { vlo ->
            if (vlo != null) {
                val testNav = TestNavHostController(ApplicationProvider.getApplicationContext())
                testNav.setGraph(R.navigation.my_graph)
                Navigation.setViewNavController(fragment.requireView(), testNav)
            }
        }
    }
}

// Option B: write a custom test host Activity that hosts NavHostFragment, declared in
// src/androidTest/AndroidManifest.xml, and use ActivityScenarioRule on it instead.
```

- [ ] **9. Close the scenario.** `FragmentScenario` is `Closeable`:

```kotlin
launchFragmentInContainer<MyFragment>().use { scenario ->
    scenario.onFragment { /* ... */ }
}
```

`close()` drives the Activity to `DESTROYED`. Calling `onFragment` after `DESTROYED` throws `IllegalStateException: The fragment has been removed from the FragmentManager already.` (FragmentScenario.kt:308-310, R2 lines 815-821).

## Patterns

### Pattern: WRONG vs RIGHT — AppCompat fragment with default theme

```kotlin
// WRONG
val scenario = launchFragmentInContainer<LoginFragment>()
// The default FragmentScenarioEmptyFragmentActivityTheme extends android:Theme.WithActionBar,
// which is the platform Holo theme. Inflating any AppCompat / Material widget crashes:
//   IllegalArgumentException: You need to use a Theme.AppCompat theme (or descendant) with the design library
```

```kotlin
// RIGHT
val scenario = launchFragmentInContainer<LoginFragment>(
    themeResId = R.style.Theme_MyApp,        // an AppCompat (or Material) theme
)
```

### Pattern: WRONG vs RIGHT — `fragment-testing-manifest` on the wrong configuration

```kotlin
// WRONG
dependencies {
    androidTestImplementation("androidx.fragment:fragment-testing:1.8.5")
    androidTestImplementation("androidx.fragment:fragment-testing-manifest:1.8.5")
}
// WRONG because: the manifest artifact must be on debugImplementation (or testImplementation
// for host tests) so the EmptyFragmentActivity declaration is merged into the test APK
// manifest. On androidTestImplementation it isn't merged; launchFragmentInContainer crashes
// with ActivityNotFoundException for EmptyFragmentActivity.
```

```kotlin
// RIGHT
dependencies {
    androidTestImplementation("androidx.fragment:fragment-testing:1.8.5")
    debugImplementation("androidx.fragment:fragment-testing-manifest:1.8.5")
}
```

### Pattern: WRONG vs RIGHT — `initialState = DESTROYED`

```kotlin
// WRONG
launchFragmentInContainer<MyFragment>(initialState = Lifecycle.State.DESTROYED)
// WRONG because: FragmentScenario.kt:496-498 rejects DESTROYED with
//   IllegalArgumentException: Cannot set initial Lifecycle state to DESTROYED for FragmentScenario
```

```kotlin
// RIGHT
val scenario = launchFragmentInContainer<MyFragment>(initialState = Lifecycle.State.CREATED)
scenario.moveToState(Lifecycle.State.DESTROYED)   // moveToState DOES allow DESTROYED (terminal)
```

### Pattern: WRONG vs RIGHT — `findNavController()` from a `FragmentScenario`-hosted fragment

```kotlin
// WRONG
launchFragmentInContainer<MyFragment>().onFragment { fragment ->
    fragment.findNavController().navigate(R.id.action_to_detail)
}
// WRONG because: EmptyFragmentActivity has no NavHostFragment; the lookup throws
//   IllegalStateException: Fragment ... does not have a NavController set
```

```kotlin
// RIGHT
launchFragmentInContainer {
    MyFragment().also { fragment ->
        fragment.viewLifecycleOwnerLiveData.observeForever { vlo ->
            if (vlo != null) {
                val nav = TestNavHostController(ApplicationProvider.getApplicationContext())
                nav.setGraph(R.navigation.my_graph)
                Navigation.setViewNavController(fragment.requireView(), nav)
            }
        }
    }
}
```

## Mandatory rules

- **MUST** put `androidx.fragment:fragment-testing-manifest` on `debugImplementation` (instrumentation) or `testImplementation` (host). On `androidTestImplementation` the manifest entry is not merged.
- **MUST** override `themeResId` for any fragment that uses AppCompat / Material widgets. The default theme is the platform Holo theme.
- **MUST NOT** call `findNavController()` from a fragment hosted by `EmptyFragmentActivity` without first setting a `TestNavHostController` on its view.
- **MUST NOT** pass `initialState = Lifecycle.State.DESTROYED` — it is rejected with `IllegalArgumentException` at FragmentScenario.kt:496-498.
- **MUST NOT** call `onFragment` / `withFragment` after `moveToState(DESTROYED)` or `close()` — throws `IllegalStateException`.
- **MUST NOT** call `launchFragmentInContainer` / `launchFragment` / `moveToState` from the main thread — they `await(...)` an instrumentation barrier and deadlock from the UI thread (same constraint as `ActivityScenario`; see `../launching-activities-with-activityscenario/SKILL.md`).
- **PREFERRED:** use the inline factory lambda `launchFragmentInContainer { MyFragment(FakeRepo()) }` over a hand-rolled `FragmentFactory` for the common single-fragment case.
- **PREFERRED:** wrap ad-hoc launches in `use { }` so `close()` runs on test failure too.

## Verification

- [ ] `grep -r "fragment-testing-manifest" build.gradle*` shows it on `debugImplementation` (or `testImplementation`), never `androidTestImplementation`.
- [ ] No test calls `launchFragmentInContainer<F>()` for an AppCompat/Material fragment without an explicit `themeResId =`.
- [ ] No test passes `initialState = Lifecycle.State.DESTROYED` to a launcher.
- [ ] Tests using `findNavController()` install a `TestNavHostController` before `moveToState(STARTED)`.
- [ ] `./gradlew :<module>:connectedDebugAndroidTest` runs without `ActivityNotFoundException: EmptyFragmentActivity` or `You need to use a Theme.AppCompat theme`.
- [ ] No leaked fragment reference outside `onFragment { }` / `withFragment { }` blocks.

## References

- Android Developers — Test your fragments (FragmentScenario): https://developer.android.com/guide/fragments/test
- Android Developers — Fragment testing reference: https://developer.android.com/reference/androidx/fragment/app/testing/FragmentScenario
- AndroidX Fragment release notes: https://developer.android.com/jetpack/androidx/releases/fragment
- `fragment/fragment-testing/src/main/java/androidx/fragment/app/testing/FragmentScenario.kt` — full implementation; lines 263-323 (`launchFragmentInContainer`), 325-389 (`launchFragment`), 393-462 (`moveToState` / `recreate` / `onFragment`), 496-498 (DESTROYED rejection), 504-508 (intent + theme extras).
- `fragment/fragment-testing-manifest/src/main/java/androidx/fragment/app/testing/EmptyFragmentActivity.kt` lines 28-55 — host activity that reads the theme out of the intent BEFORE `super.onCreate`.
- `fragment/fragment-testing-manifest/src/main/AndroidManifest.xml` lines 568-580 — `<activity android:name="androidx.fragment.app.testing.EmptyFragmentActivity" android:theme="@style/FragmentScenarioEmptyFragmentActivityTheme" android:exported="true" />`.
- `fragment/fragment-testing-manifest/src/main/res/values/styles.xml` line 469 — `<style name="FragmentScenarioEmptyFragmentActivityTheme" parent="android:Theme.WithActionBar">`.
- `tasks/research/R2-scenario.md` — full FragmentScenario report. Lines 247-389 (API surface), 466-501 (theme trap), 813-919 (12 documented pitfalls).
- `docs/CORPUS.md` Section H.4 — FragmentScenario API surface and theme trap summary.
