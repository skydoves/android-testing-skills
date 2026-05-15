---
name: installing-and-managing-apps
description: Use this skill to install, uninstall, list, inspect, and reset Android apps via `adb install` (with `-r` reinstall, `-d` allow-downgrade, `-t` allow test packages, `-g` grant all runtime permissions, `--user` per-user install, `adb install-multiple` for split APKs) and the on-device `pm` tool (`pm list packages [-f|-d|-e|-s|-3|-i|-u]`, `pm path`, `pm clear`, `pm grant` / `pm revoke`, `pm enable` / `pm disable`, `pm uninstall [-k]`, `cmd package dump-profiles`). Calls out the canonical hermetic-reset pattern (`pm clear` wipes data; `am force-stop` does NOT) and the common install errors (`INSTALL_FAILED_USER_RESTRICTED`, `INSTALL_FAILED_VERSION_DOWNGRADE`, signing-conflict). Use when the user mentions "reset app between tests", `pm clear`, `am force-stop` confusion, install fails after debugger, split APK install, runtime permission grants, multi-user profile installs, or "how do I list third-party apps".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - adb-install
  - pm-clear
  - pm-grant
  - pm-list-packages
  - force-stop
  - install-failed-user-restricted
  - install-failed-version-downgrade
  - install-multiple
  - baseline-profile
  - hermetic-test-reset
---

# Installing and Managing Apps — `adb install` and `pm`

App lifecycle from the host. `adb install` ships an APK; `pm` is the on-device PackageManager front-end that lists, paths, clears, enables/disables, and grants permissions. The most common subtle bug in test scaffolding is using `am force-stop` to "reset" an app between tests — that command kills the process but **leaves SharedPreferences, the Room DB, and files on disk**. The hermetic equivalent is `pm clear`.

## When to use this skill

- The user wants to install an APK from the host or push a debug build between every test run.
- The user asks "how do I reset the app's state between tests?" or sees stale data carry over from one test to the next.
- The user hits `INSTALL_FAILED_VERSION_DOWNGRADE`, `INSTALL_FAILED_USER_RESTRICTED`, or signature mismatch errors.
- The user wants to grant all runtime permissions at install (`-g`), or grant/revoke a single permission post-install.
- The user wants to list only third-party packages, find an APK on disk, or dump a Baseline Profile.

## When NOT to use this skill

- The user wants to actually run instrumented tests against the installed app — use `../../tests/running-instrumented-tests-via-adb/SKILL.md`.
- The user wants to choose a device or wait for boot — use `../../devices/connecting-to-devices/SKILL.md`.
- The user wants the architecture / environment / authentication picture — use `../../architecture/understanding-adb-architecture/SKILL.md`.

## Prerequisites

- A connected device with state `device` (see `../../devices/connecting-to-devices/SKILL.md`).
- The APK(s) signed with a debug or release key the developer controls.
- For test APKs, the manifest sets `android:testOnly="true"` (Gradle does this for `androidTest` builds) and install requires `-t`.
- For runtime permissions, the device runs API 23+.

## Workflow

- [ ] **1. Install with the right flags.** From `pm install` (and the `adb install` wrapper) options table:
    | Flag | Meaning |
    |---|---|
    | `-r` | "Reinstall an existing app, keeping its data." **Required** when the package is already installed; without it `adb install` returns `INSTALL_FAILED_ALREADY_EXISTS`. CI scripts MUST always pass `-r` for repeat runs. |
    | `-d` | "Allow version code downgrade." Required when the new APK has a lower `versionCode`. Note: per `adb help`, restricted to debuggable packages on stock builds. |
    | `-t` | "Allow test APKs to be installed." Required for any APK whose manifest carries `android:testOnly="true"` (i.e. all Gradle `androidTest` builds). |
    | `-g` | "Grant all permissions listed in the app manifest." Best for hermetic test runs. |
    | `-l` | Forward-lock the application (legacy DRM). Almost never needed today. |
    | `-s` | Install on the SD card / external storage (when device supports it). |
    | `--user <id>` | Install for a specific user profile. Default: all users. `current` resolves to the foreground user. |
    | `--install-location 0|1|2` | 0=default, 1=internal, 2=external. |
    | `-f` | Install on internal system memory. |
    | `-i <installer_pkg>` | Tag the install with an installer package name. |
    | `--fastdeploy` | Patch-only install (only changed parts of the APK). |
    | `--incremental` / `--no-incremental` | Streamed install; requires APK Signature Scheme v4 sidecar `.idsig`. `--wait` blocks until full APK is available. |

    Common recipes:
    ```bash
    adb install -r app-debug.apk                       # ordinary reinstall
    adb install -r -d app-debug.apk                    # allow downgrade
    adb install -r -t -g app-androidTest.apk           # test APK with all permissions granted
    adb install -r -d --user current app-debug.apk     # force foreground-user install
    ```

