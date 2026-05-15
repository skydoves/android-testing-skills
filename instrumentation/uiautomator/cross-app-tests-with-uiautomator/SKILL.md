---
name: cross-app-tests-with-uiautomator
description: Use this skill to drive cross-app and system-UI flows from instrumentation tests using UiAutomator 2.3.0 — `UiDevice`, `BySelector` / `UiObject2` (modern), `UiSelector` / `UiObject` (legacy), `Until` conditions, and `Configurator` global timeouts. Covers the singleton acquisition (`UiDevice.getInstance(InstrumentationRegistry.getInstrumentation())`), `pressBack/pressHome/pressRecentApps`, `device.findObject(By...)` cached-node refresh and `StaleObjectException` recovery, the `executeShellCommand` `@Discouraged` path vs `Context.startActivity` plus `<queries>`, and the full `By` factory catalog. Use when the user reports `StaleObjectException`, `Espresso InjectEventSecurityException`, `findObject returns null`, asks "how do I open Settings from a test", "toggle Wi-Fi", "dismiss notification shade", or "test that spans my app and another".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - uiautomator
  - UiDevice
  - BySelector
  - UiObject2
  - UiSelector
  - Until
  - Configurator
  - StaleObjectException
  - cross-app-test
  - system-ui-test
  - pressHome
  - executeShellCommand
---

# Cross-App Tests with UiAutomator — Drive System UI and Other Apps

UiAutomator is the only AndroidX test framework that can drive UI **across process boundaries** — Settings, the dialer, system notifications, another app's activity. Espresso refuses (`InjectEventSecurityException`) when an event would cross to a foreign window. UiAutomator works against the platform `AccessibilityNodeInfo` tree, so it sees every window. This skill encodes the modern `BySelector` / `UiObject2` API, the singleton acquisition, the `StaleObjectException` recovery pattern, and the cross-app launch path.

## When to use this skill

- The test must drive system UI (Settings, dialer, system notifications, quick settings, recents).
- The test spans the developer's app and another app (share sheet, contacts picker, camera).
- Espresso threw `InjectEventSecurityException` because the test reached a foreign window.
- The user reports `StaleObjectException` and is debugging cached-node lifetime.
- The user asks how to send the device home, press recents, or toggle airplane mode in a test.
- The user mixes `BySelector` and `UiSelector` in the same test and is confused which is current.

## When NOT to use this skill

- The UI is entirely inside the developer's app — Espresso (Views) or Compose tests are faster and more reliable. See `../../espresso/writing-espresso-tests/SKILL.md` for Views.
- The runner stack is not yet set up — start with `../../runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.
- The user only needs to push device-side state changes (animations off, intent stub) — see `../../../adb/control/injecting-input-and-state/SKILL.md`.
- The test only needs to scrape logs after a run — see `../../../adb/observability/extracting-logs-with-logcat/SKILL.md`.

## Prerequisites

- Runner stack from `../../runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.
- `androidTestImplementation("androidx.test.uiautomator:uiautomator:2.3.0")`.
- A real or virtual Android device with the `accessibility_service` available (the default).
- For `Context.startActivity` to a foreign package on API 30+, a `<queries>` entry in the test APK manifest naming the target package or intent action.

## Workflow

- [ ] **1. Acquire `UiDevice` once per test, via the singleton with explicit instrumentation.** The no-arg `getInstance()` is `@Deprecated` (R4 lines 56-58) — it throws `IllegalStateException` if not previously initialized:

```kotlin
import androidx.test.platform.app.InstrumentationRegistry
import androidx.test.uiautomator.UiDevice

private lateinit var device: UiDevice

@Before fun setUp() {
    device = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation())
}
```

- [ ] **2. Send hard keys and global actions via `UiDevice` methods.** Source: R4 lines 88-104.

```kotlin
device.pressBack()
device.pressHome()
device.pressMenu()
device.pressRecentApps()                                // GLOBAL_ACTION_RECENTS
device.pressKeyCode(KeyEvent.KEYCODE_VOLUME_DOWN)
device.openNotification()                                // GLOBAL_ACTION_NOTIFICATIONS
device.openQuickSettings()                               // GLOBAL_ACTION_QUICK_SETTINGS
device.wakeUp()                                          // power-on if off
device.sleep()                                           // power-off if on
```

- [ ] **3. Find UI elements with `BySelector` (modern) via the `By` factory.** Selectors are immutable-ish builders; the package-private constructor enforces use of the static factories. Criteria are joined as implicit AND. Source: R4 lines 185-258.

