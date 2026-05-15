---
name: scripting-adb-for-ci
description: Use this skill to wire `adb` reliably into CI — bash idioms, exit codes, parallel device fan-out with `xargs -P`, port forwarding (`adb forward` LOCAL REMOTE vs `adb reverse` REMOTE LOCAL — opposite argument order, the most common scripting bug), test-runner status codes (`-1` error, `-2` failure, `-3` ignored, `-4` assumption-failure), `am instrument -w -r` with `--num-shards` / `--shard-index`, the `timeout` wrapper (since `adb -t` is transport-id, NOT timeout), retry-on-transient-error with `adb kill-server`, idempotent setup (`pm clear` + animations to 0), `trap` cleanup, capture-on-failure (`screencap` + `logcat -d`), and Test Orchestrator wiring through `androidTestUtil("androidx.test:orchestrator:1.6.1")` (NOT `androidTestImplementation`). If the user mentions "adb forward vs reverse argument order", "am instrument exit code 0 even on failure", "adb timeout flag", "retry adb kill-server", "trap cleanup adb", or "androidTestUtil orchestrator", use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - adb
  - ci
  - bash
  - port-forwarding
  - test-orchestrator
  - parallel-fanout
  - exit-codes
  - retries
  - timeout
---

# Scripting adb For CI — Bash Idioms, Exit Codes, Fan-out

This skill is the connective tissue that turns the other ADB skills into a CI pipeline: exit-code propagation, retries, timeouts, parallel device fan-out, port forwarding, idempotent setup, capture-on-failure, and Test Orchestrator wiring. Each section cross-links to the skill that owns the underlying primitive.

## When to use this skill

- A CI job invokes `adb shell am instrument` and "passes" even when tests fail — it gates on `$?`, but `am instrument` always exits `0`; the real status lives in `INSTRUMENTATION_STATUS_CODE` lines on stdout (and `-w` is needed just to get that stdout at all).
- The team has multiple physical devices on a single agent and wants to run sharded tests in parallel.
- `adb forward` and `adb reverse` are confused — most teams have at least one bug from the opposite argument order.
- Transient `error: device not found` / `error: closed` failures are not retried; the job fails on the first hiccup.
- A script lacks cleanup so leaked port forwards or modified `settings put global ...` survive into the next run.
- The team wires Test Orchestrator with `androidTestImplementation` — the build silently fails to install the orchestrator APK.

## When NOT to use this skill

- The single primitive (logcat, screencap, pull, push, settings, input, am instrument) is not yet understood. Read the relevant sibling skill first.
- The CI itself (Gradle plugin, Firebase Test Lab cloud config). Use FTL's own docs for that layer.
- Compose / instrumentation test mechanics. Use `../../../compose/synchronization/synchronizing-with-idle/SKILL.md` and friends.

## Prerequisites

- Bash (zsh works for most of this; macOS users should `brew install coreutils` for `gtimeout` if relying on `timeout`).
- Platform Tools 24+ on the host. On older toolchains `adb shell` exit codes are unreliable.
- Devices on **API 24+** for reliable shell exit-code propagation. Pre-Nougat use a sentinel `__ADB_RC=$?` in stdout and grep on the host.

## Exit codes — what adb propagates

| Operation | Behavior |
|---|---|
| `adb push` / `adb pull` | `0` on success, non-zero on transport or filesystem failure. Always check. |
| `adb install` / `adb uninstall` | `0` on success. Modern adb returns non-zero on `INSTALL_FAILED_*`, but old versions exit `0` even on package-manager failure — verify by piping stderr. |
| `adb wait-for-device` (and `wait-for-usb-device`, `wait-for-local-recovery`, etc.) | Block until the state is reached, then exit `0`. Useful boot gate: `adb wait-for-device shell getprop sys.boot_completed | grep -q 1`. |
| `adb shell <cmd>` | On platform-tools ≥ 24 **and** device adbd ≥ Android 7.0 (Nougat / API 24), the device-side command's exit code is propagated. Older devices always returned `0` regardless. |
| `adb -t <transport-id>` | Transport ID, **NOT** a timeout. Most common scripting confusion. |
| `am instrument` | **Always exits `0`**, pass or fail — AOSP `Instrument.java` ends `run()` with an unconditional `System.exit(0)` and AndroidJUnitRunner calls `finish(Activity.RESULT_OK, …)` regardless of test outcome. `-w` makes the shell *wait* for the runner so its stdout is complete; without `-w` the shell returns immediately with nothing usable. Either way, gate CI on parsed `INSTRUMENTATION_STATUS_CODE` lines (below), never on `$?`. |