- [ ] **2. For split APKs use `adb install-multiple`.** A modern App Bundle produces a base APK plus per-feature/per-config splits — `adb install` of just one fails:
    ```bash
    adb install-multiple -r -t -g \
      base-master.apk base-arm64_v8a.apk base-xxhdpi.apk
    ```

- [ ] **3. Reset app state between tests with `pm clear`. Do NOT use `am force-stop` for hermetic isolation.**
    - `am force-stop <pkg>` "Force-stop everything associated with `package`." — kills processes only. SharedPreferences, Room/SQLite databases, files in `/data/data/<pkg>`, and cache **persist**.
    - `pm clear <pkg>` "Delete all data associated with a package." — wipes user data and cache. The next launch starts from a fresh-install state.
    For genuine hermetic isolation between tests (the default for instrumented suites), use `pm clear`. `am force-stop` is appropriate only when the explicit goal is to kill the process while preserving state (e.g. testing a "resume from killed" path).

    ```bash
    adb shell pm clear com.example.app
    ```

    Most pipelines wire this up via Test Orchestrator's `clearPackageData` instrumentation arg — see `../../tests/running-instrumented-tests-via-adb/SKILL.md`.

- [ ] **4. List, find, and inspect packages with `pm list packages` and `pm path`.**
    ```
    pm list packages [-f] [-d] [-e] [-s] [-3] [-i] [-u] [--user <id>] [<filter>]
    ```
    | Flag | Meaning |
    |---|---|
    | `-f` | "See associated file." |
    | `-d` | "Filter to only show disabled packages." |
    | `-e` | "Filter to only show enabled packages." |
    | `-s` | "Filter to only show system packages." |
    | `-3` | "Filter to only show third-party packages." |
    | `-i` | "See the installer for the packages." |
    | `-u` | "Include uninstalled packages." |
    | `--user <id>` | "The user space to query." |

    Recipes:
    ```bash
    adb shell pm list packages -3 | sort                   # all sideloaded apps
    adb shell pm list packages -f com.example              # APK paths for matching pkgs
    adb shell pm path com.example.app                      # package:/data/app/.../base.apk
    adb shell dumpsys package com.example.app | head -200  # detailed install/permission info
    ```

- [ ] **5. Manage runtime permissions with `pm grant` / `pm revoke` (API 23+).**
    > "Grant a permission to an app. On devices running Android 6.0 (API level 23) and higher, the permission can be any permission declared in the app manifest."

    ```bash
    adb shell pm grant  com.example.app android.permission.CAMERA
    adb shell pm revoke com.example.app android.permission.CAMERA
    adb shell pm reset-permissions                         # reset every app to defaults
    ```

    Flagging `-g` at install time grants every manifest-declared runtime permission in one shot — preferred for hermetic test runs.

- [ ] **6. Enable / disable packages or specific components.**
    ```bash
    adb shell pm disable com.example.app                       # whole package
    adb shell pm disable-user --user 0 com.example.app         # per-user disable (preserves data)
    adb shell pm enable  com.example.app
    adb shell pm enable  com.example.app/.MyTestActivity       # specific component
    ```
    Useful for forcing a known accessibility-service / IME state in tests, or for staging A/B comparisons by disabling rivals.

