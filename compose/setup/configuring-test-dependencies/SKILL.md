---
name: configuring-test-dependencies
description: Use this skill to wire the correct Gradle dependency matrix for Jetpack Compose UI tests. Covers `androidTestImplementation("androidx.compose.ui:ui-test-junit4")`, the `debugImplementation("androidx.compose.ui:ui-test-manifest")` requirement that makes `createComposeRule()` work, the host-test (Robolectric) trio, the accessibility add-ons (`ui-test-accessibility`, `ui-test-junit4-accessibility`), and the `TestManifestGradleConfiguration` lint warning. Use when the user reports `ActivityNotFoundException: ComponentActivity`, `createComposeRule unresolved`, `lint warning ui-test-manifest`, `Cannot find test rule`, or asks "what dependencies do I need for Compose UI tests" / "why won't my Compose test compile".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - gradle-dependencies
  - ui-test-manifest
  - debug-implementation
  - createComposeRule
  - ComponentActivity
  - ui-test-junit4
  - robolectric
  - accessibility-checks
---

# Configuring Test Dependencies — Get the Gradle Matrix Right

Compose UI tests fail at the dependency layer in three predictable ways: the JUnit4 surface is on the wrong source set, the `ui-test-manifest` artifact is missing or on the wrong configuration, or the host-test trio is incomplete. This skill encodes the exact matrix and the lint warning that polices it. Subsequent skills (`choosing-test-rule-vs-runtest/SKILL.md`, `setting-up-host-vs-device-tests/SKILL.md`) assume the dependencies are correct.

## When to use this skill

- The user reports `java.lang.RuntimeException: Could not launch activity within 45 seconds. Examined activities: ... androidx.activity.ComponentActivity` or `ActivityNotFoundException`.
- The IDE flags `createComposeRule` as unresolved or `import androidx.compose.ui.test.junit4.createComposeRule` as red.
- Lint surfaces `TestManifestGradleConfiguration` (severity WARNING) on a `build.gradle.kts` line.
- The user wants to add Robolectric / host (JVM) tests for Compose and asks which artifacts to depend on.
- The user wants to opt into `enableAccessibilityChecks(...)` and is unsure which artifact owns it.

## When NOT to use this skill

- Dependencies are already correct and the user is choosing between `createComposeRule()` and `runComposeUiTest { }` — use `./choosing-test-rule-vs-runtest/SKILL.md`.
- Dependencies compile but the test runs in the wrong source set (`test/` vs `androidTest/`) — use `./setting-up-host-vs-device-tests/SKILL.md`.
- The test compiles and runs but is flaky/timing-out — start with `../../synchronization/synchronizing-with-idle/SKILL.md`.

## Prerequisites

