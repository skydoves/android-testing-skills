---
name: organizing-test-source-sets
description: Use this skill to organize Android test source sets — `src/test/`, `src/androidTest/`, the community `src/sharedTest/` convention, and the modern KMP-style `androidHostTest` / `androidDeviceTest` split that Compose itself adopted after AGP 7.2 broke the classic `sourceSets` wiring. Includes the `testImplementation` / `androidTestImplementation` / `debugImplementation` configuration matrix per Google's `/training/testing/local-tests` and `/training/testing/instrumented-tests` pages. Use when the user asks "where do I put my tests", "test or androidTest", "sharedTest setup", "AGP 7.2 broke my sourceSets", "androidHostTest vs androidDeviceTest", "testImplementation vs androidTestImplementation vs debugImplementation", or "why is my Robolectric test in androidTest".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - source-sets
  - testImplementation
  - androidTestImplementation
  - debugImplementation
  - sharedTest
  - androidHostTest
  - androidDeviceTest
  - kmp-testing
  - agp-source-sets
---

# Organizing Test Source Sets — `src/test/`, `src/androidTest/`, and the KMP Split

Where a test file lives determines which JVM runs it, which dependencies it sees, and whether it ships in the test APK. Most modules need only `src/test/` and `src/androidTest/`. Modules that share helpers between host and device tests historically used `src/sharedTest/` (a Codelab convention, not Google-documented as such), which broke on AGP 7.2+ and pushed Compose itself to the KMP-style `androidHostTest` / `androidDeviceTest` split. This skill encodes the matrix so the agent can place a test file with confidence.

## When to use this skill

- The user asks "where do I put my tests" or "src/test/ or src/androidTest/".
- The user asks "what's testImplementation vs androidTestImplementation vs debugImplementation".
- The user reports `Unresolved reference: createComposeRule` and the agent suspects the dependency is on the wrong configuration.
- The user is migrating off `src/sharedTest/` after AGP 7.2+ broke their `sourceSets { test.java.srcDir(...) }` block.
- The user mentions `androidHostTest` / `androidDeviceTest` and asks how to set them up in a non-KMP module.
- The user asks "why is my Robolectric test in `src/androidTest/`" — it should be in `src/test/`.

## When NOT to use this skill