```kotlin
import androidx.test.uiautomator.By

By.text("Submit")                              // exact, case-sensitive
By.textContains("ubmi")                        // case-sensitive substring
By.textMatches("Sub.*")                        // Pattern.compile regex
By.desc("Open menu")                           // content description
By.res("com.example.app", "submit_btn")        // package + res id
By.res("com.example.app:id/submit_btn")        // qualified res id
By.clazz("android.widget.Button")              // fully-qualified, or ".widget.Button" auto-prefixed with "android"
By.clazz(android.widget.Button::class.java)
By.pkg("com.android.settings")
By.depth(2)                                    // depth from window root
By.checkable(true)
By.hasChild(By.text("Inner"))                  // sugar for hasDescendant(child, 1)
By.hasDescendant(By.text("anywhere"))
By.hasParent(By.clazz("...LinearLayout"))
By.hasAncestor(By.res("...container"))
```

- [ ] **4. Resolve to `UiObject2` via `device.findObject(...)` and act.** `findObject` returns `null` when no match (NEVER throws). For interactions, prefer `device.wait(Until.findObject(By...), timeoutMillis)` so the lookup is robust to slow renders:

```kotlin
import androidx.test.uiautomator.Until

val submit = device.wait(Until.findObject(By.res("com.example", "submit")), 5_000)
    ?: error("Submit not found within 5s")

submit.click()
submit.longClick()
submit.setText("hello")
submit.clear()
submit.swipe(Direction.LEFT, 0.8f)            // percent ∈ [0,1]
submit.scroll(Direction.DOWN, 1.0f)
submit.scrollUntil(Direction.DOWN, Until.findObject(By.text("Bottom")))
submit.pinchOpen(0.5f)
submit.pinchClose(0.5f)
```

`UiObject2` caches an `AccessibilityNodeInfo`. Each action calls `getAccessibilityNodeInfo()` which calls `device.waitForIdle()` then `mCachedNode.refresh()`; on failure it runs registered `UiWatcher`s, retries once, and on a second failure throws `StaleObjectException` (R4 lines 281-286, 622-637).

- [ ] **5. Wait for asynchronous UI states with `Until` conditions.** Three condition flavors per R4 lines 495-540:

```kotlin
// SearchCondition (consume the device tree)
device.wait(Until.hasObject(By.text("Loaded")), 5_000)
device.wait(Until.gone(By.text("Loading")), 5_000)
val obj: UiObject2? = device.wait(Until.findObject(By.text("OK")), 5_000)
val list: List<UiObject2>? = device.wait(Until.findObjects(By.clazz("...Button")), 3_000)

// UiObject2Condition (consume a UiObject2 — for state predicates)
val checked = button.wait(Until.checked(true), 5_000)

// EventCondition (consume AccessibilityEvents — for window transitions)
val newWindowFired = device.performActionAndWait(
    { button.click() }, Until.newWindow(), 10_000,
)
```

- [ ] **6. Recover from `StaleObjectException` by re-finding.** `StaleObjectException extends RuntimeException` — unchecked, **must** be handled when holding a long-lived `UiObject2`:

```kotlin
fun clickRetry(selector: BySelector, timeoutMillis: Long = 5_000) {
    val obj = device.wait(Until.findObject(selector), timeoutMillis)
        ?: error("Not found: $selector")
    try {
        obj.click()
    } catch (e: StaleObjectException) {
        device.wait(Until.findObject(selector), timeoutMillis)?.click()
            ?: throw e
    }
}
```

Better: re-look-up before each interaction instead of holding a long-lived reference. Common causes: activity transition, Compose recomposition replacing semantic nodes, `RecyclerView` recycle, configuration change (R4 lines 639-645).

- [ ] **7. Launch foreign packages — prefer `Context.startActivity` over `executeShellCommand`.** `executeShellCommand` is `@Discouraged` per R4 lines 147-151 (`UiDevice.java:1432`); the Javadoc recommends `UiAutomation.executeShellCommandRwe` for stdin/stderr. For routine app launches, use the in-process intent route plus a `<queries>` manifest entry on API 30+:

```xml
<!-- src/androidTest/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <queries>
        <package android:name="com.android.settings" />
        <intent>
            <action android:name="android.settings.SETTINGS" />
        </intent>
    </queries>
    <!-- ... -->
</manifest>
```

```kotlin
val ctx = ApplicationProvider.getApplicationContext<Context>()
ctx.startActivity(
    Intent(android.provider.Settings.ACTION_WIFI_SETTINGS)
        .addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP)
)
device.wait(Until.hasObject(By.pkg("com.android.settings").depth(0)), 10_000)
```

If the user really needs the shell route (e.g. a deep-link no `Intent` resolves):

```kotlin
device.executeShellCommand("am start -n com.android.settings/.Settings")
device.wait(Until.hasObject(By.pkg("com.android.settings")), 5_000)
```

