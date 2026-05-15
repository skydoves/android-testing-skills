---
name: running-instrumented-tests-via-adb
description: Use this skill to run instrumented Android tests directly through `adb shell am instrument -w -r` without going through Gradle. Covers the required `-w` (wait — REQUIRED for meaningful exit codes) and `-r` (raw output) flags, the `-e` argument table (`class`, `class#method`, `package`, `size`, `numShards`/`shardIndex`, `debug`, `annotation` / `notAnnotation`, `listener`, `clearPackageData`, `targetInstrumentation`), the canonical runners `AndroidJUnitRunner` and `AndroidTestOrchestrator`, the orchestrator wrapping pattern (target = orchestrator, `-e targetInstrumentation <pkg>/<runner>`), and the output framing (`INSTRUMENTATION_STATUS_CODE` 1=start, 0=ok, -1=error, -2=failure, -3=ignored, -4=assumption-failure; `INSTRUMENTATION_RESULT`; `INSTRUMENTATION_CODE`). Use when the user mentions `am instrument`, `AndroidJUnitRunner`, "run tests from CI without Gradle", "Orchestrator", `clearPackageData`, `targetInstrumentation`, exit codes from `am instrument`, or `INSTRUMENTATION_STATUS_CODE`.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - am-instrument
  - AndroidJUnitRunner
  - AndroidTestOrchestrator
  - clearPackageData
  - targetInstrumentation
  - INSTRUMENTATION_STATUS_CODE
  - test-sharding
  - numShards
  - shardIndex
  - run-listener
---

# Running Instrumented Tests via ADB — `am instrument` without Gradle

`adb shell am instrument` is the underlying command Gradle invokes; running it directly is the right tool for CI scripts that already manage their own APKs, for sharding fan-out across many devices, and for tight-loop debugging of a single test method. The pitfall most CI scripts fall into: omitting `-w`, which makes the exit code meaningless. The second-most-common pitfall: flipping the orchestrator-target relationship.

## When to use this skill

- The user wants a single test method to run from a script: `adb shell am instrument -w -r -e class com.example.MyTest#myMethod ...`.
- The user wants to shard a test suite across N devices using `numShards` / `shardIndex`.
- The user is wiring AndroidX Test Orchestrator with `clearPackageData true` and gets the target/`targetInstrumentation` order confused.
- The user's CI script reports green when tests actually failed because `$?` is `0` even though `INSTRUMENTATION_STATUS_CODE: -2` shows a failure.
- The user wants the on-device runner to wait for a debugger attach before running tests.

## When NOT to use this skill

- The user wants to install or reset the app under test — use `../../apps/installing-and-managing-apps/SKILL.md`.
- The user wants to choose a device, wait for boot, or set up Wi-Fi debugging — use `../../devices/connecting-to-devices/SKILL.md` and `../../devices/connecting-over-wifi/SKILL.md`.
- The user is writing the JUnit4 / `AndroidJUnit4` test class itself — use `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`.
- The user wants Gradle to do the run for them (`./gradlew connectedDebugAndroidTest`) — that path also reads instrumentation runner args from `testInstrumentationRunnerArguments`.

## Prerequisites

- App APK installed (`com.example.app`) and test APK installed (`com.example.app.test`) — see `../../apps/installing-and-managing-apps/SKILL.md`.
- The test APK was built with `-t` allowed (`android:testOnly="true"`).
- For Test Orchestrator: `androidx.test.orchestrator` APK installed (typically via `androidTestUtil("androidx.test:orchestrator:1.6.1")` and `adb install -r androidx.test.orchestrator.apk`).
- Device in `device` state and animations disabled for stable runs (see `docs/CORPUS.md` §I.6 hermetic test setup).

## Workflow

- [ ] **1. Compose the canonical command.** Verbatim from developer.android.com/studio/test/command-line and `am help`:
    ```bash
    adb shell am instrument -w -r \
      [-e <key> <value>] ... \
      <pkg>/<runner>
    ```
    - `-w` "Wait for instrumentation to finish before returning. **Required for test runners.**" Without it, the shell returns immediately and `$?` is meaningless.
    - `-r` "Print raw results (otherwise decode `report_key_streamresult`)." Pair with `-w` for CI-friendly output.

    Common runners:
    - `androidx.test.runner.AndroidJUnitRunner` — the default for AGP `androidTest`.
    - `androidx.test.orchestrator.AndroidTestOrchestrator` — the wrapper that runs each test in its own instrumentation invocation.

    Bare invocation against the AndroidJUnitRunner:
    ```bash
    adb shell am instrument -w -r \
      com.example.app.test/androidx.test.runner.AndroidJUnitRunner
    ```

