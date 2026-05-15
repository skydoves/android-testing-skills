---
name: running-tests-on-gradle-managed-devices
description: Use this skill to run instrumented Android tests on Gradle Managed Devices (GMD) — emulators that Gradle provisions, boots, runs tests on, and tears down, so CI and every developer get the same device without managing an emulator by hand. Covers the `android.testOptions.managedDevices { localDevices { create(...) { device; apiLevel; systemImageSource } } }` DSL, `ManagedVirtualDevice`, device `groups`, the generated tasks (`<deviceName>DebugAndroidTest`, `<groupName>GroupDebugAndroidTest`, `allDevicesCheck`), Automated Test Device (ATD) images (`aosp-atd`/`google-atd`) for lighter headless CI, test sharding across managed-device copies, where the HTML report and extra outputs land, and how this differs from `connectedAndroidTest` and `adb shell am instrument`. Use when the user mentions "Gradle Managed Devices", "managedDevices", "allDevicesCheck", "ManagedVirtualDevice", "pixel2api30DebugAndroidTest", "ATD image", "automated test device", or "reproducible emulator for tests".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - instrumented-tests
  - gradle-managed-devices
  - managedDevices
  - allDevicesCheck
  - ManagedVirtualDevice
  - atd-image
  - ci-cd
  - test-sharding
  - emulator
---

# Running Tests On Gradle Managed Devices — Gradle Owns The Emulator

Gradle Managed Devices (GMD) move the emulator into the build: you declare devices in `build.gradle.kts`, and `./gradlew <device>DebugAndroidTest` downloads the system image, boots a fresh emulator, runs `androidTest`, collects results, and shuts it down. The payoff is reproducibility — CI and every machine run the exact same device — and no hand-managed emulators. This skill covers the DSL, the generated tasks, ATD images for cheap CI, sharding, and where GMD sits relative to `connectedAndroidTest` and raw `am instrument` (see `../../../adb/tests/running-instrumented-tests-via-adb/SKILL.md`).

## When to use this skill

- The user wants instrumented tests to run on a defined, reproducible emulator in CI without `adb`-managing a device.
- The user mentions `managedDevices`, `allDevicesCheck`, `ManagedVirtualDevice`, or a generated task like `pixel2api30DebugAndroidTest`.
- The user wants the test matrix (which API levels / form factors) to live in version control, not in someone's local AVD list.
- CI emulator runs are slow/flaky and the user asks about ATD ("Automated Test Device") images.
- The user wants to shard a slow instrumented suite across several emulator copies.

## When NOT to use this skill

- The user wants to run tests on a physical device or an emulator they already have running — that is `./gradlew connectedAndroidTest` / `connectedDebugAndroidTest`; GMD is for emulators Gradle creates.
- The user is invoking the runner directly without Gradle (`adb shell am instrument -w -r …`, sharding via `-e numShards`, Test Orchestrator wiring) — use `../../../adb/tests/running-instrumented-tests-via-adb/SKILL.md` and `../../../adb/automation/scripting-adb-for-ci/SKILL.md`.
- The user is choosing the `AndroidJUnit4` runner / writing the test class itself — use `../../runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.
- The user wants Compose UI tests specifically — those still run as instrumented tests; GMD just hosts them. See `../../../compose/setup/setting-up-host-vs-device-tests/SKILL.md` for host-vs-device choice first.

## Prerequisites

- Android Gradle Plugin with GMD support (the stable `managedDevices` DSL; older AGP exposed parts of it under `android.testOptions.managedDevices` experimentally — check the docs for your AGP version).
- `androidTest` set up normally: `testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"`, `androidTestImplementation` dependencies. GMD changes *where* tests run, not how they are written.
- Disk space and a working KVM/HAXM/hypervisor on the build machine (CI runners need KVM enabled) — GMD launches a real emulator.
- License acceptance for the system images GMD downloads (`sdkmanager --licenses`, or accepted in CI setup).

## Workflow

- [ ] **1. Declare the managed devices.** In the module's `build.gradle.kts`:

```kotlin
android {
    testOptions {
        managedDevices {
            localDevices {
                create("pixel6api34") {
                    device = "Pixel 6"          // a Device Manager profile name
                    apiLevel = 34
                    systemImageSource = "aosp"  // "aosp" | "google" | "google_apis_playstore" | "aosp-atd" | "google-atd"
                }
                create("pixel2api30") {
                    device = "Pixel 2"
                    apiLevel = 30
                    systemImageSource = "aosp-atd"  // ATD: stripped-down, headless, faster — good for CI
                }
            }
            groups {
                create("ciMatrix") {
                    targetDevices.add(localDevices["pixel6api34"])
                    targetDevices.add(localDevices["pixel2api30"])
                }
            }
        }
    }
}
```

  Each `create("name") { … }` is a `ManagedVirtualDevice`: `device` is a Device Manager profile, `apiLevel` the system image API, `systemImageSource` which image family. Use `require64Bit = true` if you need to force the 64-bit image. A `group` bundles devices so one task runs the suite across all of them.