- [ ] **8. Tune `Configurator` global timeouts when needed; restore on teardown.** Settings are process-wide. Defaults (R4 lines 550-558): wait-for-idle 10s, wait-for-selector 10s (legacy `UiObject` only), action-acknowledgment 3s, scroll-event-wait 1s.

```kotlin
import androidx.test.uiautomator.Configurator

@Before fun setUp() {
    val c = Configurator.getInstance()
    originalIdle = c.waitForIdleTimeout
    c.waitForIdleTimeout = 5_000   // tighter for fast tests
}

@After fun tearDown() {
    Configurator.getInstance().waitForIdleTimeout = originalIdle
}
```

`setKeyInjectionDelay` is `@Deprecated` ("This parameter is no longer used (text is set directly rather than by key)." — R4 lines 575-577). Do not call it.

- [ ] **9. Recognize when to fall back to legacy `UiSelector` / `UiObject`.** `UiSelector` re-resolves the node on every action via `Configurator.getWaitForSelectorTimeout()`, so it cannot go stale, but every call is more expensive and uses the checked `UiObjectNotFoundException`. The modern `BySelector` / `UiObject2` is the recommended default. Use legacy only for interop with old test code or when re-resolution semantics are mandatory. R4 lines 594-619.

- [ ] **10. Never call `findObject(By.text("..."))` immediately after launch and assume it sees the target window.** `By.text` matches whatever is on screen now, which may be a transient splash or the previous app. Always combine with `By.pkg(...)` or wait explicitly:

```kotlin
device.pressHome()
ctx.startActivity(Intent(Settings.ACTION_SETTINGS).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK))
device.wait(Until.hasObject(By.pkg("com.android.settings")), 5_000)   // gate on package
device.wait(Until.findObject(By.text("Network & internet")), 5_000)?.click()
```

## Patterns

### Pattern: WRONG vs RIGHT — driving Settings without waiting on package transition

```kotlin
// WRONG
device.findObject(By.text("Settings")).click()
// WRONG because: at the moment of the call, By.text("Settings") matches whatever text reads
// "Settings" anywhere on the current screen — that might be a launcher tile, a notification,
// or a previous-app menu item. The test silently navigates to the wrong place.
```

```kotlin
// RIGHT
device.pressHome()
device.executeShellCommand("am start -a android.settings.SETTINGS")
device.wait(Until.hasObject(By.pkg("com.android.settings")), 5_000)
device.wait(Until.findObject(By.text("Network & internet")), 5_000)?.click()
```

### Pattern: WRONG vs RIGHT — `UiDevice.getInstance()` no-arg

```kotlin
// WRONG
val device = UiDevice.getInstance()
// WRONG because: UiDevice.java:344 — the no-arg getInstance() is @Deprecated and throws
// IllegalStateException if the singleton has not been previously initialized with the
// instrumentation argument. Setting it implicitly via androidTest is unreliable across
// runner versions.
```

```kotlin
// RIGHT
val device = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation())
```

### Pattern: WRONG vs RIGHT — long-lived `UiObject2` across a transition

```kotlin
// WRONG
val tile = device.wait(Until.findObject(By.descContains("Wi-Fi")), 5_000)
device.openQuickSettings()
tile.click()                            // throws StaleObjectException — quick settings replaced the layout
```

```kotlin
// RIGHT — re-acquire after the transition
device.openQuickSettings()
val tile = device.wait(Until.findObject(By.descContains("Wi-Fi").clickable(true)), 5_000)
tile?.click()
device.pressBack()
```

### Pattern: dismissing all notifications

```kotlin
device.openNotification()
val shade = device.wait(
    Until.findObject(By.res("com.android.systemui:id/notification_stack_scroller")),
    5_000,
) ?: error("Notification shade not visible")

for (n in shade.findObjects(By.res("com.android.systemui:id/notification_layout"))) {
    n.swipe(Direction.RIGHT, 1.0f)
}
device.pressBack()
```

### Pattern: launching the app under test from a clean state

```kotlin
@Before fun launchAppFromHome() {
    device.pressHome()
    val ctx = ApplicationProvider.getApplicationContext<Context>()
    val launch = ctx.packageManager.getLaunchIntentForPackage("com.example.app")!!
        .addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP)
    ctx.startActivity(launch)
    device.wait(Until.hasObject(By.pkg("com.example.app").depth(0)), 5_000)
}
```

## Mandatory rules