- [ ] **2. Use the `-e` argument table for selection, sharding, and runner behavior.** The full table understood by AndroidJUnitRunner / AndroidX Test:
    | Key | Value | Meaning |
    |---|---|---|
    | `package` | `<java_package>` | "Fully qualified Java package name. Any test class using this package executes." Takes precedence over `class`. Comma-separate for multiple. |
    | `class` | `<fqcn>` | "Only this test case class executes." |
    | `class` | `<fqcn>#<method>` | "Only this method executes." Hash separator. |
    | `class` | `<fqcn1>,<fqcn2>#m,...` | Comma-separate to combine multiple selectors. |
    | `size` | `small` / `medium` / `large` | "Runs test methods annotated with `@SmallTest`, `@MediumTest`, or `@LargeTest`." |
    | `annotation` | `<fqcn>` | Filter to tests carrying this annotation. |
    | `notAnnotation` | `<fqcn>` | Exclude tests carrying this annotation. |
    | `numShards` | `<N>` | Total shard count for parallel runs. |
    | `shardIndex` | `<i>` | This shard's 0-based index (0 ≤ i < N). |
    | `debug` | `true` | "Runs tests in debug mode." Waits for debugger attach before each test. |
    | `log` | `true` | "Loads and logs all specified tests but doesn't run them." Use to verify filter combinations. |
    | `listener` | `<fqcn>` | Register a `RunListener`. Comma-separate for multiples. |
    | `filter` | `<fqcn>` | Register a custom JUnit `Filter`. |
    | `runnerBuilder` | `<fqcn>` | Custom `RunnerBuilder`. |
    | `disableAnalytics` | `true` | Disable AndroidJUnitRunner usage stats. |
    | `clearPackageData` | `true` | **AndroidX Orchestrator only**: `pm clear` between tests. |
    | `targetInstrumentation` | `<pkg>/<runner>` | **AndroidX Orchestrator only**: the wrapped instrumentation. |
    | `coverage` | `true` | Collect JaCoCo coverage. |
    | `coverageFile` | `<path>` | Override device coverage file location. |

    Recipes:
    ```bash
    # One class.
    adb shell am instrument -w -r \
      -e class com.example.app.LoginTests \
      com.example.app.test/androidx.test.runner.AndroidJUnitRunner

    # One method.
    adb shell am instrument -w -r \
      -e class com.example.app.LoginTests#login_succeeds \
      com.example.app.test/androidx.test.runner.AndroidJUnitRunner

    # @LargeTest only.
    adb shell am instrument -w -r -e size large \
      com.example.app.test/androidx.test.runner.AndroidJUnitRunner

    # Shard 0 of 4 (run on 4 devices in parallel for 4× speedup).
    adb shell am instrument -w -r -e numShards 4 -e shardIndex 0 \
      com.example.app.test/androidx.test.runner.AndroidJUnitRunner

    # Wait for debugger before each test.
    adb shell am instrument -w -e debug true \
      com.example.app.test/androidx.test.runner.AndroidJUnitRunner

    # Custom RunListener.
    adb shell am instrument -w -r \
      -e listener com.example.app.MyRunListener \
      com.example.app.test/androidx.test.runner.AndroidJUnitRunner
    ```

- [ ] **3. Wire AndroidX Test Orchestrator correctly.** The orchestrator runs **each test in its own instrumentation invocation**, isolating crashes and (with `clearPackageData`) wiping app state between tests. To use it:
    - The **target** (last positional arg) is the orchestrator: `androidx.test.orchestrator/.AndroidTestOrchestrator`.
    - The **wrapped runner** is passed via `-e targetInstrumentation <pkg>/<runner>`.

    ```bash
    adb shell am instrument -w -r \
      -e clearPackageData true \
      -e targetInstrumentation com.example.app.test/androidx.test.runner.AndroidJUnitRunner \
      androidx.test.orchestrator/.AndroidTestOrchestrator
    ```

    Common copy-paste mistake is to flip the two — putting the AndroidJUnitRunner as the positional argument and the orchestrator inside `targetInstrumentation`. That fails with confusing errors because the orchestrator is the runner and the AndroidJUnitRunner is the target.