- The user is wiring Gradle dependencies for Compose test (`ui-test-junit4`, `ui-test-manifest`) — use `../../../compose/setup/configuring-test-dependencies/SKILL.md`.
- The user is choosing test scope (small/medium/big) — use `../../concepts/understanding-the-testing-pyramid/SKILL.md`.
- The user is configuring `AndroidJUnitRunner` itself — use `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.
- The user is configuring a JUnit4 runner for JVM tests — use `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`.

## Prerequisites

- A module with the Android Gradle Plugin (`com.android.application` or `com.android.library`).
- Familiarity with Gradle configuration names (`testImplementation`, `androidTestImplementation`, `debugImplementation`).
- An understanding that JVM tests run on the host's JVM and instrumented tests run on a device/emulator. (See `../../concepts/understanding-the-testing-pyramid/SKILL.md`.)

## The two source sets Google documents

### `src/test/` — local JVM tests

Per `/training/testing/local-tests`:

> "By default, the source files for local unit tests are placed in `module-name/src/test/`. This directory already exists when you create a new project using Android Studio."
> — `developer.android.com/training/testing/local-tests`

> "A **local** test runs directly on your own workstation, rather than an Android device or emulator. As such, it uses your local Java Virtual Machine (JVM)."
> — `developer.android.com/training/testing/local-tests`

Driven by `testImplementation`. Runs via `./gradlew test` or `./gradlew testDebugUnitTest`. No Android device, no emulator. The Android framework is unavailable unless Robolectric simulates it.

### `src/androidTest/` — instrumented tests

Per `/training/testing/instrumented-tests`:

> "In your Android Studio project, you store the source files for instrumented tests in `module-name/src/androidTest/java/`. This directory already exists when you create a new project and contains an example instrumented test."
> — `developer.android.com/training/testing/instrumented-tests`

Driven by `androidTestImplementation`. Runs via `./gradlew connectedAndroidTest` or `./gradlew connectedDebugAndroidTest`. Requires a connected device or emulator. Uses `AndroidJUnitRunner` per CORPUS §H.2.

The page is also explicit on the `testImplementation` vs `androidTestImplementation` split:

> "**Note:** `testImplementation` adds dependencies for local tests and `androidTestImplementation` adds dependencies for Instrumented tests."
> — `developer.android.com/training/testing/local-tests`

## The configuration matrix

```
Configuration                Source set        Variant       Ships in
-----------------------------------------------------------------------------------
implementation               main              all           release + debug + tests
api                          main              all           release + debug + tests
debugImplementation          debug only        debug         debug APK + androidTest APK
releaseImplementation        release only      release       release APK only
testImplementation           src/test/         all (host)    JVM test classpath
testDebugImplementation      src/test/         debug (host)  JVM test classpath, debug only
androidTestImplementation    src/androidTest/  all (device)  androidTest APK
androidTestUtil              n/a               all           orchestrator/util install (NOT classpath)
```

Three configurations the agent must keep straight:

- **`testImplementation`** — JVM (host) tests. Includes Mockito, MockK, Robolectric, kotlinx-coroutines-test, Turbine.
- **`androidTestImplementation`** — instrumented (device) tests. Includes Espresso, UiAutomator, AndroidJUnitRunner, Compose `ui-test-junit4`.
- **`debugImplementation`** — main `debug` variant. Used by `androidx.compose.ui:ui-test-manifest` and `androidx.fragment:fragment-testing-manifest` because the test APK is built on top of the debug APK and needs the manifest entries merged in.

`androidx.compose.ui:ui-test-manifest` MUST go on `debugImplementation`, not `androidTestImplementation` — see `../../../compose/setup/configuring-test-dependencies/SKILL.md` for the lint check (`TestManifestGradleConfiguration`) that enforces this.

## `src/sharedTest/` — community convention, NOT Google-documented

Per `tasks/research/R8-android-fundamentals.md`:

> `src/sharedTest/` is **not mentioned by name** on any of the six in-scope `developer.android.com/training/testing/...` pages or on the fetched Hilt-testing / rules pages.

It originated in Google's own *Architecture Blueprints* sample and the *Advanced Android Testing* Codelab. The pattern uses Gradle `sourceSets` to inject the same Java/Kotlin folder into BOTH `test` and `androidTest`:

```kotlin
// pre-AGP 7.2 — the classic sharedTest wiring
android {
    sourceSets {
        getByName("test").java.srcDir("src/sharedTest/java")
        getByName("androidTest").java.srcDir("src/sharedTest/java")
    }
}
```

Why it existed: write a test once, run it as a fast Robolectric host test in CI and as a high-fidelity instrumented test on emulator. The test class uses `androidx.test.ext.junit.runners.AndroidJUnit4` (which dispatches correctly in both environments per CORPUS §G.6).

Why it broke: AGP 7.2+ enforces that the same source directory cannot be shared across multiple source sets at build time — the build itself refuses, with an error pointing at the duplicated `srcDir(...)` call. (The mode of failure is build-time refusal in AGP, not a runtime `LinkageError`.) Tracked at https://github.com/robolectric/robolectric/issues/7432 (Robolectric maintainer thread, with reproductions and AGP-version table). Many modules either:

- Stopped sharing tests and accepted duplication.
- Migrated to a separate Gradle sub-module (`:core-testing`) that ships test helpers via `testImplementation` and `androidTestImplementation`.
- Adopted KMP-style source set names (`androidHostTest` / `androidDeviceTest`).

## KMP-style `androidHostTest` / `androidDeviceTest`

Compose UI tests internally use this layout, per CORPUS §C ("Source set conventions (KMP)"):

```
commonTest
└── androidCommonTest          (helpers shared by host + device)
    ├── androidDeviceTest      (instrumentation APK on emulator/device)
    └── androidHostTest        (Robolectric on JVM)