- [ ] **2. Run the generated tasks.** GMD synthesizes a task per device, per group, and an all-devices task:

```bash
./gradlew pixel6api34DebugAndroidTest          # androidTest on one managed device
./gradlew ciMatrixGroupDebugAndroidTest        # androidTest on every device in the "ciMatrix" group
./gradlew allDevicesCheck                      # androidTest on ALL managed devices defined in the project
```

  Variant naming follows the build variant (`…DebugAndroidTest`, `…ReleaseAndroidTest`, flavor-prefixed if you have flavors). These tasks: download the system image if missing, boot a fresh emulator, install the app + test APKs, run the suite, write results, and tear the emulator down — no `adb` choreography from you.

- [ ] **3. Read the results.** Per-device HTML reports land under `app/build/reports/androidTests/managedDevice/<deviceName>/` (and an aggregated report when running a group / `allDevicesCheck`); machine-readable results under `app/build/outputs/androidTest-results/managedDevice/`; anything your tests route through `TestStorage` / additional test output under `app/build/outputs/managed_device_android_test_additional_output/<deviceName>/`. Wire those paths into the CI artifact archive.

- [ ] **4. Prefer ATD images for CI.** Automated Test Device images (`systemImageSource = "aosp-atd"` or `"google-atd"`) are pared down for headless test execution — no setup wizard, no UI niceties, smaller, faster to boot, lower memory. They keep the Google APIs you usually need for tests (the `google-atd` variant) without the Play Store. Use a full image only when a test genuinely needs Play services / Play Store behavior.

- [ ] **5. Shard a slow suite across emulator copies.** GMD can run the suite on N copies of a managed device in parallel, splitting tests across them, via the documented Gradle property (e.g. `-Pandroid.experimental.androidTest.numManagedDeviceShards=N`) and `--max-concurrent-shards` to cap how many run at once. Distribution is hash-bucketed by test name, same as `am instrument -e numShards`. Check the GMD docs for the exact property name on your AGP version before relying on it.

- [ ] **6. CI wiring.** A GMD task is a normal Gradle task; CI just needs KVM and accepted licenses:

```yaml
# .github/workflows/instrumented-tests.yml
jobs:
  androidTest:
    runs-on: ubuntu-latest
    timeout-minutes: 45
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: zulu, java-version: 17 }
      - name: Enable KVM
        run: |
          echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' | sudo tee /etc/udev/rules.d/99-kvm4all.rules
          sudo udevadm control --reload-rules && sudo udevadm trigger --name-match=kvm
      - run: ./gradlew pixel2api30DebugAndroidTest   # ATD device declared above
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: androidTest-report
          path: |
            app/build/reports/androidTests/managedDevice/
            app/build/outputs/managed_device_android_test_additional_output/
```

  The emulator is headless by default. (If you need to watch it locally — debugging a UI test — see the GMD docs for the option to show the emulator window; it is off in CI.)

## Patterns

### Pattern: using `connectedAndroidTest` in CI and managing the emulator by hand

```yaml
# WRONG — boot an emulator with adb/avdmanager scripting, then connectedAndroidTest
- run: |
    avdmanager create avd -n ci -k 'system-images;android-30;default;x86_64'
    emulator -avd ci -no-window -no-snapshot &
    adb wait-for-device shell 'while [[ -z $(getprop sys.boot_completed) ]]; do sleep 1; done'
    ./gradlew connectedDebugAndroidTest
# WRONG because: every repo reinvents this, the device definition is not in version control, and
# "works on my machine" diverges from CI. GMD makes the device a build input, provisioned identically everywhere.
```

```kotlin
// RIGHT — declare it once in build.gradle.kts; CI just runs the task
android { testOptions { managedDevices { localDevices { create("ciDevice") {
    device = "Pixel 2"; apiLevel = 30; systemImageSource = "aosp-atd"
} } } } }
// CI:  ./gradlew ciDeviceDebugAndroidTest
```

### Pattern: full system image in CI when ATD would do

```kotlin
// WRONG
create("ciDevice") { device = "Pixel 6"; apiLevel = 34; systemImageSource = "google_apis_playstore" }
// WRONG because: the Play Store image is the heaviest one — slow to download, slow to boot, more RAM —
// and the test suite does not exercise Play Store behavior. CI minutes burn for nothing.
```