- [ ] **7. Uninstall correctly.**
    ```
    pm uninstall [-k] [--user <id>] [--versionCode <vc>] <package>
    ```
    - `-k` "Keep the data and cache directories after package removal." Useful for upgrade-from-clean tests where the user data simulates an existing install.
    - `--user <id>` per-user removal.
    - `--versionCode <vc>` "Only uninstalls if the app has the given version code." Idempotent uninstall in CI scripts.

    ```bash
    adb uninstall com.example.app                       # equivalent to: pm uninstall <pkg>
    adb shell pm uninstall -k com.example.app           # uninstall but preserve data
    adb shell pm uninstall --user 0 com.example.app     # only from primary user
    ```

- [ ] **8. Dump a Baseline Profile when needed.**
    > "Starting in Android 7.0 (API level 24), the Android Runtime (ART) collects execution profiles for installed apps... `adb shell cmd package dump-profiles <package>` then `adb pull /data/misc/profman/<package>.prof.txt`."
    The doc warns: "It is only possible to retrieve the execution profile filename if you have root access to the file system, for example, on an emulator." On user builds without root, use `androidx.benchmark` Macrobenchmark to extract the profile programmatically.

- [ ] **9. Diagnose common install failures:**
    | Error | Cause | Fix |
    |---|---|---|
    | `INSTALL_FAILED_VERSION_DOWNGRADE` | New APK's `versionCode` is lower than the installed one. | Add `-d`. |
    | `INSTALL_FAILED_USER_RESTRICTED` | Multi-user device: the install user (typically the secondary or work profile) blocks installs. | `--user 0` or `--user current`; have the user disable "Install via USB" restriction in their profile. |
    | `INSTALL_FAILED_UPDATE_INCOMPATIBLE` (signature mismatch) | New APK signed with a different key than the installed one. | Uninstall first (`adb uninstall <pkg>`); resign with the same key. |
    | `INSTALL_FAILED_TEST_ONLY` | APK has `android:testOnly="true"` but `-t` was omitted. | Add `-t` (typical for `app-androidTest.apk`). |
    | `INSTALL_FAILED_INVALID_APK` (split APK only) | Tried to install one APK of a split set. | Use `adb install-multiple`. |
    | `INSTALL_FAILED_INSUFFICIENT_STORAGE` | Out of space. | `adb shell df /data` and cleanup. |

- [ ] **10. Quote the shell correctly (Platform Tools 23+ rule).** Verbatim from the adb page:
    > "With Android Platform Tools 23 and higher, `adb` handles arguments the same way that the `ssh(1)` command does. ... `adb shell setprop key 'two words'` is now an error ... To make the command work, quote twice, once for the local shell and once for the remote shell, as you do with `ssh(1)`. For example, `adb shell setprop key \"'two words'\"` works because the local shell takes the outer level of quoting and the device still sees the inner level of quoting."
    Same rule applies for any `pm` argument with spaces, special characters, or shell metacharacters.

## Patterns

### Pattern: WRONG vs RIGHT — resetting app state between tests

```bash
# WRONG
adb shell am force-stop com.example.app
# Run next test...
# WRONG because: force-stop kills the process but leaves SharedPreferences,
# Room DB, files in /data/data/<pkg>, and cache. Tests that read any of those
# carry stale state into the next case. Flake follows.
```

```bash
# RIGHT
adb shell pm clear com.example.app
# Run next test against fresh-install state.
```

For test runs, the equivalent done by the runner itself is `-e clearPackageData true` to AndroidX Test Orchestrator — see `../../tests/running-instrumented-tests-via-adb/SKILL.md`.

### Pattern: WRONG vs RIGHT — installing an `androidTest` APK

```bash
# WRONG
adb install app-androidTest.apk
# adb: failed to install app-androidTest.apk: Failure [INSTALL_FAILED_TEST_ONLY:
#   installPackageLI]
# WRONG because: any APK with android:testOnly="true" (every Gradle-built test
# APK) requires `-t`.
```

```bash
# RIGHT
adb install -r -t -g app-androidTest.apk
# -r: reinstall keeping data (or overwriting); -t: allow test packages;
# -g:  grant all runtime permissions for hermetic tests.
```

### Pattern: WRONG vs RIGHT — version downgrade after a debug install

```bash
# WRONG
adb install -r app-release.apk
# adb: failed to install app-release.apk: Failure [INSTALL_FAILED_VERSION_DOWNGRADE]
# WRONG because: the installed debug APK has a higher versionCode than the
# release APK being installed.
```

