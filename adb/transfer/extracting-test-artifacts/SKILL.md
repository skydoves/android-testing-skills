---
name: extracting-test-artifacts
description: Use this skill to move files between host and device with `adb pull` and `adb push`, including the modern `-z` (compression), `-Z` (no compression), `--sync`, and `-a` (preserve attrs) flags. Covers the path-permission rules — `/data/local/tmp/` is freely writable, `/sdcard/` (alias `/storage/emulated/0/`) is shell-writable with scoped-storage rules on API 30+, `/data/data/<pkg>/` requires `run-as <pkg>` on debuggable builds, and `/sdcard/Android/data/<pkg>/files/` is package-owned but pullable. Includes `adb shell run-as <pkg> cat <path>` for text grabs, `adb exec-out run-as <pkg> tar cf - <path> | tar xf -` for binary-clean directory grabs, the `connected_android_test_additional_output/` Gradle output dir, and the `androidx.test:services` `useTestStorageService` flag for the modern artefact-collection API. If the user mentions "adb pull permission denied data data", "run-as", "scoped storage shell", "tar through adb", "test storage service", or "exec-out vs shell for binary", use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - adb
  - adb-pull
  - adb-push
  - run-as
  - scoped-storage
  - test-storage-service
  - exec-out
  - debuggable-build
---

# Extracting Test Artifacts — adb pull / push and run-as

This skill covers moving files between host and device, the path permission rules that decide whether a `pull` succeeds, and the `androidx.test:services` `TestStorage` API for routing per-test artefacts. Capture-on-failure orchestration (when to pull what) lives in `../../automation/scripting-adb-for-ci/SKILL.md`.

## When to use this skill

- A test produces artefacts (screenshots, JSON, captured DBs) that need to land on CI as build outputs.
- `adb pull /data/data/com.example/databases/main.db` fails with "Permission denied"; the developer doesn't know `run-as` exists.
- A binary file (PNG, MP4, SQLite DB) arrives on the host with mangled bytes — the missing `exec-out` is the fix.
- A push of a large APK or test data file is slow; the developer wants the modern compression flags (`-z brotli`).
- The test app writes screenshots to `/sdcard/Android/data/com.example/files/screenshots/` and the developer wants to know whether that's pullable on API 30+ (it is, when the package owns it).
- The team is migrating from "write to /sdcard, pull on the host" to the modern AndroidX `TestStorage` API.

## When NOT to use this skill

- Capturing the screenshot or video that becomes the artefact. Use `../../capture/capturing-screenshots-and-screenrecord/SKILL.md`.
- Capturing logs to extract. Use `../../observability/extracting-logs-with-logcat/SKILL.md`.
- Installing or uninstalling APKs. Use `../../apps/installing-and-managing-apps/SKILL.md`.
- Driving the device (taps, key events). Use `../../control/injecting-input-and-state/SKILL.md`.
- The whole CI orchestration (when to pull, retries, parallelisation). Use `../../automation/scripting-adb-for-ci/SKILL.md`.

## Prerequisites

- `adb get-state` returns `device`. See `../../devices/connecting-to-devices/SKILL.md`.
- For `run-as`: the target APK must be `android:debuggable="true"` in its merged manifest (the default `debug` build variant).
- For `-z brotli`/`zstd`: platform-tools 30+ on the host and adbd ≥ Android 11 on the device.
- For `TestStorage`: `androidx.test:services` and the `-e useTestStorageService true` argument to `am instrument`.

## Symmetric syntax

> "Unlike the `install` command, which only copies an APK file to a specific location, the `pull` and `push` commands let you copy arbitrary directories and files to any location in a device." — developer.android.com/tools/adb

```
adb pull <remote> [<local>]      # device → host (defaults to cwd)
adb push <local>  <remote>       # host   → device
```

## Modern flags (platform-tools ≥ 30)

```
push [--sync] [-z ALGORITHM] [-Z] LOCAL... REMOTE
pull [-a]     [-z ALGORITHM] [-Z] REMOTE... LOCAL
```

| Flag | Meaning |
|---|---|
| `-z <algo>` | Enable compression. `<algo>` ∈ `any` / `none` / `brotli` / `lz4` / `zstd`. `any` lets adbd pick. |
| `-Z` | Disable compression. |
| `--sync` (push only) | Only push files with different timestamps — fast incremental updates. |
| `-a` (pull only) | Preserve file timestamp and mode. |
| `-n` | Dry-run. |
| `-q` | Suppress progress messages. |