- Android Gradle Plugin with the `com.android.application` or `com.android.library` plugin applied.
- A Compose-enabled module (the `org.jetbrains.kotlin.plugin.compose` plugin or `buildFeatures.compose = true`).
- A JUnit4 dependency on the test classpath (Compose's `ui-test-junit4` artifact still uses JUnit4; JUnit5 requires a third-party Vintage engine bridge and is out of scope).

## Workflow

- [ ] **1. Decide which test surfaces the module needs.** A typical Compose feature module wants three: instrumentation tests for screenshot/RenderThread/full-stack coverage, host (Robolectric) tests for fast logic checks, and the manifest helper for `createComposeRule()`. Pick from the matrix in step 2.

- [ ] **2. Add dependencies on the correct configurations.** Copy this into the module's `build.gradle.kts`:

```kotlin
dependencies {
    // --- Instrumentation tests (src/androidTest/) ---
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    // MUST be debugImplementation (see Pattern: WRONG vs RIGHT for ui-test-manifest below)
    debugImplementation("androidx.compose.ui:ui-test-manifest")

    // --- Host tests with Robolectric (src/test/) ---
    testImplementation("androidx.compose.ui:ui-test")            // common test API
    testImplementation("androidx.compose.ui:ui-test-junit4")     // ComposeTestRule + createComposeRule
    testImplementation("androidx.compose.ui:ui-test-manifest")   // ComponentActivity manifest entry
    testImplementation("org.robolectric:robolectric:4.13")       // pin a version

    // --- Optional: accessibility validation (API 34+) ---
    androidTestImplementation("androidx.compose.ui:ui-test-junit4-accessibility")
    // Or, when using runComposeUiTest { } (no Rule):
    androidTestImplementation("androidx.compose.ui:ui-test-accessibility")
}
```

The exact GA versions come from the Compose UI BOM or the developer's chosen `composeVersion`. Pin Robolectric independently — it does not ship in the Compose BOM.

- [ ] **3. If the test fails with `ActivityNotFoundException` for `androidx.activity.ComponentActivity`, the manifest artifact is the cause 99 percent of the time.** That artifact is a single-file library that merges this `<activity>` entry into the test APK manifest, taken from `compose/ui/ui-test-manifest/src/main/AndroidManifest.xml`:

```xml
<activity android:theme="@android:style/Theme.Material.Light.NoActionBar"
    android:name="androidx.activity.ComponentActivity" android:exported="true" />
```

Without that entry, `createComposeRule()` and `runComposeUiTest { }` (which both default to launching `androidx.activity.ComponentActivity` via `ActivityScenario.launch`) crash at launch time. Diagnosis:

  - Confirm `ui-test-manifest` is in the dependency list at all.
  - Confirm it is on `debugImplementation` (or `testImplementation` for host tests). If it is on `androidTestImplementation`, lint flags it and Gradle still wires it in — but the AGP variant model intentionally separates `debug` and `androidTest` manifest merging, and putting it on `androidTestImplementation` does NOT merge the `<activity>` into the host APK that the instrumentation runs against. The lint check exists precisely to catch this.

- [ ] **4. Address the `TestManifestGradleConfiguration` lint warning by moving the dependency to `debugImplementation`.** The lint check lives in `compose/ui/ui-test-manifest-lint/src/main/java/androidx/compose/ui/test/manifest/lint/GradleDebugConfigurationDetector.kt`. Severity is `Severity.WARNING` and the quick fix replaces the configuration name with `debugImplementation`. The check fires when the artifact appears on `implementation`, `api`, `compileOnly`, `runtimeOnly`, `androidTestImplementation`, `testImplementation`, or any custom configuration whose name starts with `android` / `test` / one of the supported config names.

- [ ] **5. Decide whether the manifest artifact is needed at all.** It is required when:
  - The test uses `createComposeRule()` (no type parameter — defaults to `androidx.activity.ComponentActivity`).
  - The test uses `runComposeUiTest { }` (no type parameter — same default).

  It is NOT required when:
  - The test uses `createAndroidComposeRule<MyActivity>()` AND the developer declares `<activity android:name=".MyActivity" .../>` themselves in `src/androidTest/AndroidManifest.xml` (or `src/debug/AndroidManifest.xml`).
  - The test uses `createEmptyComposeRule()` and launches its own `ActivityScenario` against an Activity already declared in the production manifest.

- [ ] **6. For host tests, ensure all three artifacts are on `testImplementation`.** Robolectric drives the JVM Looper, but Compose's idling/clock implementations require `ui-test` (common) plus `ui-test-junit4` (`ComposeTestRule` interface and `createComposeRule` actual). Without `ui-test-manifest` on `testImplementation`, Robolectric throws the same `ActivityNotFoundException` because it parses the same manifest. The host source set in androidx itself uses this configuration — see `compose/ui/ui-test/src/androidHostTest/kotlin/androidx/compose/ui/test/RobolectricComposeTest.kt`.

- [ ] **7. Add accessibility artifacts only if the developer plans to call `enableAccessibilityChecks`.** They are independent of the JUnit4 surface:
  - `ui-test-accessibility` extends `ComposeUiTest` (the receiver of `runComposeUiTest { }`).
  - `ui-test-junit4-accessibility` extends `ComposeTestRule` / `AndroidComposeTestRule`.
  Both are `@RequiresApi(34)`, both are no-op on Robolectric (`Build.FINGERPRINT == "robolectric"` triggers a logged warning) — see the accessibility skill in this repo for follow-up.

- [ ] **8. Sync Gradle and re-run the test.** A clean `./gradlew :app:connectedDebugAndroidTest` (instrumentation) or `./gradlew :app:testDebugUnitTest` (host) should compile and locate `ComponentActivity`.

## Patterns

### Pattern: WRONG vs RIGHT — `ui-test-manifest` configuration

```kotlin
// WRONG
dependencies {
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    androidTestImplementation("androidx.compose.ui:ui-test-manifest")
}
// WRONG because: lint emits TestManifestGradleConfiguration (WARNING) and the manifest
// merger does not pull the <activity> entry into the merged APK manifest from the
// androidTest source set. createComposeRule() crashes with ActivityNotFoundException.
```

```kotlin
// RIGHT
dependencies {
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-test-manifest")
}
```

The lint check explicitly allows `debugImplementation` and warns on every other configuration. From `GradleDebugConfigurationDetector.kt`:

```text
"The androidx.compose.ui:ui-test-manifest dependency is needed for launching a
 Compose host, such as with createComposeRule. However, it only needs to be present
 in testing configurations therefore use this dependency with the debugImplementation
 configuration"
```

### Pattern: WRONG vs RIGHT — host-test dependencies

```kotlin
// WRONG
dependencies {
    testImplementation("androidx.compose.ui:ui-test-junit4")
    testImplementation("org.robolectric:robolectric:4.13")
    // Missing ui-test-manifest -> ActivityNotFoundException at runComposeUiTest launch
    // Missing ui-test          -> linker errors on common test API symbols
}
```

```kotlin
// RIGHT
dependencies {
    testImplementation("androidx.compose.ui:ui-test")
    testImplementation("androidx.compose.ui:ui-test-junit4")
    testImplementation("androidx.compose.ui:ui-test-manifest")
    testImplementation("org.robolectric:robolectric:4.13")
}
```

### Pattern: WRONG vs RIGHT — custom Activity without its own manifest entry

```kotlin
// WRONG
@get:Rule val rule = createAndroidComposeRule<MyHostActivity>()
// build.gradle.kts has only debugImplementation("androidx.compose.ui:ui-test-manifest")
// and src/androidTest/AndroidManifest.xml does NOT declare MyHostActivity.
// WRONG because: ui-test-manifest only declares ComponentActivity. Custom activities
// must be declared in the test APK manifest by the developer. Result: ActivityNotFoundException.
```

```xml
<!-- RIGHT — src/androidTest/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application>
        <activity android:name=".MyHostActivity" android:exported="true" />
    </application>
</manifest>
```

### Pattern: complete sample `build.gradle.kts`

```kotlin
plugins {
    id("com.android.library")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
}

android {
    namespace = "com.example.feature"
    compileSdk = 35
    defaultConfig {
        minSdk = 23
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }
    buildFeatures { compose = true }
    testOptions {
        unitTests.isIncludeAndroidResources = true   // required for Robolectric + Compose
    }
}

dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2025.06.00")
    implementation(composeBom)
    androidTestImplementation(composeBom)
    testImplementation(composeBom)

    // Production
    implementation("androidx.compose.ui:ui")
    implementation("androidx.activity:activity-compose")

    // Instrumentation
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-test-manifest")
    androidTestImplementation("androidx.test.ext:junit:1.2.1")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.6.1")

    // Host (Robolectric)
    testImplementation("androidx.compose.ui:ui-test")
    testImplementation("androidx.compose.ui:ui-test-junit4")
    testImplementation("androidx.compose.ui:ui-test-manifest")
    testImplementation("org.robolectric:robolectric:4.13")
    testImplementation("junit:junit:4.13.2")
}
```

`testOptions.unitTests.isIncludeAndroidResources = true` is required so Robolectric can read merged resources during host tests of Compose content.

## Mandatory rules

- **MUST** put `androidx.compose.ui:ui-test-manifest` on `debugImplementation` for instrumentation tests, and on `testImplementation` for host tests. Never on `androidTestImplementation`, `implementation`, or `api`.
- **MUST** keep `ui-test-junit4` on the same source set as the test that imports `createComposeRule` / `ComposeTestRule`. Mixing `androidTestImplementation` (instrumentation) and `testImplementation` (host) in the same module is fine; mixing them on the wrong test class is a compile error.
- **MUST** declare any non-`ComponentActivity` test host (custom Activity) in the test APK manifest at `src/androidTest/AndroidManifest.xml` or `src/debug/AndroidManifest.xml`. The `ui-test-manifest` artifact only declares `androidx.activity.ComponentActivity`.
- **MUST NOT** remove the `TestManifestGradleConfiguration` lint check via `lintOptions { disable("TestManifestGradleConfiguration") }`. Apply the quick fix instead — the check exists to prevent silent test failures.
- **MUST NOT** put `ui-test-manifest` on a release configuration (`releaseImplementation`, `implementation`). The artifact is purely a test scaffold and bloats production APKs with an unwanted `<activity>` entry.
- **PREFERRED:** rely on the Compose BOM (`androidx.compose:compose-bom`) so all `ui-test-*` artifacts share a coherent version. Robolectric is independent and pins separately.
- **PREFERRED:** when in doubt about which artifact owns a symbol, search `androidx/compose/ui/ui-test*` paths from the corpus rather than guessing — see References.

## Verification

- [ ] `./gradlew :<module>:connectedDebugAndroidTest` compiles and runs without `ActivityNotFoundException` for `androidx.activity.ComponentActivity`.
- [ ] `./gradlew :<module>:lint` does NOT report `TestManifestGradleConfiguration`.
- [ ] `./gradlew :<module>:testDebugUnitTest` compiles when host tests use `runComposeUiTest { }` or `createComposeRule()`.
- [ ] The dependency block visibly separates `androidTestImplementation` (instrumentation), `debugImplementation` (manifest), `testImplementation` (host trio).
- [ ] If the developer added accessibility artifacts, `enableAccessibilityChecks(...)` resolves at compile time inside an instrumentation test.
- [ ] The test APK manifest contains exactly one `<activity android:name="androidx.activity.ComponentActivity"/>` entry (run `./gradlew :app:processDebugAndroidTestManifest` and inspect `build/intermediates/.../AndroidManifest.xml`).

## References

- Compose testing setup (Android Developers): https://developer.android.com/develop/ui/compose/testing#setup
- Compose testing overview: https://developer.android.com/develop/ui/compose/testing
- Compose UI release notes: https://developer.android.com/jetpack/androidx/releases/compose-ui
- Compose Multiplatform testing: https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-test.html
- `compose/ui/ui-test-manifest-lint/src/main/java/androidx/compose/ui/test/manifest/lint/GradleDebugConfigurationDetector.kt` — `TestManifestGradleConfiguration` issue definition (severity WARNING, quick-fix to `debugImplementation`).
- `compose/ui/ui-test-manifest/src/main/AndroidManifest.xml` — the merged `<activity android:name="androidx.activity.ComponentActivity">` entry that `createComposeRule()` depends on.
- `compose/ui/ui-test/src/androidMain/kotlin/androidx/compose/ui/test/ComposeUiTest.android.kt` — `runComposeUiTest` launches `ComponentActivity::class.java` by default (lines 184-197).
- `compose/ui/ui-test/src/androidHostTest/kotlin/androidx/compose/ui/test/RobolectricComposeTest.kt` — canonical host-test class skeleton with Robolectric (`@RunWith(AndroidJUnit4::class) @Config(minSdk = RobolectricMinSdk)`).
- `compose/ui/ui-test/src/androidHostTest/kotlin/androidx/compose/ui/test/Constants.kt` — `internal const val RobolectricMinSdk = 23`.