- [ ] **4. Read the output framing.** Verbatim format:
    ```
    INSTRUMENTATION_STATUS: <key>=<value>      # repeated per Bundle entry per status frame
    INSTRUMENTATION_STATUS_CODE: <int>          # one per status frame
    ...
    INSTRUMENTATION_RESULT: <key>=<value>      # repeated per Bundle entry of the final result
    INSTRUMENTATION_CODE: <int>                 # final, exactly once
    ```
    Status code values (per `Instrumentation.REPORT_VALUE_RESULT_*` plus AndroidJUnitRunner additions):
    | Code | Meaning |
    |---|---|
    | `1` | Test started (`REPORT_VALUE_RESULT_START`). |
    | `0` | Test passed (`REPORT_VALUE_RESULT_OK`). |
    | `-1` | Process error / unexpected throw (`REPORT_VALUE_RESULT_ERROR`). |
    | `-2` | Assertion failure (`REPORT_VALUE_RESULT_FAILURE`). |
    | `-3` | Ignored (`@Ignore`) — added by AndroidJUnitRunner. |
    | `-4` | Assumption failure (`org.junit.AssumeViolatedException`) — added by AndroidJUnitRunner. |

    AndroidJUnitRunner adds `-3` and `-4` on top of the framework's four canonical values; pure framework code only emits `1`, `0`, `-1`, `-2` for **per-test** `INSTRUMENTATION_STATUS_CODE`. The final `INSTRUMENTATION_CODE` line is **separate** and uses Android's `Activity.RESULT_*` constants — `-1` (`RESULT_OK`) when the run completes without a runner-level error and `0` (`RESULT_CANCELED`) when the runner itself errored. **Neither this line nor the shell `$?` reliably reports test pass/fail**: AndroidJUnitRunner calls `finish(Activity.RESULT_OK, results)` regardless, and AOSP `frameworks/base/cmds/am/.../Instrument.java` ends `run()` with unconditional `System.exit(0)` — so `$?` is `0` even when tests fail. **CI scripts MUST parse `INSTRUMENTATION_STATUS_CODE: -2` (failure) and `INSTRUMENTATION_STATUS_CODE: -1` (error) lines from stdout** to detect failures. See step 6 for the canonical grep idiom.

    Standard Bundle keys per status frame (from the `Instrumentation` reference):
    - `REPORT_KEY_NAME_CLASS` — current test FQCN.
    - `REPORT_KEY_NAME_TEST` — current test method.
    - `REPORT_KEY_NUM_CURRENT` — 1-based index of the current test.
    - `REPORT_KEY_NUM_TOTAL` — total test count.
    - `REPORT_KEY_STACK` — failure stack trace.
    - `REPORT_KEY_STREAMRESULT` — pretty stream output (decoded by default; pass `-r` to keep raw).

- [ ] **5. Do NOT trust the shell exit code as a pass/fail signal.** With `-w`, `$?` reports `0` on overall pass AND on test failures — `am instrument` ends with `System.exit(0)` and AndroidJUnitRunner's `finish()` reports `RESULT_OK` regardless of test outcome. Without `-w`, the shell returns immediately and `$?` is even less meaningful. **MUST parse stdout for `INSTRUMENTATION_STATUS_CODE: -2` (failure) and `-1` (error) lines** — see step 6. `-w -r` is still required (the wait + raw flags AGP/Gradle and Android Studio invoke with), just not for its exit code.

- [ ] **6. Parse the output for CI.** Two practical options:
    - **Greppable lines** for fast pipelines:
      ```bash
      adb shell am instrument -w -r ... \
        | tee instrument.log
      grep -E '^INSTRUMENTATION_(CODE|STATUS_CODE|RESULT)' instrument.log
      grep '^INSTRUMENTATION_STATUS_CODE: -2' instrument.log && exit 1
      ```
    - **Let Gradle write the XML** at `app/build/outputs/androidTest-results/connected/<variant>/TEST-*.xml` — required when integrating with JUnit XML test reporters (most CI dashboards). Use `./gradlew connectedDebugAndroidTest` for that path. The two approaches are not mutually exclusive; many pipelines run `am instrument` directly for sharded execution, then post-process raw status streams into JUnit XML themselves.