The legacy `adb pull -p` (show progress) is now a no-op — modern adb shows progress by default and `-q` silences it. Don't rely on `-p` in CI scripts.

## Path permission rules

This is the load-bearing table. Most "permission denied" `adb pull` failures come from misunderstanding it.

| Path | adb shell writable? | `pull` works? |
|---|---|---|
| `/data/local/tmp/` | Yes | Yes |
| `/sdcard/` (= `/storage/emulated/0/`) | Yes (scoped storage rules on API 30+) | Yes |
| `/data/data/<pkg>/` | Only via `run-as <pkg>` (debuggable builds) | Only via `run-as` |
| `/sdcard/Android/data/<pkg>/files/` | Package-owned, scoped storage | Yes for the owning package, and yes for `adb shell` (FUSE/sdcardfs bridge) |

### `/data/local/tmp/`

Readable/writable by the `shell` user that backs `adb`. No app-permission ceremony required. Use as the **staging directory** for everything: pushed APKs, test data, captured screenshots, JSON inputs, dexed instrumentation jars.

Caveat: your app runs as `u0_aXX`, not `shell`, so the app cannot read shell-written files unless you `chmod 0644` and the parent is world-readable. `/data/local/tmp/` itself is `0771`, so be deliberate when pushing files the app must read.

### `/sdcard/`

Alias `/storage/emulated/0/`, sometimes `/storage/self/primary/`. Public/external storage. ADB's `shell` user retains FUSE/sdcardfs access on API 30+, so `adb push /sdcard/...` and `adb pull /sdcard/...` work for tests even after scoped storage tightening. App-side writes are gated through MediaStore on API 30+.

### `/data/data/<pkg>/` — `run-as` only

Owned by the app's UID; `0700` on the dir itself. A bare `adb pull /data/data/com.example/...` fails on production user builds. Two viable paths:

- **`run-as <pkg>`** on debuggable builds (most common).
- **`adb root`** on `userdebug`/`eng` builds (covered briefly in `../../automation/scripting-adb-for-ci/SKILL.md`).

### `/sdcard/Android/data/<pkg>/files/`

The app's "external private" storage — written from the app via `Context.getExternalFilesDir(null)`. Readable from `adb shell` because it lives under FUSE-emulated SD card. The path of least resistance for "write artefact, pull from CI" on Android 11+:

```bash
# Inside the test app:
val out = File(context.getExternalFilesDir(null), "screenshots/fail.png")
```

```bash
# From CI:
adb pull /sdcard/Android/data/com.example/files/screenshots/fail.png ./
```

## `run-as` on debuggable builds

`run-as <pkg>` is a setuid helper baked into Android since Gingerbread. It drops shell into the app's UID **if and only if** the APK has `android:debuggable="true"` in its merged manifest (the default for the `debug` build variant). Usage: `adb shell run-as <pkg> <cmd>`.

### Text-file grab

```bash
adb shell run-as com.example.app cat databases/main.db > main.db
```

This works for short text files. For binary files it can corrupt bytes via PTY translation (see `exec-out` below).

### Binary-clean grab via `exec-out` + `tar`

For directories, binary files, or anything with `\n` in the bytes, **MUST** use `exec-out` (binary-clean stdout) and pipe through `tar`:

```bash
adb exec-out run-as com.example.app tar cf - databases/ \
  | tar xf - -C ./pulled
```

The `tar cf -` produces an archive on stdout; the local `tar xf -` extracts it into `./pulled/`. No intermediate file, no PTY translation, works for arbitrary binary content.

### Single-file binary grab

```bash
adb exec-out run-as com.example.app cat databases/main.db > main.db
```

`exec-out` keeps stdout binary-clean.

### Listing

```bash
adb shell run-as com.example.app ls -lR files/
```

### Three things `run-as` cannot do