```kotlin
// RIGHT — ATD: headless, stripped, keeps the Google APIs tests usually need
create("ciDevice") { device = "Pixel 6"; apiLevel = 34; systemImageSource = "google-atd" }
```

### Pattern: expecting GMD task `$?` to be the only signal

```bash
# WRONG — assume a green exit code means everything passed and stop there
./gradlew allDevicesCheck && echo "all good"
# WRONG because: Gradle does fail the build on test failures here (unlike raw `am instrument`), but a
# device that fails to provision, an OOM-killed emulator, or a flaky boot can also fail the task with
# nothing useful on stdout. Always archive app/build/reports/androidTests/managedDevice/ so a failure is diagnosable.
```

```yaml
# RIGHT — keep the reports regardless of outcome
- run: ./gradlew allDevicesCheck
- uses: actions/upload-artifact@v4
  if: always()
  with: { name: androidTest-report, path: app/build/reports/androidTests/managedDevice/ }
```

## Mandatory rules

- **MUST** declare managed devices in `android.testOptions.managedDevices` in version control — the test device matrix is a build input, not a per-machine AVD.
- **MUST** prefer ATD images (`aosp-atd` / `google-atd`) for CI; use a full or Play Store image only when a test needs GMS / Play Store behavior.
- **MUST** enable KVM (or the platform hypervisor) on CI runners and accept SDK licenses before invoking a GMD task — otherwise the emulator never boots.
- **MUST** archive `app/build/reports/androidTests/managedDevice/` (and `…/managed_device_android_test_additional_output/`) on every run, pass or fail, so failures are diagnosable.
- **MUST NOT** hand-script `avdmanager`/`emulator`/`adb wait-for-device` and then run `connectedAndroidTest` when GMD applies — GMD owns provisioning, boot, and teardown.
- **MUST NOT** assume a managed-device task's exit code distinguishes test failures from infra failures; read the HTML report.
- **PREFERRED:** group devices (`groups { create("ciMatrix") { … } }`) and run `ciMatrixGroupDebugAndroidTest` so the matrix is one task; use `allDevicesCheck` for the full sweep.
- **PREFERRED:** shard slow suites with the documented `numManagedDeviceShards` property + `--max-concurrent-shards` rather than splitting the suite manually.

## Verification

- [ ] `./gradlew tasks --all | grep -i AndroidTest` lists the generated `<deviceName>DebugAndroidTest`, `<groupName>GroupDebugAndroidTest`, and `allDevicesCheck` tasks.
- [ ] `./gradlew <deviceName>DebugAndroidTest` boots an emulator, runs `androidTest`, and produces `app/build/reports/androidTests/managedDevice/<deviceName>/index.html`.
- [ ] The device definitions are in `build.gradle.kts` (committed), not relying on a local AVD.
- [ ] CI runs a GMD task with KVM enabled and uploads the managed-device report directory with `if: always()`.
- [ ] CI devices use an ATD image (`*-atd`) unless a specific test documents why it needs a full/Play Store image.

## References

- developer.android.com/studio/test/gradle-managed-devices — the `managedDevices` / `localDevices` DSL, `ManagedVirtualDevice` (`device`, `apiLevel`, `systemImageSource`, `require64Bit`), device `groups`, the generated `<device>…AndroidTest` / `<group>Group…AndroidTest` / `allDevicesCheck` tasks, ATD images, sharding (`numManagedDeviceShards`, `--max-concurrent-shards`), report/output locations, and showing the emulator window.
- developer.android.com/studio/test/advanced-test-setup — `testOptions.animationsDisabled`, Test Orchestrator, sharding context that also applies to GMD runs.
- developer.android.com/training/testing/instrumented-tests — instrumented test fundamentals; GMD is one execution environment for them.
- developer.android.com/tools/adb — `adb` and `am instrument`, for the lower-level alternative when GMD is not in play.
- Sibling skill: `../../runner/running-instrumented-tests-with-androidjunit4/SKILL.md` — the `AndroidJUnit4` runner the tests use, regardless of where they execute.
- Cross-set: `../../../adb/tests/running-instrumented-tests-via-adb/SKILL.md` — running the same tests via `adb shell am instrument -w -r` without Gradle.
- Cross-set: `../../../adb/automation/scripting-adb-for-ci/SKILL.md` — CI bash idioms, sharding via `-e numShards`, Test Orchestrator wiring, capture-on-failure.
- Cross-set: `../../../compose/preview/capturing-preview-screenshots-in-ci/SKILL.md` — uses a managed device (or `android-emulator-runner`) to render `@Preview` screenshots in CI.