```

Real androidx evidence: `compose/ui/ui-test/src/androidHostTest/kotlin/androidx/compose/ui/test/RobolectricComposeTest.kt` is a host (Robolectric) test, while `compose/material3/material3/src/androidDeviceTest/.../SwitchTest.kt` is a device test. Both compile against `commonTest` for shared helpers.

For non-KMP Android-only modules, the same effect can be approximated with separate sub-modules:

```kotlin
// :feature:home/build.gradle.kts — production module
plugins { id("com.android.library") }

dependencies {
    testImplementation(project(":feature:home-testing"))
    androidTestImplementation(project(":feature:home-testing"))
}

// :feature:home-testing/build.gradle.kts — pure Kotlin or Android-library "test fixtures"
plugins { id("java-library") }   // or com.android.library if Android types needed
```

The `:feature:home-testing` module hosts `FakeUserRepository`, `MainDispatcherRule`, custom matchers — anything both host and device tests want.

Alternative: AGP 7.0+ `testFixtures` flag (`android { testFixtures { enable = true } }`), which exposes `testFixtures()` Gradle artifacts for `testFixturesImplementation` and is consumable from sibling modules' `testImplementation` / `androidTestImplementation`.

**MUST NOT** cite `developer.android.com/training/testing/...` for `sharedTest` — none of those pages document it. **MUST** cite the Architecture Blueprints repo or `tasks/research/R8-android-fundamentals.md` instead.

## File placement decision tree

```
Where does this test go?
│
├── Pure Kotlin/JVM logic, no Android framework      → src/test/
│   (testImplementation deps)
│
├── Robolectric (Android framework simulated on JVM) → src/test/
│   (testImplementation deps + Robolectric)
│   NOT src/androidTest/.
│
├── ViewModel + coroutines + fakes                   → src/test/
│
├── Real Room database integration                    → src/androidTest/
│   (small instrumented test per /fundamentals)
│
├── Espresso UI test                                  → src/androidTest/
├── UiAutomator cross-app test                        → src/androidTest/
├── Compose UI test against real Activity            → src/androidTest/
│   + debugImplementation("...:ui-test-manifest")
│
├── Compose UI test on Robolectric                    → src/test/
│   (testImplementation("...:ui-test-junit4") + Robolectric)
│
├── Test helper (FakeRepository, MainDispatcherRule)  → :core-testing module OR src/sharedTest/
│   (with the AGP 7.2+ caveats above)
```

## Patterns

### Pattern: WRONG vs RIGHT — `testImplementation` vs `androidTestImplementation`

```kotlin
// WRONG — Espresso on testImplementation
dependencies {
    testImplementation("androidx.test.espresso:espresso-core:3.7.0")
    testImplementation("androidx.test.ext:junit:1.3.0")
}
// WRONG because: Espresso requires a real or simulated Android device. testImplementation
// only puts the JAR on the JVM classpath. The test compiles but ClassNotFoundException /
// IllegalStateException at runtime when Espresso looks for an Instrumentation instance.
```

```kotlin
// RIGHT — Espresso on androidTestImplementation
dependencies {
    androidTestImplementation("androidx.test.espresso:espresso-core:3.7.0")
    androidTestImplementation("androidx.test.ext:junit:1.3.0")
}
```

### Pattern: WRONG vs RIGHT — Robolectric in `src/androidTest/`

```kotlin
// WRONG
// src/androidTest/java/com/example/SomeRobolectricTest.kt
@RunWith(RobolectricTestRunner::class)
class SomeRobolectricTest { /* ... */ }
// WRONG because: Robolectric is a JVM Android simulator (per /fundamentals: "Big local test:
// You can use an Android simulator that runs locally, such as Robolectric"). It belongs in
// src/test/. Putting it in src/androidTest/ either fails to find Robolectric (which is on
// testImplementation) or runs it on the device JVM where its shadow mechanism does not work.
```

```kotlin
// RIGHT
// src/test/java/com/example/SomeRobolectricTest.kt
@RunWith(AndroidJUnit4::class)         // dispatches to Robolectric on JVM
@Config(sdk = [Build.VERSION_CODES.UPSIDE_DOWN_CAKE])
class SomeRobolectricTest { /* ... */ }
```

### Pattern: WRONG vs RIGHT — `ui-test-manifest` configuration

```kotlin
// WRONG
dependencies {
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    androidTestImplementation("androidx.compose.ui:ui-test-manifest")  // wrong config
}
// WRONG because: ui-test-manifest contributes <activity android:name="ComponentActivity">
// to the merged debug APK manifest. On androidTestImplementation, the merger does not pick
// it up; createComposeRule() crashes with ActivityNotFoundException. Lint also flags this
// as TestManifestGradleConfiguration (WARNING).
```

```kotlin
// RIGHT
dependencies {
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")
    debugImplementation("androidx.compose.ui:ui-test-manifest")
}
```

See `../../../compose/setup/configuring-test-dependencies/SKILL.md` for the lint detail.

### Pattern: AGP 7.2+ `sharedTest` migration

```kotlin
// BROKE on AGP 7.2+
android {
    sourceSets {
        getByName("test").java.srcDir("src/sharedTest/java")
        getByName("androidTest").java.srcDir("src/sharedTest/java")
    }
}
```

```kotlin
// MIGRATION OPTION A — extract a :core-testing sub-module
// :core-testing/build.gradle.kts
plugins { id("java-library") }
// classes here are consumed by both test sourcesets in dependent modules