1. Work on release / non-debuggable builds. Will print "Package is not debuggable".
2. Switch to a system UID (only the target package's UID).
3. Cross profile boundaries by itself. Use `--user N` if needed.

## Common test-artefact paths

| Path | Origin |
|---|---|
| `app/build/outputs/connected_android_test_additional_output/<flavor><Variant>/connected/<deviceId>/` | Gradle's standard "additional output" sink. The instrumentation runner pulls anything declared via `androidx.test:runner` arg `additionalTestOutputDir`. |
| `app/build/outputs/androidTest-results/connected/<flavor><Variant>/TEST-*.xml` | JUnit-style XML test results — ingested directly by Jenkins/GHA/etc. |
| `app/build/reports/androidTests/connected/` | HTML test report. |

(See developer.android.com/studio/test/command-line: HTML at `module/build/reports/androidTests/connected/`, XML at `module/build/outputs/androidTest-results/connected/`.)

## Modern artefact API — `TestStorage`

The legacy pattern is "write to `/sdcard/...`, pull from CI script". The modern pattern routes through `androidx.test.services.storage.TestStorage` and a `useTestStorageService` flag.

### Wiring

```kotlin
// dependencies
androidTestImplementation("androidx.test:runner:1.7.0")
androidTestUtil("androidx.test.services:test-services:1.6.0")
```

```kotlin
// am instrument args (or Gradle: testInstrumentationRunnerArguments)
adb shell am instrument -w -r \
  -e useTestStorageService true \
  com.example.test/androidx.test.runner.AndroidJUnitRunner
```

### In-test usage

```kotlin
val storage = androidx.test.platform.io.PlatformTestStorageRegistry.getInstance()
storage.openOutputFile("screenshots/fail-${test.methodName}.png").use { out ->
    out.write(pngBytes)
}
```

### What `TestStorage` solves

- Output files land in a TestStorageService-managed directory rather than `/sdcard/` proper.
- Gradle's `connectedAndroidTest` automatically pulls them into `app/build/outputs/connected_android_test_additional_output/`.
- Per-test artefact namespacing happens through the runner, not via hand-rolled `Class.method` filename mangling.
- Compatible with Test Orchestrator's `clearPackageData: 'true'` — the storage service holds artefacts even though the test app is wiped between tests.

For non-Gradle pipelines, the storage backs onto the same FS layout, addressable as `/sdcard/googletest/test_outputfiles/...` for legacy compatibility.

## Patterns

### Pattern: WRONG — `adb pull /data/data/<pkg>/...` on a debuggable build

```bash
# WRONG
adb pull /data/data/com.example/databases/main.db
# adb: error: failed to stat remote object '/data/data/com.example/databases/main.db': Permission denied
# WRONG because: /data/data/<pkg>/ is owned by the app UID with mode 0700. The shell user
# cannot traverse it. run-as solves this on debuggable builds.
```

```bash
# RIGHT — run-as + exec-out + cat for a binary-clean grab
adb exec-out run-as com.example cat databases/main.db > main.db
file main.db                 # → "SQLite 3.x database"
```

Or for a directory:

```bash
adb exec-out run-as com.example tar cf - databases/ | tar xf - -C ./pulled
```

### Pattern: WRONG — `adb shell run-as <pkg> cat <binary>` for binary files

```bash
# WRONG (on Windows / certain CI hosts)
adb shell run-as com.example cat databases/main.db > main.db
# WRONG because: `adb shell` allocates a PTY and translates LF to CRLF on stdout. SQLite
# headers and other binary content get corrupted. The pulled DB fails to open.
```

```bash
# RIGHT — exec-out is binary-clean
adb exec-out run-as com.example cat databases/main.db > main.db
```

### Pattern: writing artefacts in-app, pulling from CI

```kotlin
// In production / test code
val out = File(context.getExternalFilesDir(null), "captures/${name}.png")
out.outputStream().use { it.write(bitmapBytes) }
```

```bash
# CI script
adb pull /sdcard/Android/data/com.example/files/captures/ ./artifacts/
```

This works on API 30+ without `run-as` because `/sdcard/Android/data/<pkg>/files/` is the package's own external-private dir. Both the app and `adb shell` (FUSE bridge) can read it.

### Pattern: fast incremental push of large test data

```bash
adb push --sync -z brotli ./fixtures/ /sdcard/fixtures/
```

`--sync` skips files with matching timestamps; `-z brotli` compresses the wire bytes. For a fixtures dir with hundreds of unchanged files, the second push transfers nothing.

### Pattern: capture-on-failure with `TestStorage`

```kotlin
class CaptureOnFailureRule : TestWatcher() {
    override fun failed(e: Throwable, description: Description) {
        val storage = PlatformTestStorageRegistry.getInstance()
        val name = "${description.className}.${description.methodName}"
        storage.openOutputFile("captures/$name.png").use { out ->
            out.write(captureScreenshotPng())
        }
    }
}
```

After `connectedAndroidTest`, the artefact lands in `app/build/outputs/connected_android_test_additional_output/<flavor><Variant>/connected/<device>/captures/<name>.png` automatically.

## Mandatory rules

- **MUST** use `adb exec-out run-as <pkg> cat <path>` (not `adb shell run-as ...`) for binary file grabs. PTY translation corrupts binary on Windows and some CI hosts.
- **MUST** use `adb exec-out run-as <pkg> tar cf - <dir> | tar xf -` for directory grabs from `/data/data/<pkg>/`. There is no `adb pull` equivalent that works through `run-as`.
- **MUST** verify the build is debuggable before scripting `run-as`. On a release APK, `run-as` exits non-zero with "Package is not debuggable".
- **MUST** prefer `/sdcard/Android/data/<pkg>/files/` (via `Context.getExternalFilesDir(null)`) over arbitrary `/sdcard/<dir>/` for app-written test artefacts on API 30+. The latter is gated by MediaStore for app writes.
- **MUST** prefer `TestStorage` (`-e useTestStorageService true` + `androidx.test:services`) over hand-rolled `/sdcard/...` paths for new test infrastructure. It namespaces per-test artefacts and integrates with Gradle's `connected_android_test_additional_output/` sink.
- **MUST NOT** rely on `adb pull -p` (show progress) — it is a legacy no-op on platform-tools 30+. Use the default progress meter (or `-q` to silence).
- **MUST NOT** assume `adb pull` of `/data/data/<pkg>/...` works without `run-as`. It does not, even on debuggable builds — `pull` is a `shell`-user operation.
- **PREFERRED:** `--sync -z brotli` for repeated pushes of large fixture sets. Brotli + timestamp-skip is dramatically faster on slow USB 2.0 cables.
- **PREFERRED:** Test Orchestrator + `clearPackageData: 'true'` + `useTestStorageService: 'true'` for hermetic test isolation with retained artefacts.

## Verification

- [ ] No script invokes `adb pull /data/data/<pkg>/...` directly. All such pulls go through `run-as`.
- [ ] Binary-file pulls use `adb exec-out`, not `adb shell`.
- [ ] All app-written test artefacts target `Context.getExternalFilesDir(...)` (lands at `/sdcard/Android/data/<pkg>/files/`), not `/sdcard/<arbitrary>/`.
- [ ] CI archives `app/build/outputs/connected_android_test_additional_output/**/*` for the modern artefact API.
- [ ] Repeated pushes of the same fixture dir use `adb push --sync` so unchanged files are skipped.
- [ ] `file` reports the correct binary type for every pulled artefact (no CRLF corruption).
- [ ] Release APKs have no test code paths that try to `adb pull` from `/data/data/<pkg>/...` — those would fail.

## References

- developer.android.com/tools/adb — `pull`/`push` syntax, "let you copy arbitrary directories and files to any location in a device" quote.
- developer.android.com/studio/test/command-line — Gradle output dir conventions: HTML at `module/build/reports/androidTests/connected/`, XML at `module/build/outputs/androidTest-results/connected/`.
- developer.android.com/training/data-storage/app-specific — `Context.getExternalFilesDir()` and the API-30+ scoped-storage rules.
- developer.android.com/reference/androidx/test/services/storage/TestStorage — the modern artefact API.
- developer.android.com/reference/androidx/test/platform/io/PlatformTestStorageRegistry — host-side handle.
- Research note `tasks/research/A3-adb-observability-automation.md` — full pull/push surface, path permission rules, run-as patterns, scoped storage on Android 11+, TestStorage wiring.
- Sibling skill: `../../architecture/understanding-adb-architecture/SKILL.md` — server / daemon, exec-out vs shell.
- Sibling skill: `../../devices/connecting-to-devices/SKILL.md` — `adb get-state`, multi-device targeting.
- Sibling skill: `../../devices/connecting-over-wifi/SKILL.md` — wireless transport.
- Sibling skill: `../../apps/installing-and-managing-apps/SKILL.md` — `pm install` and the debuggable-build flag.
- Sibling skill: `../../tests/running-instrumented-tests-via-adb/SKILL.md` — `am instrument` arguments including `useTestStorageService`.
- Sibling skill: `../../control/injecting-input-and-state/SKILL.md` — driving the scenario that produces the artefact.
- Sibling skill: `../../capture/capturing-screenshots-and-screenrecord/SKILL.md` — producing PNG / MP4 to pull.
- Sibling skill: `../../observability/extracting-logs-with-logcat/SKILL.md` — pulling rotated log files via `adb shell logcat -f` + `adb pull`.
- Sibling skill: `../../automation/scripting-adb-for-ci/SKILL.md` — full CI orchestration.
- Cross-set: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md` — `TestWatcher` rules wired to `TestStorage`.
- Cross-set: `../../../instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md` — Activity-driven scenarios that emit artefacts.
- Cross-set: `../../../fundamentals/strategies/applying-testing-strategies/SKILL.md` — when artefacts are the right observability layer.