`adb shell` exit-code propagation can be **disabled** with `-x`:

```
shell [-e ESCAPE] [-n] [-Tt] [-x] [COMMAND...]
   -x: disable remote exit codes and stdout/stderr separation
```

You almost never want `-x` in CI. The default is what you want.

### AndroidJUnitRunner status codes

`am instrument -w -r` emits per-test `INSTRUMENTATION_STATUS_CODE: <int>` followed by a final `INSTRUMENTATION_CODE: <int>`:

| Code | Symbolic name | Meaning |
|---|---|---|
| `1` | `REPORT_VALUE_RESULT_START` | Test started |
| `0` | `REPORT_VALUE_RESULT_OK` | Test passed |
| `-1` | `REPORT_VALUE_RESULT_ERROR` | Unexpected error (process death, runner failure) |
| `-2` | `REPORT_VALUE_RESULT_FAILURE` | Assertion failed |
| `-3` | `REPORT_VALUE_RESULT_IGNORED` (AndroidJUnitRunner extension) | `@Ignore`d / filtered out |
| `-4` | `REPORT_VALUE_RESULT_ASSUMPTION_FAILURE` (AndroidJUnitRunner extension) | JUnit `assumeXxx` returned false |

**Don't treat `-3` or `-4` as failures** — they are skips; only `-1` and `-2` are failures. The trailing `INSTRUMENTATION_CODE` line is unrelated: it's an `Activity.RESULT_*` value (`-1` = `RESULT_OK` = no runner-level crash; `0` = `RESULT_CANCELED` = runner died) and, like the shell `$?` (always `0`), it does **not** reflect test pass/fail. Grep stdout for per-test `INSTRUMENTATION_STATUS_CODE: -1` / `-2` (see the WRONG/RIGHT pattern below).

## Port forwarding — `adb forward` vs `adb reverse`

**The argument order is opposite.** This is the most common scripting bug.

```
adb forward [--no-rebind] LOCAL  REMOTE        # host → device.  LOCAL  REMOTE
adb reverse [--no-rebind] REMOTE LOCAL         # device → host.  REMOTE LOCAL
```

| Direction | Use case | Recipe |
|---|---|---|
| `adb forward` | Host needs to drive a device-side service. | `adb forward tcp:9222 localabstract:chrome_devtools_remote` |
| `adb reverse` | Device needs to reach a local-only host service (CI runner, dev laptop). | `adb reverse tcp:8080 tcp:8080` (Metro/local backend) |

Token grammar (from `adb help`):

| Token | Where |
|---|---|
| `tcp:<port>` | Either side. `tcp:0` on LOCAL = pick any free port; chosen port is printed to stdout. |
| `localabstract:<name>` | Linux abstract Unix domain socket. Used by Chrome DevTools / Stetho / Flipper. |
| `localreserved:<name>` | Reserved Unix socket namespace. |
| `localfilesystem:<name>` | Filesystem-backed Unix socket. |
| `jdwp:<pid>` | REMOTE only. Java Debug Wire Protocol for the given device-side pid. |
| `vsock:<CID>:<port>` | REMOTE only. virtio-vsock (emulator). |
| `dev:<char-device>` | Character device passthrough. |
| `acceptfd:<fd>` | LOCAL only — listen on a pre-opened fd. |

