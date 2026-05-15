---
name: capturing-screenshots-and-screenrecord
description: Use this skill to capture visual artefacts from a device for test failures, golden image generation, QA repro, and demo videos. Covers `adb shell screencap -p` (PNG screenshot), `adb exec-out screencap -p > out.png` (binary-clean stream that avoids CRLF translation on Windows), `adb shell screenrecord` with `--size`, `--bit-rate`, `--time-limit`, `--rotate`, `--bugreport`, `--verbose` flags, the 3-minute hard cap, scoped-storage rules for `/sdcard/` on API 30+, and the JUnit4 TestWatcher capture-on-failure pattern that grabs a screencap plus `logcat -d` on failure. If the user mentions "screenshot device", "screencap PNG", "raw RGBA dump", "screenrecord 3 minute limit", "scrcpy / Vysor streaming", "exec-out vs shell", or "bugreport overlay timestamp", use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - adb
  - screencap
  - screenrecord
  - capture-on-failure
  - test-artifacts
  - exec-out
  - scoped-storage
  - golden-images
---

# Capturing Screenshots and screenrecord — Visual Artefacts For Tests

This skill covers grabbing PNG screenshots and MP4 screen recordings from a device through `adb`, plus the JUnit4 TestWatcher pattern for capturing on test failure. It does not cover Android Studio's screenshot tool, Compose `captureToImage()`, or third-party streaming tools (scrcpy/Vysor).

## When to use this skill

- A flaky CI test needs a screenshot at the moment of failure plus a logcat dump.
- The team generates golden screenshots from a reference device for visual regression.
- QA needs to record a 30 s repro of a bug for a bug report attachment.
- The dev's `adb shell screencap /sdcard/out.png` produces a corrupt file — the missing `-p` flag wrote raw RGBA, not PNG.
- A Windows CI agent receives screenshots with CRLF corruption — the fix is `adb exec-out screencap -p`.

## When NOT to use this skill

- Compose-level screenshot tests using `captureToImage()` / Roborazzi / Paparazzi. Those are unit-test mechanisms and live under the Compose set, not ADB.
- Continuous live mirroring of the device (presentations, demos). Use scrcpy or Vysor — `screenrecord` cannot stream.
- Driving the device first (taps, key events). Use `../../control/injecting-input-and-state/SKILL.md`.
- Pulling files generally (not screenshots). Use `../../transfer/extracting-test-artifacts/SKILL.md`.

## Prerequisites