```bash
# RIGHT
adb install -r -d app-release.apk     # allow downgrade
# Or, when signature also changes:
adb uninstall com.example.app
adb install     app-release.apk
```

### Pattern: WRONG vs RIGHT — listing only third-party packages

```bash
# WRONG
adb shell pm list packages | grep -v 'com.android'
# WRONG because: many third-party packages also have names containing
# "com.android" (e.g. third-party Android library demos), and Google's own
# pre-installed apps carry varied prefixes (com.google.android.*).
```

```bash
# RIGHT
adb shell pm list packages -3
# `-3` filters to third-party (non-system) packages only.
```

## Mandatory rules

- **MUST** use `pm clear <pkg>` to reset app state between tests; `am force-stop` does NOT clear data.
- **MUST** add `-t` when installing any test APK (`android:testOnly="true"`); add `-d` when downgrading; add `-g` to grant all manifest runtime permissions.
- **MUST** use `adb install-multiple` for split APKs (App Bundles). Single-APK install of one split fails with `INSTALL_FAILED_INVALID_APK`.
- **MUST NOT** assume `adb install` of a freshly built debug APK will replace a release-signed install — signing-conflict errors require uninstall first.
- **MUST NOT** rely on `pm list packages | grep` heuristics; use the documented filters (`-3`, `-s`, `-d`, `-e`, `-u`).
- **PREFERRED:** wire `clearPackageData true` through Test Orchestrator (`../../tests/running-instrumented-tests-via-adb/SKILL.md`) instead of running `pm clear` manually before every test.
- **PREFERRED:** quote string arguments twice when they contain spaces or shell metacharacters — Platform Tools 23+ requires this (ssh-style quoting).

## Verification

- [ ] `adb install -r -t -g app-androidTest.apk` returns `Success`.
- [ ] `adb shell pm path com.example.app` prints a `package:/data/app/...base.apk` line for the just-installed package.
- [ ] `adb shell pm clear com.example.app` returns `Success`, and `adb shell run-as com.example.app ls /data/data/com.example.app` shows only freshly created `cache/` and `code_cache/` entries.
- [ ] `adb shell pm list packages -3 | grep com.example.app` lists the package after install.
- [ ] After `adb shell pm grant com.example.app android.permission.CAMERA`, `adb shell dumpsys package com.example.app | grep CAMERA` shows the permission as granted.
- [ ] `adb uninstall com.example.app` returns `Success` and `pm list packages com.example` no longer shows the app.
- [ ] CI run wipes app state between tests via Orchestrator (`-e clearPackageData true`) or an explicit `pm clear` step in `@Before` / `@After`.

## References

- ADB user guide (`pm install`, `pm uninstall`, `pm clear`, `pm grant`, `pm list packages`): https://developer.android.com/tools/adb#pm
- AGP/Studio command-line testing (`am instrument` and orchestrator wiring): https://developer.android.com/studio/test/command-line
- Baseline Profile dump via `cmd package dump-profiles`: https://developer.android.com/topic/performance/baselineprofiles
- App Bundle / split APK install: https://developer.android.com/studio/build/build-variants
- `tasks/research/A2-adb-shell-commands.md` — verbatim `pm install`/`pm list`/`pm clear` flag tables, `force-stop` vs `pm clear` distinction, ssh-style double-shell quoting since Platform Tools 23.
- `docs/CORPUS.md` §I.10 — "force-stop does NOT clear data — pair with `pm clear` for hermetic reset."
- Sibling skills:
  - High-level architecture: `../../architecture/understanding-adb-architecture/SKILL.md`
  - Connect a device: `../../devices/connecting-to-devices/SKILL.md`
  - Wireless ADB: `../../devices/connecting-over-wifi/SKILL.md`
  - Run tests via `am instrument` (and Orchestrator `clearPackageData`): `../../tests/running-instrumented-tests-via-adb/SKILL.md`
- Cross-set neighbours:
  - Run instrumented tests with `AndroidJUnit4`: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`
  - Configure JUnit4 on Android: `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`
  - Source-set strategy: `../../../fundamentals/strategies/organizing-test-source-sets/SKILL.md`