// :feature:home/build.gradle.kts
dependencies {
    testImplementation(project(":core-testing"))
    androidTestImplementation(project(":core-testing"))
}
```

```kotlin
// MIGRATION OPTION B — testFixtures
android {
    testFixtures { enable = true }
}
// src/testFixtures/java/...    contains FakeUserRepository, MainDispatcherRule

// dependent module
dependencies {
    testImplementation(testFixtures(project(":core")))
    androidTestImplementation(testFixtures(project(":core")))
}
```

```kotlin
// MIGRATION OPTION C — KMP source-set naming
// In a KMP Android module:
kotlin {
    androidTarget {
        compilations.getByName("debug") { /* ... */ }
    }
    sourceSets {
        val androidHostTest by getting          // Robolectric, JVM
        val androidDeviceTest by getting        // emulator/device
        val androidCommonTest by getting        // helpers shared by both
        androidHostTest.dependsOn(androidCommonTest)
        androidDeviceTest.dependsOn(androidCommonTest)
    }
}
```

Compose itself uses Option C internally (per CORPUS §B "Source set conventions").

## Module-level Gradle sample

```kotlin
plugins {
    id("com.android.library")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.example.feature.home"
    compileSdk = 35
    defaultConfig {
        minSdk = 23
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }
    testOptions {
        unitTests.isIncludeAndroidResources = true   // required for Robolectric
        animationsDisabled = true                    // hermetic instrumented tests
    }
}