- `adb get-state` returns `device`. See `../../devices/connecting-to-devices/SKILL.md`.
- For `screenrecord`: device API 19+ (Android 4.4) — `screenrecord` was added in KitKat.
- For `screenrecord` + Wear OS: not supported. The utility refuses (developer.android.com/tools/adb#screenrecord).
- For `/sdcard/` writes on API 30+: scoped-storage rules apply; the `shell` user retains FUSE/sdcardfs access, but app-private writes route through `/sdcard/Android/data/<pkg>/`.

## `screencap` — PNG screenshots

> "The `screencap` command is a shell utility for taking a screenshot of a device display." — developer.android.com/tools/adb#screencap

```
screencap [-h] [-p] [-d <displayId>] [<path-on-device>]
```

| Flag | Meaning |
|---|---|
| (no flag) | Writes raw RGBA framebuffer to file. **Not** a PNG. |
| `-p` | Encode as PNG. Required when streaming to stdout for the host. |
| `-d <displayId>` | Capture a specific logical display (multi-display devices). |
| `-h` | Help. |

Three idiomatic invocations:

```bash
# 1. Save to device, then pull (good when the test continues running).
adb shell screencap -p /sdcard/screen.png
adb pull /sdcard/screen.png ./screen.png

# 2. Stream PNG bytes back to host without an intermediate file.
adb exec-out screencap -p > screen.png

# 3. Multi-display capture: external display + main.
adb shell screencap -p -d 0 /sdcard/main.png
adb shell screencap -p -d 1 /sdcard/external.png
```

### `exec-out` vs `shell` — the binary-clean trap

`adb shell` sets up a PTY by default, which on some hosts (notably Windows) translates `\n` into `\r\n` on the way out. For binary streams (PNG, MP4, tar archives) this corrupts the file. `adb exec-out` is the binary-clean stdout variant.

Recipe — never use `adb shell screencap -p > out.png` for the streaming variant on Windows. Always:

```bash
adb exec-out screencap -p > out.png
```

For files written on the device first and pulled afterwards, the issue does not apply (`adb pull` is binary-clean).

## `screenrecord` — MP4 screen recording

> "The `screenrecord` command is a shell utility for recording the display of devices running Android 4.4 (API level 19) and higher. The utility records screen activity to an MPEG-4 file." — developer.android.com/tools/adb#screenrecord

```
screenrecord [--size WIDTHxHEIGHT] [--bit-rate RATE] [--time-limit TIME]
             [--rotate] [--bugreport] [--verbose] <path-on-device>
```

| Flag | Verbatim meaning (developer.android.com/tools/adb#screenrecord) |
|---|---|
| `--help` | "Display command syntax and options". |
| `--size WxH` | "Set the video size. The default value is the device's native display resolution (if supported), 1280x720 if not." |
| `--bit-rate R` | "Set the video bit rate for the video, in megabits per second. The default value is 20Mbps." |
| `--time-limit T` | "Set the maximum recording time, in seconds. The default and maximum value is 180 (3 minutes)." |
| `--rotate` | "Rotate the output 90 degrees. This feature is experimental." |
| `--bugreport` | Embeds a textual frame info / timestamp overlay used by bug-report attachments. |
| `--verbose` | "Display log information on the command-line screen." |

### Hard limits (verbatim)

- "Audio is not recorded with the video file."
- "Video recording is not available for devices running Wear OS."
- "Some devices might not be able to record at their native display resolution. If you encounter problems with screen recording, try using a lower screen resolution."
- "Rotation of the screen during recording is not supported."
- The default and **maximum** time limit is 180 s. To capture longer scenarios, chain multiple invocations and concatenate post-hoc, or use scrcpy.

### No streaming flag — the limitation

`screenrecord` writes a single MP4 to the device-side path; it does **not** stream to stdout. For live mirroring (presentations, demos), use **scrcpy** or **Vysor** (out of scope for this skill — both bypass `screenrecord` entirely).

### Output paths

| Path | When |
|---|---|
| `/sdcard/run.mp4` | Default. User-visible video. Scoped-storage rules apply on API 30+ but `shell` user has FUSE/sdcardfs access. |
| `/data/local/tmp/run.mp4` | Use when the test app must produce output before the SD card is mounted, or to avoid touching the public store. Always pullable by `shell`. |

### Stopping early

`screenrecord` runs until `--time-limit` elapses or it receives SIGINT. From an interactive shell, Ctrl+C. From a script, send the signal to the on-device process:

```bash
adb shell screenrecord --time-limit 180 /sdcard/run.mp4 &
REC=$!
# … run instrumentation; on failure, send SIGINT to the on-device process …
adb shell pkill -INT screenrecord
wait $REC
adb pull /sdcard/run.mp4 ./run.mp4
```

### `--bugreport` overlay

`--bugreport` adds a frame-info + timestamp overlay to every frame, useful for QA repros where the bug-report-side timestamp must align with logcat. Pair with `adb logcat -v threadtime,year,UTC` to correlate frame numbers and log lines (see `../../observability/extracting-logs-with-logcat/SKILL.md`).

## Capture-on-failure pattern (JUnit4 TestWatcher)

The standard pattern: a `TestRule` or `TestWatcher` fires `failed()` on test failure, runs `screencap` + `logcat -d`, and pulls the artefacts to a per-test directory.

```kotlin
class CaptureOnFailureRule(private val outputDir: File) : TestWatcher() {
    override fun failed(e: Throwable, description: Description) {
        val testName = "${description.className}.${description.methodName}"
        val device = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation())
        val target = File(outputDir, testName).apply { mkdirs() }

        // PNG screenshot
        device.executeShellCommand("screencap -p /sdcard/fail.png")
        device.executeShellCommand("logcat -d -v threadtime").let { log ->
            File(target, "logcat.txt").writeText(log)
        }
        // The test process can pull /sdcard/fail.png via Context.getExternalFilesDir
        // or push through TestStorageService — see ../../transfer/extracting-test-artifacts/SKILL.md.
    }
}
```

Host-side equivalent in a CI script (see `../../automation/scripting-adb-for-ci/SKILL.md` for the full version):

```bash
set +e
./gradlew connectedDebugAndroidTest
RC=$?
set -e

if [ "$RC" -ne 0 ]; then
  mkdir -p artifacts
  adb shell screencap -p /sdcard/fail.png
  adb pull /sdcard/fail.png artifacts/fail.png
  adb logcat -d -v threadtime > artifacts/logcat.txt
  adb logcat -d -b crash      -v threadtime > artifacts/crash.txt
fi
exit $RC
```

This captures the device's last frame plus a logcat dump only on failure (saves CI storage on green runs). For richer per-test artefact routing, prefer the AndroidX TestStorageService API — see `../../transfer/extracting-test-artifacts/SKILL.md`.

## Patterns

### Pattern: WRONG — `screencap` without `-p`

```bash
# WRONG
adb shell screencap /sdcard/out.png
file /sdcard/out.png      # → "data" (not a PNG)
# WRONG because: without -p, screencap writes a raw RGBA framebuffer (header + naked pixel
# bytes), not a PNG. Image viewers refuse to open it. The .png extension is misleading.
```

```bash
# RIGHT
adb shell screencap -p /sdcard/out.png
adb pull /sdcard/out.png ./out.png
```

### Pattern: WRONG — `adb shell screencap -p > out.png` on Windows

```bash
# WRONG (on Windows / certain CI hosts)
adb shell screencap -p > out.png
# WRONG because: `adb shell` allocates a PTY and translates LF to CRLF on stdout. For
# binary streams (PNG bytes) this corrupts the file.
```

```bash
# RIGHT (binary-clean stdout)
adb exec-out screencap -p > out.png
```

### Pattern: long capture exceeds 3-minute cap

```bash
# WRONG — assumes a single screenrecord can capture 10 minutes
adb shell screenrecord --time-limit 600 /sdcard/run.mp4
# WRONG because: --time-limit is capped at 180. The utility silently truncates to 180.
```

```bash
# RIGHT — chain
for i in 1 2 3 4; do
  adb shell screenrecord --time-limit 180 /sdcard/run-$i.mp4
done
adb pull /sdcard/run-1.mp4 ./
adb pull /sdcard/run-2.mp4 ./
adb pull /sdcard/run-3.mp4 ./
adb pull /sdcard/run-4.mp4 ./
# Concatenate with ffmpeg if needed:
# ffmpeg -f concat -i list.txt -c copy out.mp4
```

(Or switch to scrcpy for indefinite-duration captures.)

### Pattern: bug-report overlay for QA repros

```bash
adb shell screenrecord --bugreport --time-limit 60 \
  --bit-rate 8000000 --size 720x1280 /sdcard/repro.mp4
# Each frame has a frame-info + timestamp overlay aligning with logcat -v threadtime,year,UTC.
adb pull /sdcard/repro.mp4 ./repro.mp4
adb logcat -d -v threadtime,year,UTC > repro-logcat.txt
```

## Mandatory rules

- **MUST** pass `-p` to `screencap` for PNG output. Without it, the file is raw RGBA and unusable in image viewers.
- **MUST** prefer `adb exec-out screencap -p > out.png` over `adb shell screencap -p > out.png` for streaming captures, especially on Windows or any CI host where `\n → \r\n` translation may apply.
- **MUST** treat `screenrecord --time-limit` as bounded by 180 s. Anything longer requires chaining or scrcpy.
- **MUST** stop `screenrecord` with `adb shell pkill -INT screenrecord` (not `kill -9` from the host) — SIGKILL leaves the MP4 unfinalised and unplayable.
- **MUST NOT** assume `screenrecord` captures audio — it does not (developer.android.com/tools/adb#screenrecord).
- **MUST NOT** use `screenrecord` on Wear OS — unsupported.
- **PREFERRED:** capture-on-failure only (not on every run). Saves CI storage and points at the failing test directly.
- **PREFERRED:** pair every `screenrecord --bugreport` with a `logcat -v threadtime,year,UTC` dump captured at the same time. The overlay is otherwise hard to correlate.

## Verification

- [ ] `file ./out.png` reports `PNG image data` after `adb shell screencap -p /sdcard/out.png && adb pull /sdcard/out.png`.
- [ ] On Windows CI, screenshots stream via `adb exec-out screencap -p > out.png` (no `\r\n` corruption).
- [ ] CI captures `screencap` and `logcat -d` only when the test exit code is non-zero.
- [ ] No `screenrecord --time-limit <N>` with `N > 180` exists in any script.
- [ ] No `kill -9` on a `screenrecord` process — only `pkill -INT screenrecord` so the MP4 finalises.
- [ ] CI artefact archive contains per-test sub-directories named after `Class.method`.

## References

- developer.android.com/tools/adb#screencap — `screencap` syntax, `-p` flag, exec-out streaming.
- developer.android.com/tools/adb#screenrecord — `screenrecord` syntax, flags, hard limits ("Audio is not recorded", 180 s cap, no Wear OS, no rotation mid-recording).
- developer.android.com/tools/adb#shellcommands — quoting + the `exec-out` vs `shell` distinction.
- developer.android.com/training/testing/instrumented-tests/stability — screenshot/repro hygiene in instrumented tests.
- Research note `tasks/research/A2-adb-shell-commands.md` — full `screencap` / `screenrecord` flag tables.
- Research note `tasks/research/A3-adb-observability-automation.md` — capture-on-failure CI pattern; `exec-out` rationale.
- Sibling skill: `../../architecture/understanding-adb-architecture/SKILL.md` — server / daemon / `adb shell` vs `adb exec-out`.
- Sibling skill: `../../devices/connecting-to-devices/SKILL.md` — `adb get-state`, multi-device targeting.
- Sibling skill: `../../devices/connecting-over-wifi/SKILL.md` — `adb pair` for wireless capture.
- Sibling skill: `../../apps/installing-and-managing-apps/SKILL.md` — installing APKs to capture against.
- Sibling skill: `../../tests/running-instrumented-tests-via-adb/SKILL.md` — `am instrument` invocation.
- Sibling skill: `../../control/injecting-input-and-state/SKILL.md` — driving gestures before capturing.
- Sibling skill: `../../observability/extracting-logs-with-logcat/SKILL.md` — companion logcat capture in capture-on-failure.
- Sibling skill: `../../transfer/extracting-test-artifacts/SKILL.md` — `adb pull` and TestStorageService.
- Sibling skill: `../../automation/scripting-adb-for-ci/SKILL.md` — full CI capture-on-failure pattern.
- Cross-set: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md` — the JUnit4 `TestWatcher` mechanic.
- Cross-set: `../../../instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md` — driving an Activity to the failing state before capture.
- Cross-set: `../../../fundamentals/strategies/applying-testing-strategies/SKILL.md` — when capture-on-failure beats screenshot-every-run.
