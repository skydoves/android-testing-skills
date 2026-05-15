---
name: extracting-logs-with-logcat
description: Use this skill to read device logs for test failures, debug, smoke testing, and CI repros. Covers `adb logcat` (stream), `adb logcat -d` (dump and exit), `adb logcat -c` (clear), buffer selection (`-b main|system|crash|events|radio|kernel|all`), priority ladder (V/D/I/W/E/F/S), filter expressions like `MyApp:D *:S`, format flags (`-v threadtime`, `-v json` on Android 11+), `--pid $(adb shell pidof -s pkg)`, time/count filters (`-T '01-01 12:00:00.000'`, `-t 100`), buffer rotation (`-r <kbytes>`, `-n <count>`, `-f <file>`), buffer sizing (`-G`, `-g`), and stripping `Log.d` calls in release builds via R8 `-assumenosideeffects`. If the user mentions "logcat filter only my app", "events buffer am_proc_start", "logcat json format", "grep logcat expensive", "missing logs after restart", "stripping Log.d release", or "logcat -f writes to host or device", use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - adb
  - logcat
  - log-buffers
  - log-filters
  - threadtime
  - log-rotation
  - r8-keep-rules
  - assumenosideeffects
---

# Extracting Logs With logcat — Reading Device Logs

This skill covers `adb logcat` end-to-end: streaming vs dumping, buffer selection, filter expressions, format flags, PID and time filters, file rotation, and the R8 rule that strips `Log.d` from release builds. The companion CI capture-on-failure pattern lives in `../../automation/scripting-adb-for-ci/SKILL.md`.

## When to use this skill

- A test fails on CI; the developer needs to dump the device log into the artefact archive.
- The developer's `adb logcat | grep MyApp` is overwhelming the SSH session — the logcat-side filter is the fix.
- A scenario only reproduces inside `system_server` or `crash` buffers, not the default `main`.
- The developer wants structured logs (`-v json`) for machine ingestion.
- A `Log.d("Sensitive", "...")` call appears to leak in release — the R8 `-assumenosideeffects` rule is missing.
- Logs disappear after the app is killed by Doze/ANR — `--pid` plus a re-resolution loop is needed.

## When NOT to use this skill

- Capturing a screenshot or video. Use `../../capture/capturing-screenshots-and-screenrecord/SKILL.md`.
- Pulling a generic file from the device. Use `../../transfer/extracting-test-artifacts/SKILL.md`.
- The whole script — retries, port forwarding, parallel device fan-out. Use `../../automation/scripting-adb-for-ci/SKILL.md`.
- Driving gestures or settings changes. Use `../../control/injecting-input-and-state/SKILL.md`.

## Prerequisites

- `adb get-state` returns `device`. See `../../devices/connecting-to-devices/SKILL.md`.
- For `-v json`: device API 30+ (Android 11+). Stack with `-v UTC -v year` for unambiguous timestamps.
- For per-PID filtering: `adb shell pidof -s <pkg>` requires API 24+ (toybox `pidof -s`); on older releases use `adb shell ps -A | grep <pkg>`.

## Three fundamental operations

| Command | Behavior |
|---|---|
| `adb logcat` | Stream the device log to stdout; runs until interrupted. |
| `adb logcat -d` | "Dumps the log and exits." Snapshot mode — perfect for CI. |
| `adb logcat -c` | "Clears the log buffer." Run this before reproducing a bug. |

(Verbatim quotes from developer.android.com/tools/logcat.)

Idiomatic CI capture pattern:

```bash
adb logcat -c                           # clear before scenario
./run-scenario.sh                       # reproduce
adb logcat -d > artifacts/log.txt       # dump after
```

## Buffer selection

`-b <name>` selects which kernel/userspace ring buffer is read. Buffers (developer.android.com/tools/logcat):

| Buffer | What it contains |
|---|---|
| `main` | Default app-side buffer. Does NOT contain system/crash. |
| `system` | Framework / system_server messages. |
| `crash` | Tombstones + unhandled-exception output. |
| `events` | Structured/binary system event buffer. Pair with `-v descriptive` to decode tag names. |
| `radio` | Radio/telephony related messages. |
| `kernel` | Kernel buffer. |
| `all` | Every buffer. |
| `default` | Implicit set: `main`, `system`, `crash`. |

Multiple `-b` flags or comma-separated lists both work:

```bash
adb logcat -b radio
adb logcat -b main -b radio -b events
adb logcat -b main,radio,events
```