dependencies {
    // production
    implementation("androidx.activity:activity-compose")

    // host (JVM) tests — src/test/
    testImplementation("junit:junit:4.13.2")
    testImplementation("androidx.test:core:1.7.0")
    testImplementation("androidx.test.ext:junit:1.3.0")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.10.0")
    testImplementation("org.mockito:mockito-core:5.13.0")
    testImplementation("org.mockito.kotlin:mockito-kotlin:5.4.0")
    testImplementation("org.robolectric:robolectric:4.13")
    testImplementation("app.cash.turbine:turbine:1.1.0")

    // instrumented (device) tests — src/androidTest/
    androidTestImplementation("androidx.test.ext:junit:1.3.0")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.7.0")
    androidTestImplementation("androidx.compose.ui:ui-test-junit4")

    // debug-only — merged manifest entries for test scaffolds
    debugImplementation("androidx.compose.ui:ui-test-manifest")
    debugImplementation("androidx.fragment:fragment-testing-manifest:1.8.5")

    // orchestrator (note: androidTestUtil, not androidTestImplementation)
    androidTestUtil("androidx.test:orchestrator:1.6.1")
}
```

## Mandatory rules

- **MUST** put pure Kotlin / JVM tests in `src/test/` with `testImplementation` deps.
- **MUST** put Robolectric tests in `src/test/`. Robolectric is a JVM Android simulator, not an instrumented runner.
- **MUST** put real-device / emulator tests in `src/androidTest/` with `androidTestImplementation` deps.
- **MUST** put `androidx.compose.ui:ui-test-manifest` and `androidx.fragment:fragment-testing-manifest` on `debugImplementation`, never `androidTestImplementation`.
- **MUST NOT** cite `developer.android.com/training/testing/...` as the source for `src/sharedTest/`. It is not documented there. Cite the Architecture Blueprints repo or `tasks/research/R8-android-fundamentals.md`.
- **MUST NOT** add Espresso, UiAutomator, or `ui-test-junit4` to `testImplementation`. They will compile but fail at runtime when no `Instrumentation` is present.
- **PREFERRED:** for shared test helpers, extract a `:core-testing` Gradle sub-module or use `android.testFixtures.enable = true` rather than the AGP-7.2-fragile `sharedTest` source set.
- **PREFERRED:** in KMP modules, follow Compose's lead — `androidCommonTest` for helpers, `androidHostTest` for Robolectric, `androidDeviceTest` for emulator tests.
- **PREFERRED:** name backtick test methods freely in `src/test/`; restrict to API 30+ devices in `src/androidTest/` per `/training/testing/instrumented-tests`.

## Verification

- [ ] No Robolectric test exists under `src/androidTest/`. Run `find src/androidTest -name '*.kt' -exec grep -l 'org.robolectric' {} +`; expect empty output.
- [ ] No Espresso / UiAutomator import exists under `src/test/`. Run `find src/test -name '*.kt' -exec grep -l 'androidx.test.espresso\|androidx.test.uiautomator' {} +`; expect empty output.
- [ ] `androidx.compose.ui:ui-test-manifest` is declared exactly on `debugImplementation`. Run `./gradlew :<module>:lint`; expect no `TestManifestGradleConfiguration` warning.
- [ ] `./gradlew :<module>:testDebugUnitTest` runs JVM tests without invoking `connectedDebugAndroidTest`.
- [ ] `./gradlew :<module>:connectedDebugAndroidTest` invokes only tests under `src/androidTest/` (verify with `--info` output).
- [ ] If shared helpers exist, they live in a `:core-testing` sub-module, in `testFixtures`, or in `androidCommonTest` — never in a literal `src/sharedTest/` folder wired via `sourceSets { ... srcDir(...) }`.
- [ ] `testOptions.unitTests.isIncludeAndroidResources = true` is set when any host test references `R.*`.

## References

- `developer.android.com/training/testing/local-tests` — `src/test/`, `testImplementation`, "local test runs directly on your own workstation".
- `developer.android.com/training/testing/instrumented-tests` — `src/androidTest/`, `androidTestImplementation`, AndroidJUnitRunner, backtick caveat.
- `developer.android.com/training/testing/fundamentals` — "Big local test: You can use an Android simulator that runs locally, such as Robolectric".
- `tasks/research/R8-android-fundamentals.md` — verbatim quotes; documents that `src/sharedTest/` is NOT on any of the six in-scope `/training/testing/...` pages.
- Architecture Blueprints (origin of the `sharedTest` pattern): https://github.com/android/architecture-samples
- AGP source-set documentation: https://developer.android.com/build/build-variants#sourcesets
- `testFixtures` flag: https://developer.android.com/build/dependencies#test-fixtures
- CORPUS §C "Source set conventions (KMP)" — Compose's internal `androidHostTest` / `androidDeviceTest` layout.
- `compose/ui/ui-test/src/androidHostTest/kotlin/androidx/compose/ui/test/RobolectricComposeTest.kt` — canonical host-test skeleton in androidx.
- `compose/material3/material3/src/androidDeviceTest/.../SwitchTest.kt` — canonical device-test skeleton.
- Sibling skills: `../../concepts/understanding-the-testing-pyramid/SKILL.md`, `../../concepts/choosing-what-to-test/SKILL.md`, `../../doubles/picking-test-doubles/SKILL.md`, `../applying-testing-strategies/SKILL.md`.
- Cross-category: `../../../compose/setup/configuring-test-dependencies/SKILL.md`, `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`, `../../../jvm-tests/robolectric/using-robolectric-correctly/SKILL.md`, `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`, `../../../adb/tests/running-instrumented-tests-via-adb/SKILL.md`.