Cleanup is the same surface: `--list`, `--remove`, `--remove-all`. Always remove what was added in a `trap` so re-runs on the same agent don't fail with "cannot bind: address already in use".

## Timeouts — wrap with `timeout`, not `adb -t`

`adb -t <transport-id>` is unrelated to timeout. The host has no per-command adb timeout flag. Wrap with the GNU/BSD `timeout` utility:

```bash
timeout 60s adb -s emulator-5554 shell am instrument -w -r ...
echo $?
# 124 = timeout reached (process killed); 137 = had to escalate to SIGKILL.
# 0 just means am instrument exited (it always does) — see the timeout pattern below for pass/fail parsing.
```

On macOS `timeout` is `gtimeout` from `coreutils` (`brew install coreutils`).

For background captures, set a max duration so a stuck job doesn't fill the disk:

```bash
( timeout 600s adb logcat -v threadtime > artifacts/logcat.txt ) &
LOGCAT_PID=$!
trap 'kill $LOGCAT_PID 2>/dev/null' EXIT
```

## Retries on transient errors

The usual offenders: `error: device 'XYZ' not found`, `error: closed`, `error: protocol fault`, `error: device offline`. A `kill-server` + reconnect resolves most.

```bash
retry() {
  local n=0 max=3 delay=2
  until "$@"; do
    n=$((n+1))
    [ "$n" -ge "$max" ] && return 1
    echo "[retry] $* failed (attempt $n); restarting adb"
    adb kill-server; adb start-server
    sleep "$delay"
  done
}

retry adb install -r app.apk
retry adb shell am instrument -w -r ...
```

For `am instrument` retries specifically, also `pm clear` the test pkg first — otherwise transient state leaks across attempts. Note `retry` only re-runs on *transport* errors (the `adb` host binary exits non-zero); a genuine test failure leaves `am instrument` exiting `0`, so it won't be retried — detect those by parsing the output, not by exit code.

## Parallel device fan-out

```bash
adb devices \
  | tail -n +2 \
  | awk '$2=="device"{print $1}' \
  | xargs -I {} -P 4 adb -s {} <command>
```

Key fragments:

- `tail -n +2` skips the `List of devices attached` header.
- `awk '$2=="device"'` filters out `unauthorized`, `offline`, `recovery`, etc.
- `-P 4` runs 4 in parallel; tune to physical USB-hub count.
- `adb -s {}` targets a serial. `$ANDROID_SERIAL` is the env-var equivalent inside the spawned shells.

For tests with shared device-state (logcat clear, app install), serialise the prep then parallelise the run:

```bash
SERIALS=$(adb devices | awk '$2=="device"{print $1}')
echo "$SERIALS" | xargs -I {} -P 0 adb -s {} install -r app-debug.apk
echo "$SERIALS" | xargs -I {} -P 0 adb -s {} install -r app-debug-androidTest.apk

# Distribute shards across devices (one shard per device)
i=0
for S in $SERIALS; do
  adb -s "$S" logcat -c
  adb -s "$S" shell am instrument -w -r \
       -e numShards $(echo "$SERIALS" | wc -l) -e shardIndex "$i" \
       com.example.test/androidx.test.runner.AndroidJUnitRunner \
       > "results-$S.txt" &
  i=$((i+1))
done
wait

# am instrument always exits 0 — derive pass/fail from the captured output:
if grep -qE '^INSTRUMENTATION_STATUS_CODE: -[12]$' results-*.txt; then
  echo "test failures detected"; exit 1
fi
```

(`xargs -P 0` runs as many concurrently as there are inputs.)

### Sharding via `numShards` / `shardIndex`

> "If you need to parallelize the execution of your tests ... use the `-e numShards` option to specify the number of separate shards to create and the `-e shardIndex` option to specify which shard to run." — AndroidJUnitRunner docs.

Distribution is hash-bucketed by test name — deterministic across runs given the same test set. Adding/removing tests reshuffles buckets.

### Per-device artefact paths

Always namespace by serial:

```bash
artifact_dir() { echo "artifacts/${1//[:_]/-}"; }
for S in $(adb devices | awk '$2=="device"{print $1}'); do
  D=$(artifact_dir "$S"); mkdir -p "$D"
  adb -s "$S" logcat -d -v threadtime > "$D/logcat.txt"
  adb -s "$S" shell screencap -p /sdcard/last.png
  adb -s "$S" pull /sdcard/last.png "$D/last.png"
done
```

## Idempotent setup + cleanup with `trap`

Never assume clean state from a previous run. Bake cleanup into setup, and use `trap` to enforce teardown.

### Minimal setup preamble

```bash
adb wait-for-device
adb shell input keyevent 82                         # wake; no-op if already on
adb shell pm clear com.example.app                  # wipe app data + cache
adb shell pm clear com.example.app.test             # wipe test process state
adb shell am force-stop com.example.app

# Hermetic animations
adb shell settings put global window_animation_scale 0
adb shell settings put global transition_animation_scale 0
adb shell settings put global animator_duration_scale 0
```

(See `../../control/injecting-input-and-state/SKILL.md` for the full `pm clear` vs `am force-stop` reasoning.)

### `trap` cleanup

```bash
cleanup() {
  adb shell am force-stop com.example.app                       || true
  adb shell pm clear     com.example.app                        || true
  adb uninstall          com.example.app                        || true
  adb forward --remove tcp:6100                                 || true
  adb reverse --remove-all                                      || true
  adb shell settings put global window_animation_scale 1        || true
  adb shell settings put global transition_animation_scale 1    || true
  adb shell settings put global animator_duration_scale 1       || true
}
trap cleanup EXIT
```

Each step is `|| true` — partial-state cleanup must never mask the original failure exit code.

### Capture-on-failure

```bash
set +e
./gradlew connectedDebugAndroidTest
RC=$?
set -e

if [ "$RC" -ne 0 ]; then
  mkdir -p artifacts
  adb shell screencap -p /sdcard/fail.png
  adb pull /sdcard/fail.png artifacts/fail.png
  adb logcat -d -v threadtime              > artifacts/logcat.txt
  adb logcat -d -b crash    -v threadtime  > artifacts/crash.txt
  adb logcat -d -b events   -v descriptive > artifacts/events.txt
  adb shell dumpsys activity               > artifacts/dumpsys-activity.txt
fi
exit $RC
```

(See `../../capture/capturing-screenshots-and-screenrecord/SKILL.md` and `../../observability/extracting-logs-with-logcat/SKILL.md` for the underlying primitives.)

## Test Orchestrator wiring

> "Android Test Orchestrator collects JUnit tests at the beginning of your test suite run, but it then executes each test separately, in its own instance of `Instrumentation`." — developer.android.com.

Each `@Test` runs in its own `Instrumentation` invocation, so process state, statics, and `Application` singletons are reset between tests.

### Gradle wiring — `androidTestUtil`, NOT `androidTestImplementation`

```gradle
android {
  defaultConfig {
    testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    testInstrumentationRunnerArguments clearPackageData: 'true'
  }
  testOptions {
    execution 'ANDROIDX_TEST_ORCHESTRATOR'
  }
}

dependencies {
  androidTestImplementation 'androidx.test:runner:1.7.0'
  androidTestUtil           'androidx.test:orchestrator:1.6.1'   // NOT androidTestImplementation
}
```

`clearPackageData: 'true'` makes the Orchestrator run `pm clear <pkg>` between tests — the strongest isolation knob.

### Direct ADB invocation (non-Gradle)

```bash
adb install --force-queryable -r path/to/orchestrator-1.6.1.apk
adb install --force-queryable -r path/to/test-services-1.6.0.apk

adb shell 'CLASSPATH=$(pm path androidx.test.services) app_process / \
 androidx.test.services.shellexecutor.ShellMain am instrument -w -e \
 targetInstrumentation com.example.test/androidx.test.runner.AndroidJUnitRunner \
 androidx.test.orchestrator/.AndroidTestOrchestrator'
```