The `events` buffer is where to look for `am_*`, `wm_*`, `input_focus`, `notification_*` — emitted by the framework for instrumentation, not for human reading. Decode tag names with:

```bash
adb logcat -b events -v descriptive
```

## Priority ladder + filter expressions

Filter specs are space-separated `tag:priority` pairs. `*` matches every tag.

| Letter | Meaning |
|---|---|
| `V` | Verbose |
| `D` | Debug |
| `I` | Info |
| `W` | Warning |
| `E` | Error |
| `F` | Fatal |
| `S` | "Silent (highest priority, nothing is printed)" |

Setting `*:S` *silences everything*, then any preceding `tag:P` re-enables that tag at priority `P` or above. The canonical "show only my app's logs" idiom:

```bash
adb logcat ActivityManager:I MyApp:D *:S
```

> "Suppress all logs except ActivityManager (Info+) and MyApp (Debug+)" — developer.android.com/tools/logcat.

Other staples:

```bash
adb logcat *:W                          # warnings and above, all tags
adb logcat *:E                          # errors and above (very common in CI)
```

zsh/bash will glob-expand `*:S` outside of quotes when there is a file named `S` in cwd — quote when scripting:

```bash
adb logcat "ActivityManager:I MyApp:D *:S"
```

The same filter can be the host default via env var:

```bash
export ANDROID_LOG_TAGS="ActivityManager:I MyApp:D *:S"
```

## Format flags — `-v <format>`

| Format | What you get |
|---|---|
| `brief` | "Displays priority, tag, and PID" |
| `long` | "All metadata fields with blank lines between messages" |
| `process` | "PID only" |
| `raw` | "Raw log message with no metadata" |
| `tag` | "Priority and tag only" |
| `thread` | "Legacy format showing priority, PID, and TID" |
| `threadtime` | DEFAULT. "Date, time, priority, tag, PID, and TID" |
| `time` | "Date, time, priority, tag, and PID" |

Format **modifiers** stack with `-v` (comma-combinable or repeatable):

| Modifier | Effect |
|---|---|
| `color` | Per-priority colour. |
| `descriptive` | Decode event log tag names. |
| `epoch` | Time in seconds since 1970-01-01. |
| `monotonic` | CPU seconds from last boot. |
| `printable` | Escape binary content. |
| `uid` | UID or Android ID of logged process. |
| `usec` | Time with microsecond precision. |
| `UTC` | Time as UTC. |
| `year` | Add year to displayed time. |
| `zone` | Add local time zone. |

```bash
adb logcat -v json -v UTC -v year       # Android 11+; structured stream, unambiguous time
adb logcat -b all -v color -d           # color dump, all buffers
```

Default-line example (developer.android.com/tools/logcat):

```
I/ActivityManager(  585): Starting activity: Intent { action=android.intent.action.MAIN ... }
```

Schema for `brief`: `<priority>/<tag>(<PID>): <message>`. `threadtime` adds date/time/TID.

## Time and PID filters

- `-T '<date> <time>'` — start at the first entry on or after the given timestamp. Format matches `threadtime`, e.g. `'01-01 12:00:00.000'`.
- `-T <count>` — print the last N entries from the buffer, then continue streaming.
- `-t <count>` — print the last N entries, **then exit** (i.e. `-t` is `-T` with auto-quit).
- `--pid <pid>` — emit only entries from a single process. Combine with `pidof`:

```bash
adb logcat --pid=$(adb shell pidof -s com.example.app)
```

`pidof -s` returns a single pid; without `-s` you get a space-separated list which `--pid` will not accept directly. Wrap a re-resolution loop if the app may be killed and restarted (Doze, ANR, crash):

```bash
while true; do
  PID=$(adb shell pidof -s com.example.app 2>/dev/null)
  [ -n "$PID" ] && adb logcat --pid="$PID"
  sleep 1
done
```

## Buffer rotation and persistence

For long captures, persist to a file with rotation:

| Flag | Meaning |
|---|---|
| `-f <file>` | Write to `<file>` instead of stdout. **Subtle:** with `adb logcat -f`, the file is host-side; with `adb shell logcat -f`, the path is device-side. |
| `-r <kbytes>` | Rotate output every N kilobytes. Requires `-f`. |
| `-n <count>` | Keep at most `<count>` rotated files (`logfile`, `logfile.1`, ...). |
| `-G <size>` | Resize the kernel log buffer for the selected `-b` ring (e.g. `1M`, `16M`). Persists for the life of `logd`. |
| `-g` | Print buffer sizes and current usages, then exit. Cheap diagnostics for "why am I missing logs?". |