- [ ] **7. Disable animations and reset state for hermetic runs.** Per `docs/CORPUS.md` §I.6:
    ```bash
    adb shell settings put global window_animation_scale 0
    adb shell settings put global transition_animation_scale 0
    adb shell settings put global animator_duration_scale 0
    adb shell pm clear com.example.app           # before each run; orchestrator does this per-test
    ```
    Or via Gradle: `testOptions.animationsDisabled = true` and Test Orchestrator's `-e clearPackageData true`.

- [ ] **8. Be aware of shell exit-code propagation limits.** `adb shell` exit-code propagation is reliable only since API 24 / Platform Tools 24. On older combinations, scripts that depend on `$?` from `adb shell <cmd>` need to capture the value on-device (`echo $? > /sdcard/exit`) and pull it back. For modern devices and Platform Tools 24+, `adb shell <cmd>; echo $?` propagates the device-side exit code as expected — but note that for `am instrument` that code is always `0` regardless of test results (step 5), so propagation is irrelevant to pass/fail detection.

## Patterns

### Pattern: WRONG vs RIGHT — exit code without `-w`

```bash
# WRONG
adb shell am instrument -r \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner
echo $?
# 0
# WRONG because: -w is omitted, so the adb shell returns immediately. $?
# reflects the launch, not the test outcome. Failures in the test run are
# silently lost.
```

```bash
# RIGHT — gate on stdout, NOT on $?
output=$(adb shell am instrument -w -r \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner)
if echo "$output" | grep -qE 'INSTRUMENTATION_STATUS_CODE: -[12]$'; then
  echo "FAILED"; exit 1
fi
# am instrument exits 0 even when tests fail (System.exit(0) in AOSP Instrument.java),
# so the only reliable signal is parsing INSTRUMENTATION_STATUS_CODE: -2 (failure) or -1 (error).
```

### Pattern: WRONG vs RIGHT — orchestrator target/`targetInstrumentation` flip

```bash
# WRONG
adb shell am instrument -w -r \
  -e clearPackageData true \
  -e targetInstrumentation androidx.test.orchestrator/.AndroidTestOrchestrator \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner
# WRONG because: the orchestrator is the runner, not the target. AndroidJUnitRunner
# does not know how to invoke an "orchestrator target", and clearPackageData is
# not a recognised arg of AndroidJUnitRunner — only of the orchestrator.
```

```bash
# RIGHT
adb shell am instrument -w -r \
  -e clearPackageData true \
  -e targetInstrumentation com.example.app.test/androidx.test.runner.AndroidJUnitRunner \
  androidx.test.orchestrator/.AndroidTestOrchestrator
# Orchestrator is the runner (last positional arg).
# AndroidJUnitRunner is the wrapped target (passed via -e targetInstrumentation).
```

### Pattern: WRONG vs RIGHT — sharding across two devices

```bash
# WRONG
adb -s emulator-5554 shell am instrument -w -r -e numShards 2 -e shardIndex 0 \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner &
adb -s emulator-5554 shell am instrument -w -r -e numShards 2 -e shardIndex 1 \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner &
wait
# WRONG because: both shards run on the same device (-s emulator-5554). They
# serialize against each other through am, defeating the parallelism goal of
# sharding. Worse, they may collide on the same app under test.
```

```bash
# RIGHT
adb -s emulator-5554 shell am instrument -w -r -e numShards 2 -e shardIndex 0 \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner &
adb -s emulator-5556 shell am instrument -w -r -e numShards 2 -e shardIndex 1 \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner &
wait
# One shard per device. Each device runs roughly half the suite.
```

### Pattern: WRONG vs RIGHT — running a single method