`--force-queryable` is needed on API 30+ so the Orchestrator can `bindService` into the target despite package-visibility rules.

### Tradeoffs

- **Pros:** each test starts from a known-good `Application`; one crash doesn't kill the rest of the run; `pm clear` between tests is real, not "I cleaned up in `@After`".
- **Cons:** per-test process spin-up adds ~1–2 s. A 500-test suite goes from ~5 minutes to ~15+ minutes. Reserve for integration suites where state leakage actually bites.

## Patterns

### Pattern: WRONG — gating CI on `am instrument`'s exit code

```bash
# WRONG
adb shell am instrument -w -r com.example.test/androidx.test.runner.AndroidJUnitRunner
RC=$?
[ "$RC" -eq 0 ] && echo "tests passed" || echo "tests failed (rc=$RC)"
# WRONG because: am instrument calls System.exit(0) (AOSP Instrument.java) and AndroidJUnitRunner's
# finish() reports RESULT_OK regardless. $? is 0 even when tests fail — with OR without -w.
```

```bash
# RIGHT — keep -w (so stdout is complete) and -r (raw, parseable), then parse the stream
adb shell am instrument -w -r com.example.test/androidx.test.runner.AndroidJUnitRunner \
  | tee instrument.log
# -1 = error, -2 = failure; -3 (ignored) and -4 (assumption failure) are skips, not failures.
if grep -qE '^INSTRUMENTATION_STATUS_CODE: -[12]$' instrument.log; then
  echo "tests failed"; exit 1
fi
echo "tests passed"
```

### Pattern: WRONG — confusing `adb forward` and `adb reverse` argument order

```bash
# WRONG — copy/pasted the forward syntax for reverse
adb reverse tcp:8080 tcp:8080      # ambiguous (works because both are tcp:8080)
adb reverse tcp:9000 tcp:8080      # this means: device port 9000 → host port 8080
# Developer expected: host service on 8080 reachable as 9000 on device.
# WRONG because: reverse argument order is REMOTE LOCAL. The device tries to dial 8080 on
# the host, but the dev meant to expose the local 8080 to the device on port 9000.
```

```bash
# RIGHT — adb reverse REMOTE LOCAL
# Expose host service on 8080 to the device as 9000:
adb reverse tcp:9000 tcp:8080
# In the app code: open a connection to localhost:9000 — it terminates at the host's :8080.

# Or, simpler, identical port:
adb reverse tcp:8080 tcp:8080      # device 8080 ⇆ host 8080
```

### Pattern: WRONG — `androidTestImplementation` for the orchestrator APK

```gradle
// WRONG
dependencies {
  androidTestImplementation 'androidx.test:orchestrator:1.6.1'
}
// WRONG because: the orchestrator is a SEPARATE APK, not a library. androidTestImplementation
// merges it into the test APK's classpath, which AGP rejects (or silently ignores). The
// correct configuration is androidTestUtil, which AGP installs as an additional APK.
```

```gradle
// RIGHT
dependencies {
  androidTestImplementation 'androidx.test:runner:1.7.0'
  androidTestUtil           'androidx.test:orchestrator:1.6.1'
}
```

### Pattern: timeout an instrumentation invocation

```bash
timeout 600s adb shell am instrument -w -r \
  com.example.test/androidx.test.runner.AndroidJUnitRunner | tee instrument.log
RC=${PIPESTATUS[0]}   # exit of `timeout`, not `tee`
case "$RC" in
  124) echo "timed out at 600s"; exit 124 ;;
  137) echo "timed out and had to be SIGKILLed"; exit 137 ;;
esac
# RC is 0 whether tests passed or failed (am instrument always exits 0); pass/fail comes from the stream:
if grep -qE '^INSTRUMENTATION_STATUS_CODE: -[12]$' instrument.log; then
  echo "failed"; exit 1
fi
echo "passed"
```

### Pattern: retry-with-server-bounce