```bash
adb logcat -G 16M                                     # grow the main buffer
adb logcat -g                                         # inspect sizes
adb logcat -b main -f /data/local/tmp/app.log \
           -r 1024 -n 8                               # 8x ~1MB rotated files (device-side)
adb shell logcat -b main -f /data/local/tmp/app.log \
           -r 1024 -n 8                               # same, but the device runs it
```

The `-G` resize is global to that buffer and persists across `logd` restarts (until reboot). Don't set 16 MB on a memory-constrained device and forget — release memory back with `-G 256K` after the suite if your CI farm is shared.

## Stripping `Log.d` in release — R8 rule

`android.util.Log` calls remain in release APKs unless code-shrinking removes them. The standard idiom in `proguard-rules.pro` (R8/ProGuard):

```pro
-assumenosideeffects class android.util.Log {
    public static *** v(...);
    public static *** d(...);
    public static *** i(...);
    public static boolean isLoggable(java.lang.String, int);
}
```

`-assumenosideeffects` lets R8 treat the listed methods as pure no-ops, so all calls (and their string-builder argument expressions, when constants) are eliminated. Keep `w`, `e`, `wtf` for production diagnostics. With Timber: strip the `BuildConfig.DEBUG`-gated `DebugTree` plant and rely on a `ReleaseTree` that forwards to Crashlytics for warnings/errors.

For Compose specifically, do **not** `-assumenosideeffects` `kotlin.jvm.internal.Intrinsics` — it removes null-checks the compiler relies on.

## Patterns

### Pattern: WRONG — `adb logcat | grep MyApp` for filtering