```bash
# WRONG
adb shell am instrument -w -r -e class com.example.app.LoginTests:login_succeeds \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner
# WRONG because: the separator between class and method is `#`, not `:`.
# AndroidJUnitRunner treats the whole string as a class name and finds nothing.
```

```bash
# RIGHT
adb shell am instrument -w -r -e class com.example.app.LoginTests#login_succeeds \
  com.example.app.test/androidx.test.runner.AndroidJUnitRunner
```

## Mandatory rules

- **MUST** pass `-w` to `am instrument` for any CI/script invocation so the shell waits for the runner to complete (necessary for stdout parsing). **MUST NOT** gate on `$?` — `am instrument` calls `System.exit(0)` regardless of test outcome.
- **MUST** pair `-w` with `-r` for CI consumption — raw output is parseable; the decoded `report_key_streamresult` form is not.
- **MUST** use `<class>#<method>` (hash separator) for single-method selection. Colons or dots fail silently.
- **MUST** put the orchestrator as the **runner** (positional last arg) and the AndroidJUnitRunner as the **target** via `-e targetInstrumentation` — not the other way around.
- **MUST NOT** rely on `am instrument` exit codes when `-w` is omitted.
- **MUST NOT** assume `clearPackageData true` works without Orchestrator — it is an Orchestrator-only argument.
- **PREFERRED:** use Test Orchestrator + `clearPackageData true` for hermetic test isolation in CI; this maps to Gradle's `testOptions.execution = "ANDROIDX_TEST_ORCHESTRATOR"`.
- **PREFERRED:** disable animations (`settings put global *_animation_scale 0`) before the run, or rely on Gradle's `testOptions.animationsDisabled = true`.

## Verification

- [ ] No script gates on `$?` from `am instrument` — it is always `0` (`System.exit(0)` in AOSP `Instrument.java`; `finish(RESULT_OK, …)` in AndroidJUnitRunner). Pass/fail is detected by grepping stdout for `INSTRUMENTATION_STATUS_CODE: -1` (error) / `-2` (failure).
- [ ] Output stream contains exactly one `INSTRUMENTATION_CODE: <int>` line at the end.
- [ ] Per-test frames carry matching `INSTRUMENTATION_STATUS_CODE: 1` (start) followed by `0` (ok) or `-1`/`-2`/`-3`/`-4`.
- [ ] When using Orchestrator, the runner positional arg is `androidx.test.orchestrator/.AndroidTestOrchestrator` and `-e targetInstrumentation` carries the AndroidJUnitRunner FQN.
- [ ] When sharding, `-e numShards N -e shardIndex i` runs on **distinct devices** (`-s` differs per shard).
- [ ] `-e class <fqcn>#<method>` runs exactly one test method (cross-check with `INSTRUMENTATION_STATUS: numtests=1`).
- [ ] Animation scales are `0.0` on the device (`adb shell settings get global window_animation_scale`).

## References

- Run tests from the command line (`am instrument`): https://developer.android.com/studio/test/command-line
- ADB user guide (`am instrument` flag table): https://developer.android.com/tools/adb#am
- AndroidX Test Orchestrator: https://developer.android.com/training/testing/instrumented-tests/androidx-test-libraries/runner
- `Instrumentation` reference (report keys, status codes): https://developer.android.com/reference/android/app/Instrumentation
- Advanced test setup (Gradle Managed Devices, sharding): https://developer.android.com/studio/test/advanced-test-setup
- `tasks/research/A2-adb-shell-commands.md` — full `-e` table, status-code values including AndroidJUnitRunner's `-3`/`-4`, orchestrator wiring, exit-code-only-with-`-w` rule.
- `docs/CORPUS.md` §I.5 (`am instrument` invocation), §I.6 (hermetic test setup), §I.10 (`-w` exit-code rule).
- Sibling skills:
  - High-level architecture: `../../architecture/understanding-adb-architecture/SKILL.md`
  - Connect a device / wait for boot: `../../devices/connecting-to-devices/SKILL.md`
  - Wireless ADB: `../../devices/connecting-over-wifi/SKILL.md`
  - Install / clear / list apps: `../../apps/installing-and-managing-apps/SKILL.md`
- Cross-set neighbours:
  - Run instrumented tests with `AndroidJUnit4`: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`
  - Configure JUnit4 on Android: `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`
  - Source-set strategy: `../../../fundamentals/strategies/organizing-test-source-sets/SKILL.md`