```bash
retry adb -s "$SERIAL" install -r app-debug.apk
retry adb -s "$SERIAL" shell am instrument -w -r \
  -e clearPackageData true \
  com.example.test/androidx.test.runner.AndroidJUnitRunner
```

The `retry` helper bounces the adb server on transient failure (see "Retries on transient errors" above).

### Pattern: parallel sharded fan-out with per-device artefacts

```bash
SERIALS=( $(adb devices | awk '$2=="device"{print $1}') )
N=${#SERIALS[@]}
mkdir -p artifacts

pids=()
for i in "${!SERIALS[@]}"; do
  S="${SERIALS[$i]}"
  D="artifacts/${S//[:_]/-}"
  mkdir -p "$D"
  (
    adb -s "$S" logcat -c
    adb -s "$S" shell pm clear com.example.app
    adb -s "$S" shell am instrument -w -r \
         -e numShards "$N" -e shardIndex "$i" \
         com.example.test/androidx.test.runner.AndroidJUnitRunner \
         > "$D/results.txt"
    # am instrument exits 0 regardless — pass/fail is in the captured stream.
    if grep -qE '^INSTRUMENTATION_STATUS_CODE: -[12]$' "$D/results.txt"; then
      adb -s "$S" shell screencap -p /sdcard/fail.png
      adb -s "$S" pull /sdcard/fail.png "$D/fail.png"
      adb -s "$S" logcat -d -v threadtime > "$D/logcat.txt"
      exit 1
    fi
  ) &
  pids+=($!)
done

rc=0
for p in "${pids[@]}"; do wait "$p" || rc=1; done   # fail the job if any shard failed
exit $rc
```

## Mandatory rules

- **MUST** pass `-w -r` to `am instrument` so the shell waits for the runner and emits raw, parseable output. Without `-w` the shell returns before any output is produced.
- **MUST NOT** gate CI on `$?` from `am instrument` — it calls `System.exit(0)` regardless of test outcome (with or without `-w`). Detect failures by grepping stdout for `INSTRUMENTATION_STATUS_CODE: -1` (error) and `-2` (failure).
- **MUST** distinguish `-3` IGNORED and `-4` ASSUMPTION_FAILURE from real failures when parsing `INSTRUMENTATION_STATUS_CODE`. `-1` and `-2` are the failure codes.
- **MUST** remember that `adb forward` is `LOCAL REMOTE` (host first) and `adb reverse` is `REMOTE LOCAL` (device first). The argument orders are **opposite**.
- **MUST** wrap long-running adb commands with `timeout <N>s` (or `gtimeout` on macOS). `adb` has no built-in timeout flag — `adb -t` is transport ID, not timeout.
- **MUST** use `androidTestUtil("androidx.test:orchestrator:1.6.1")`, **not** `androidTestImplementation`. The orchestrator is a separate APK, not a library.
- **MUST** retry transient adb errors (`device not found`, `closed`, `protocol fault`) with a bounded `retry` helper that calls `adb kill-server; adb start-server` between attempts.
- **MUST** install a `trap cleanup EXIT` that removes port forwards (`adb forward --remove-all`, `adb reverse --remove-all`), restores animation scales, and `pm clear`s the app under test. Each step `|| true` so cleanup never masks the test exit code.
- **MUST** `pm clear <pkg>` (not just `am force-stop`) before each test run. See `../../control/injecting-input-and-state/SKILL.md`.
- **MUST NOT** rely on `adb shell` exit-code propagation on devices below API 24. Use a sentinel `__ADB_RC=$?` in stdout for older targets, or pin minSdk for the test infra.
- **MUST NOT** confuse `adb -t <transport-id>` with `timeout`. They share zero semantics.
- **MUST NOT** parallelise without serialising the install / setup phase — concurrent `adb install` to the same device race.
- **PREFERRED:** Test Orchestrator + `clearPackageData: 'true'` for hermetic isolation; pair with `useTestStorageService: 'true'` for routed artefacts (see `../../transfer/extracting-test-artifacts/SKILL.md`).
- **PREFERRED:** namespace artefacts by serial (`artifacts/${SERIAL//[:_]/-}/`) so `archive artifacts/**/*` in CI captures everything cleanly.