```bash
# WRONG
adb logcat | grep MyApp
# WRONG because: the device sends every log line over the USB/TCP transport, then the host
# discards 99% via grep. On a busy device this saturates the transport and the grep can lag
# behind real time. Worse, `grep` runs without -F, so a tag that contains regex metacharacters
# (`.`, `(`, `+`) silently mismatches.
```

```bash
# RIGHT — filter at the logcat level
adb logcat MyApp:D *:S
```

The device only emits matching lines; the transport is uncongested.

### Pattern: WRONG — assuming `-T <count>` exits

```bash
# WRONG
adb logcat -T 100 > tail.txt
# Hangs forever — -T <count> prints the last 100 lines THEN CONTINUES STREAMING.
```

```bash
# RIGHT — `-t` is `-T` with auto-exit
adb logcat -t 100 > tail.txt
```

### Pattern: structured logs for machine ingestion

```bash
adb logcat -v json -v UTC -v year -d -b all > run.jsonl
```

Each line is a single JSON object — pipe through `jq`:

```bash
jq -c 'select(.priority == "ERROR")' run.jsonl
```

(`-v json` requires Android 11+ on the device.)

### Pattern: capture-on-failure — clear, run, dump

```bash
adb logcat -c
./run-scenario.sh
RC=$?
if [ "$RC" -ne 0 ]; then
  adb logcat -d -v threadtime               > artifacts/logcat.txt
  adb logcat -d -b crash    -v threadtime   > artifacts/crash.txt
  adb logcat -d -b events   -v descriptive  > artifacts/events.txt
fi
exit "$RC"
```

`-c` clears before the run so the dump captures only the scenario's logs.

### Pattern: PID-bound stream that survives a process restart

```bash
while true; do
  PID=$(adb shell pidof -s com.example.app 2>/dev/null)
  if [ -n "$PID" ]; then
    adb logcat --pid="$PID" -v threadtime
  fi
  sleep 1
done
```

The outer loop re-resolves the PID after Doze/ANR/crash kills and restarts the app.

### Pattern: `-f` host vs device side

```bash
# host-side rotation (file lives on the developer machine)
adb logcat -b main -f ~/logs/app.log -r 1024 -n 8

# device-side rotation (file lives in /data/local/tmp on the device)
adb shell logcat -b main -f /data/local/tmp/app.log -r 1024 -n 8
adb pull /data/local/tmp/app.log ~/logs/app.log     # later
```

## Mandatory rules

- **MUST** filter at the logcat level (e.g. `MyApp:D *:S`), not via host-side `grep`. Filtering on the device avoids saturating the transport.
- **MUST** quote filter strings containing `*` in zsh/bash (e.g. `"MyApp:D *:S"`) — globbing eats the asterisk if a file named `S` exists in cwd.
- **MUST** use `-t <count>` for "tail and exit". `-T <count>` continues streaming after the initial dump.
- **MUST** use `-d` for snapshot dumps in CI. Streaming `adb logcat` requires explicit termination.
- **MUST** call `adb logcat -c` before reproducing a bug whose log dump you want to attach. Otherwise the dump includes irrelevant prior history.
- **MUST** distinguish `adb logcat -f <file>` (host-side path) from `adb shell logcat -f <file>` (device-side path). The shell forms diverge.
- **MUST** add the R8 `-assumenosideeffects` rule for `android.util.Log` `v`/`d`/`i` (and `isLoggable`) when shipping release APKs that previously called `Log.d` with sensitive data. Keep `w`/`e`/`wtf`.
- **MUST NOT** `-assumenosideeffects` `kotlin.jvm.internal.Intrinsics` — that removes null-check semantics the Kotlin compiler relies on.
- **MUST NOT** rely on `-v json` on devices below Android 11 — silently emits an empty stream.
- **PREFERRED:** stack `-v threadtime,year,UTC` (or `-v threadtime -v year -v UTC`) for unambiguous absolute timestamps in CI artefacts.
- **PREFERRED:** run with `-G 16M` for the `main` buffer when capturing slow-burning bugs, and reset to default in teardown.

## Verification

- [ ] No script in `scripts/` or `.github/workflows/` pipes `adb logcat` directly into `grep` for filtering. All filters are passed as logcat args.
- [ ] CI capture-on-failure dumps include at least `main`, `crash`, and `events` buffers.
- [ ] Release APKs do not emit `Log.d` calls (verify via `apkanalyzer dex packages app-release.apk | grep 'android.util.Log'` returning only `e`/`w`/`wtf`).
- [ ] Filter strings containing `*:S` in shell scripts are quoted.
- [ ] All `--pid` invocations use `pidof -s <pkg>` (singular) and re-resolve in a loop if the app may restart.
- [ ] Every long-running `adb logcat` session in CI has a wall-clock budget (`timeout 600s adb logcat ...`) so a stuck session doesn't fill the disk.

## References

- developer.android.com/tools/logcat — buffer table, priority ladder, filter expressions, `-v threadtime`/`json`/`descriptive`, format modifiers, `-T`/`-t`/`--pid`/`-r`/`-n`/`-G`/`-g` flags, line-format schema.
- developer.android.com/tools/adb — `adb logcat` host-side shorthand vs `adb shell logcat`.
- developer.android.com/studio/build/shrink-code — R8 `-assumenosideeffects` and the `android.util.Log` strip rule.
- developer.android.com/reference/android/util/Log — `Log.v`/`d`/`i`/`w`/`e`/`wtf`/`isLoggable` signatures.
- Research note `tasks/research/A3-adb-observability-automation.md` — full logcat surface, `-f` host-vs-device subtlety, R8 keep-rule rationale, capture-on-failure CI pattern.
- Sibling skill: `../../architecture/understanding-adb-architecture/SKILL.md` — server / daemon, transport.
- Sibling skill: `../../devices/connecting-to-devices/SKILL.md` — device states, `wait-for-device`.
- Sibling skill: `../../devices/connecting-over-wifi/SKILL.md` — `adb pair` / `adb connect`.
- Sibling skill: `../../apps/installing-and-managing-apps/SKILL.md` — installing the APK whose logs are being captured.
- Sibling skill: `../../tests/running-instrumented-tests-via-adb/SKILL.md` — `am instrument -w -r` interplay with logcat.
- Sibling skill: `../../control/injecting-input-and-state/SKILL.md` — driving gestures whose effects show up in logcat.
- Sibling skill: `../../capture/capturing-screenshots-and-screenrecord/SKILL.md` — companion screenshot capture.
- Sibling skill: `../../transfer/extracting-test-artifacts/SKILL.md` — pulling rotated log files from the device.
- Sibling skill: `../../automation/scripting-adb-for-ci/SKILL.md` — full CI capture-on-failure / retry / cleanup patterns.
- Cross-set: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md` — instrumentation logs interleaved with logcat.
- Cross-set: `../../../instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md` — Activity lifecycle events surfaced in `events` buffer.
- Cross-set: `../../../fundamentals/strategies/applying-testing-strategies/SKILL.md` — when log capture is the appropriate observability layer.