- **MUST** acquire `UiDevice` via `UiDevice.getInstance(InstrumentationRegistry.getInstrumentation())`. The no-arg `getInstance()` is `@Deprecated` and unreliable.
- **MUST** prefer `BySelector` / `UiObject2` over `UiSelector` / `UiObject` for new tests. Legacy is for interop only.
- **MUST** wait on a transition with `device.wait(Until.hasObject(By.pkg("...")), timeout)` before resolving inner elements after a cross-window change. Bare `findObject` runs against the current window without gating.
- **MUST** handle `StaleObjectException` (re-find via `device.wait(Until.findObject(By...), timeout)`) when keeping a `UiObject2` reference across recompose, transition, or recyclerview recycle.
- **MUST** restore `Configurator` settings in `@After` if a test mutated them. Settings are process-wide and leak across tests.
- **MUST NOT** call `Configurator.setKeyInjectionDelay` — `@Deprecated` and a no-op.
- **MUST NOT** rely on `executeShellCommand` for routine app launches — it is `@Discouraged`. Use `Context.startActivity` plus a `<queries>` manifest entry on API 30+.
- **MUST NOT** use `device.findObject(By.text("..."))` without gating on package — text matches the current window contents, which may be a transient splash.
- **PREFERRED:** wrap any single interaction in a `device.wait(Until.findObject(...), timeout)?.click()` rather than calling `findObject` then `click` two lines apart. The two-step form races with rendering.
- **PREFERRED:** add `<queries>` for any foreign package the test launches — silent failures on API 30+ otherwise.

## Verification

- [ ] No source file calls `UiDevice.getInstance()` with no arguments.
- [ ] No source file calls `Configurator.setKeyInjectionDelay`.
- [ ] Every long-lived `UiObject2` reference has a `try { ... } catch (StaleObjectException)` around its action — or the test re-finds before each interaction.
- [ ] Tests that launch foreign packages have a `<queries>` entry for that package or its intent action in `src/androidTest/AndroidManifest.xml`.
- [ ] `./gradlew :<module>:connectedDebugAndroidTest` runs without `IllegalStateException` from `UiDevice.getInstance()` and without unhandled `StaleObjectException` propagating to the test result.
- [ ] No `findObject(By.text(...))` is called immediately after a cross-window transition without a preceding `device.wait(Until.hasObject(By.pkg(...)), ...)`.
- [ ] If `Configurator` was mutated in `@Before`, an `@After` restores the original values.

## References

- Android Developers — UiAutomator overview: https://developer.android.com/training/testing/other-components/ui-automator
- AndroidX Test (UiAutomator) release notes: https://developer.android.com/jetpack/androidx/releases/test
- UiAutomator API reference: https://developer.android.com/reference/androidx/test/uiautomator/package-summary
- `androidx/test/uiautomator/UiDevice.java` — the singleton entry point. Lines 344 (`@Deprecated` no-arg `getInstance`), 357 (current `getInstance(Instrumentation)`), 1432 (`@Discouraged executeShellCommand`).
- `androidx/test/uiautomator/By.java` — static factory for `BySelector`. Source for `text/desc/res/clazz/pkg/depth/checkable/hasChild/hasDescendant/hasParent/hasAncestor`.
- `androidx/test/uiautomator/BySelector.java` (838 lines) — full criteria catalog. Lines 128 (auto-prefix `.widget.Button` with `android`), 198/211/224 (regex / contains / starts-with conventions), 605-650 (`maxDepth`), 660 (`displayId`).
- `androidx/test/uiautomator/UiObject2.java` (1152 lines) — `click`, `setText`, `swipe`, `scroll`, `scrollUntil`, `pinchOpen/Close`, `fling`. Line 1031 — `getAccessibilityNodeInfo()` throws `StaleObjectException` after refresh failure + watcher retry.
- `androidx/test/uiautomator/Until.java` lines 43-456 — `hasObject`, `gone`, `findObject`, `findObjects`, `newWindow`, `scrollFinished`, plus `UiObject2Condition` predicates.
- `androidx/test/uiautomator/Configurator.java` lines 36 (warning to restore), 108 (`waitForSelector` legacy-only), 163 (`actionAcknowledgment` legacy-only), 189 (`setKeyInjectionDelay` `@Deprecated`), 295 (`setDefaultDisplayId`).
- `androidx/test/uiautomator/StaleObjectException.java` — `extends RuntimeException`.
- `tasks/research/R4-uiautomator.md` — full UiAutomator 2.3.0 deep-dive. Lines 56-58 (deprecated singleton), 185-258 (BySelector catalog), 268-369 (UiObject2 actions), 495-540 (`Until` conditions), 594-619 (modern vs legacy table), 622-662 (StaleObjectException recovery).
- `docs/CORPUS.md` Section H.6 — UiAutomator essentials and the StaleObjectException recovery summary.