## Verification

- [ ] Every `am instrument` invocation in scripts/Gradle has `-w -r`, and no script branches on its `$?` as a pass/fail signal — failures are detected by parsing `INSTRUMENTATION_STATUS_CODE: -1`/`-2` from stdout.
- [ ] `adb forward` and `adb reverse` appear with arguments in the documented order; CI passes a smoke test that the host can reach the device service and vice versa.
- [ ] `androidTestUtil('androidx.test:orchestrator:...')` (not `androidTestImplementation`) — `./gradlew :app:dependencies --configuration androidTestUtil` shows the orchestrator.
- [ ] `trap cleanup EXIT` is installed in the top-level CI script.
- [ ] All long-running adb commands are wrapped with `timeout`.
- [ ] CI run completes 50 iterations without a "leaked port forward" or "previous run's animation scale" flake.
- [ ] Status-code parsing treats `-3` and `-4` as skips, not failures.
- [ ] Per-device artefact directories are namespaced by serial; CI artefact archive shows them in the expected layout.
- [ ] Retry helper bounces `adb kill-server; adb start-server` between attempts on transient errors.

## References

- developer.android.com/tools/adb — `forward` / `reverse` syntax, transport, server.
- developer.android.com/studio/test/command-line — `am instrument -w -r`, `-e numShards` / `-e shardIndex`, AndroidJUnitRunner arguments, output dirs.
- developer.android.com/training/testing/instrumented-tests — Test Orchestrator overview.
- developer.android.com/reference/android/app/Instrumentation — `REPORT_VALUE_RESULT_*` constants (`OK = 0`, `START = 1`, `ERROR = -1`, `FAILURE = -2`).
- firebase.google.com/docs/test-lab/android/command-line — Firebase Test Lab CI exit codes (`0` pass, `10` test failure, `15` indeterminate, `18` incompatible, `20` infra error).
- Research note `tasks/research/A3-adb-observability-automation.md` — full bash idioms, exit-code propagation matrix, port-forward token grammar, parallel fan-out, retries, capture-on-failure, Test Orchestrator wiring.
- Research note `tasks/research/A2-adb-shell-commands.md` — `am instrument` arguments, status codes, AndroidJUnitRunner `-3`/`-4` extensions.
- Sibling skill: `../../architecture/understanding-adb-architecture/SKILL.md` — server / daemon / `adb kill-server`.
- Sibling skill: `../../devices/connecting-to-devices/SKILL.md` — device states, `wait-for-device`.
- Sibling skill: `../../devices/connecting-over-wifi/SKILL.md` — `adb pair` / `adb connect` for headless CI.
- Sibling skill: `../../apps/installing-and-managing-apps/SKILL.md` — `pm install` / `pm uninstall` / `pm clear`.
- Sibling skill: `../../tests/running-instrumented-tests-via-adb/SKILL.md` — `am instrument -w -r` deep dive.
- Sibling skill: `../../control/injecting-input-and-state/SKILL.md` — hermetic preamble (animations to 0, `pm clear`, `am force-stop`).
- Sibling skill: `../../capture/capturing-screenshots-and-screenrecord/SKILL.md` — `screencap` / `screenrecord` capture-on-failure.
- Sibling skill: `../../observability/extracting-logs-with-logcat/SKILL.md` — `logcat -d` capture-on-failure.
- Sibling skill: `../../transfer/extracting-test-artifacts/SKILL.md` — `pull` / `push` / `run-as` / `TestStorage`.
- Cross-set: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md` — runner internals.
- Cross-set: `../../../instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md` — Activity scenarios under instrumentation.
- Cross-set: `../../../fundamentals/strategies/applying-testing-strategies/SKILL.md` — when to invest in CI orchestration vs simpler tooling.
