---
name: injecting-input-and-state
description: Use this skill to drive an Android device from the host shell — inject taps, swipes, text, key events, and drag-and-drop via `adb shell input`; resize the display with `wm size` / `wm density`; flip hermetic-test settings with `settings put global window_animation_scale 0`; reset app state with `pm clear` vs `am force-stop`; launch Activities with `am start -n pkg/.Activity`; and use the modern `cmd <service>` wrapper (`cmd wifi`, `cmd connectivity`, `cmd uimode night`, `cmd locale set-app-locales`) instead of the deprecated `svc` aliases on API 30+. If the user mentions "input tap", "input swipe", "input text spaces", "KEYCODE_BACK", "wm size for screen-size matrix", "settings put animation_scale", "force-stop vs pm clear", "svc wifi no-op", "cmd wifi set-wifi-enabled", or "am start intent extras", use this skill.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - adb
  - input-injection
  - hermetic-tests
  - wm-size
  - settings-put
  - cmd-service
  - force-stop
  - pm-clear
  - keycode
---

# Injecting Input and State — Drive the Device From the Shell

This skill covers the on-device shell surface a test/QA engineer drives over `adb` to simulate user gestures, override the screen configuration, flip system settings, reset app state, and launch Activities. Transport (`adb devices`, USB/TCP) lives in `../../architecture/understanding-adb-architecture/SKILL.md`; capturing artefacts after driving the device lives in `../../capture/capturing-screenshots-and-screenrecord/SKILL.md`.

## When to use this skill

- The developer needs to script a user journey from CI without writing instrumentation: tap, swipe, type text, press hardware keys.
- A bug repro requires a specific screen size or density (`wm size 411x731`, `wm density 320`) without recreating an emulator AVD.
- The test suite is flaky because animations are on; the developer wants the canonical hermetic preamble (`window_animation_scale = 0` etc.).
- The developer needs to clear app data between runs and is unsure whether `am force-stop` is enough (it is not).
- A script uses `adb shell svc wifi disable` and silently does nothing on API 30+ — the modern `cmd wifi set-wifi-enabled disabled` is the fix.
- The developer needs to launch a deep-link Activity with typed extras (`--ez`, `--ei`, `--es`).

## When NOT to use this skill

- The bytes need to leave the device (screenshots, logs, DB). Use `../../capture/capturing-screenshots-and-screenrecord/SKILL.md`, `../../observability/extracting-logs-with-logcat/SKILL.md`, `../../transfer/extracting-test-artifacts/SKILL.md`.
- Driving the device from inside an instrumentation test, not from the host. Use `../../../instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md` or UiAutomator.
- Installing or uninstalling APKs. Use `../../apps/installing-and-managing-apps/SKILL.md`.
- Connecting to the device in the first place. Use `../../devices/connecting-to-devices/SKILL.md` or `../../devices/connecting-over-wifi/SKILL.md`.

## Prerequisites

- A reachable device or emulator (`adb get-state` returns `device`).
- Platform Tools 23+ on the host. Quoting through `adb shell` is double-shell (ssh-style) since Platform Tools 23 — load-bearing for any string with spaces (developer.android.com/tools/adb#shellcommands).
- For `cmd` wrapper APIs and reliable shell exit codes: device API 24+ (`cmd` introduced API 24).

## `input` — synthesised user events

`input` is the on-device wrapper around `InputManager.injectInputEvent()`. The full grammar:

```
input [<source>] <command> [<arg>...]
```

`<source>` defaults per command (touchscreen for `tap`/`swipe`, keyboard for `text`/`keyevent`).

### Tap, swipe, draganddrop

```bash
adb shell input tap <x> <y>
adb shell input swipe <x1> <y1> <x2> <y2> [<duration_ms>]
adb shell input draganddrop <x1> <y1> <x2> <y2> [<duration_ms>]
adb shell input roll <dx> <dy>                        # rotary/trackball axes
adb shell input motionevent <DOWN|UP|MOVE|CANCEL> <x> <y>
```

`swipe` defaults to ~300 ms when `<duration_ms>` is omitted. To synthesise a long-press, swipe in place for the long-press timeout: `input swipe <x> <y> <x> <y> 1000`. `draganddrop` differs from `swipe` in that it emits the drag-and-drop gesture sequence (DOWN → MOVE → UP) consumable by `View.OnDragListener`, not the same event stream as `swipe`.

### Text input — quoting trap

```bash
adb shell input text "<string>"
```

Caveats (from developer.android.com/tools/adb):

- **Spaces MUST be encoded as `%s`.** `input text "hello world"` types `hello`, then nothing.
- **Shell metacharacters** (`'`, `"`, `&`, `(`, `)`, `<`, `>`, `|`, `;`, `\`, backtick) MUST be escaped twice — once for the local shell, once for the device-side shell.
- **Non-ASCII / Unicode is unreliable** through `input text`. Use UiAutomator `UiObject2.setText(...)` for those.

### Key events

```bash
adb shell input keyevent [--longpress|--doubletap] <KEYCODE> [<KEYCODE>...]
adb shell input keyevent <NUMERIC_CODE>
```

Test-relevant `KEYCODE_*` (KeyEvent constants from frameworks/base; full table in research note A2):

| Mnemonic | Numeric | Notes |
|---|---|---|
| `KEYCODE_HOME` | 3 | Press Home (does not always wake screen). |
| `KEYCODE_BACK` | 4 | Back button. |
| `KEYCODE_DPAD_UP/DOWN/LEFT/RIGHT/CENTER` | 19-23 | TV / focus traversal. |
| `KEYCODE_VOLUME_UP/DOWN/MUTE` | 24/25/164 | |
| `KEYCODE_POWER` | 26 | Toggles screen off/on. |
| `KEYCODE_TAB` | 61 | |
| `KEYCODE_ENTER` | 66 | |
| `KEYCODE_DEL` | 67 | Backspace. |
| `KEYCODE_MENU` | 82 | |
| `KEYCODE_APP_SWITCH` | 187 | Recents button. |
| `KEYCODE_WAKEUP` | 224 | Wake without unlocking. |
| `KEYCODE_SLEEP` | 223 | Force sleep. |

Wake + dismiss-keyguard for emulator with no lock:

```bash
adb shell input keyevent KEYCODE_WAKEUP
adb shell input keyevent 82                     # menu, dismiss the AOSP swipe-up
```

### Source prefixes

`input <source> <command>` maps to `InputDevice.SOURCE_*`. Use a source prefix when a screen has multiple input listeners or when targeting non-touch hardware:

| Source | Typical use |
|---|---|
| `touchscreen` | Default for `tap`, `swipe`, `draganddrop`. |
| `touchpad` | External touchpads (Chromebook / Android tablet keyboard). |
| `dpad` | TV/focus traversal: `input dpad keyevent KEYCODE_DPAD_DOWN`. |
| `keyboard` | Default for `text`, `keyevent`. |
| `mouse` | Mouse buttons + scroll. |
| `trackball` | Trackball roll. |
| `gamepad` / `joystick` | Gamepad buttons / joystick axes. |
| `stylus` | Pressure-aware touch (S-Pen). |

## `wm` — display size and density overrides

`wm` is the shell wrapper for `IWindowManager`. The test-relevant subcommands:

```bash
adb shell wm size                               # read current logical size
adb shell wm size 1080x1920                     # override
adb shell wm size reset                         # restore device default

adb shell wm density                            # read current dpi
adb shell wm density 320                        # override
adb shell wm density reset

adb shell wm dismiss-keyguard                   # insecure keyguard only
adb shell wm user-rotation [free|lock [<rotation>]]   # 0,1,2,3 = 0/90/180/270
```

Use case — screen-size / density matrix without recreating the emulator:

```bash
# Per configuration in the matrix
adb shell wm size 411x731
adb shell wm density 240
# … run tests …
adb shell wm size reset
adb shell wm density reset
```

Caveats:

- **MUST** call `reset` in `@AfterClass` / suite teardown — otherwise the device stays in the modified configuration after the run, breaking later runs and any human user.
- `wm size` / `wm density` trigger a configuration change that recreates the foreground Activity. Treat them as between-test knobs, not mid-test.
- `am display-size` / `am display-density` are equivalent legacy spellings; `wm` is preferred.

## `settings` — hermetic test state

```bash
adb shell settings [--user <id>] get   <namespace> <key>
adb shell settings [--user <id>] put   <namespace> <key> <value> [<tag>] [default]
adb shell settings [--user <id>] delete <namespace> <key>
adb shell settings [--user <id>] reset <namespace> [<package>] [untrusted_defaults|untrusted_clear|trusted_defaults]
adb shell settings [--user <id>] list  <namespace>
```

`<namespace>` is one of `global`, `system`, `secure`.

### The hermetic preamble — animation knobs

Set all three to `0` for stable instrumented and screenshot tests:

```bash
adb shell settings put global window_animation_scale 0
adb shell settings put global transition_animation_scale 0
adb shell settings put global animator_duration_scale 0
```

Or via Gradle: `testOptions.animationsDisabled = true`.

### Other test-relevant keys

| Namespace | Key | Purpose |
|---|---|---|
| `system` | `font_scale` | Float (e.g. `1.0`, `1.3`); type-scale tests. |
| `secure` | `accessibility_enabled` | `1` to enable a11y; pair with `enabled_accessibility_services`. |
| `secure` | `enabled_accessibility_services` | CSV `pkg/.Service` for a11y tests. |
| `secure` | `default_input_method` | Force a known IME. |
| `secure` | `show_ime_with_hard_keyboard` | `1` to keep IME visible on emulators. |
| `secure` | `long_press_timeout` | Stabilise long-press tests. |
| `global` | `airplane_mode_on` | Pair with `am broadcast android.intent.action.AIRPLANE_MODE`. |
| `global` | `policy_control` | E.g. `immersive.full=*` to hide system bars. |

## App state — `am force-stop` vs `pm clear`

| Command | Kills processes? | Clears data/cache/prefs? | Resets runtime permissions? |
|---|---|---|---|
| `am force-stop <pkg>` | Yes | **No** | No |
| `pm clear <pkg>` | Yes (implicitly) | Yes | Yes |
| `cmd activity stop-app <pkg>` | Yes (modern, API 28+) | No | No |

For a hermetic between-test reset, use `pm clear`. For "kill the process so the next launch is cold", `am force-stop` is enough. See `../../apps/installing-and-managing-apps/SKILL.md` for the full `pm` surface.

## Launching Activities — `am start`

```bash
# Most reliable for tests: explicit component.
adb shell am start -n com.example.app/.MainActivity

# Force-stop first, then launch (clean cold start).
adb shell am start -S -W -n com.example.app/.MainActivity

# Launch by action + URI (default handler for https://).
adb shell am start -a android.intent.action.VIEW -d 'https://example.com'

# Pass typed extras.
adb shell am start -n com.example.app/.MainActivity \
  --es screen "settings" --ez debug true --ei count 42

# Return to launcher between tests.
adb shell am start -a android.intent.action.MAIN -c android.intent.category.HOME
```

Intent specifier flags (abridged from developer.android.com/tools/adb#IntentSpec):

| Token | Effect |
|---|---|
| `-a action` | Intent action (e.g. `android.intent.action.VIEW`). |
| `-d data_uri` | Data URI. |
| `-t mime_type` | MIME type. |
| `-c category` | Category. |
| `-n component` | Explicit component (`pkg/.Activity`). |
| `-f flags` | Hex `setFlags()` value. |
| `--es | -e key value` | String extra. |
| `--ez key bool` | Boolean extra. |
| `--ei key int` | Int extra. |
| `--el key long` | Long extra. |
| `--ef key float` | Float extra. |
| `--eu key uri` | URI extra. |
| `--ecn key component` | ComponentName extra. |
| `--eia/--ela/--efa key v[,v...]` | Array extras. |
| `--activity-clear-top` etc. | Map to `FLAG_ACTIVITY_*`. |
| `-S` | Force-stop the target before starting. |
| `-W` | Wait for launch to complete. |

## `cmd <service>` — the modern wrapper

`cmd` is a generic wrapper that calls into a binder service's `onShellCommand`. Available since API 24. To list services: `adb shell cmd -l` (or `dumpsys -l`). Test-relevant entry points:

| Invocation | Effect |
|---|---|
| `cmd activity stop-app <pkg>` | Modern equivalent of `am force-stop`. |
| `cmd activity broadcast …` | Same as `am broadcast`. |
| `cmd package install -r -t -g <path>` | `pm install` equivalent. |
| `cmd package compile -m speed -f <pkg>` | Force AOT compilation. |
| `cmd appops set <pkg> <op> allow|deny|ignore|default` | Toggle an AppOp (e.g. `RUN_IN_BACKGROUND`). |
| `cmd jobscheduler run -f <pkg> <jobId>` | Force a JobService to run now. |
| `cmd uimode night yes|no|auto` | Toggle dark mode for the current user. |
| `cmd locale set-app-locales <pkg> --locales en-US` | Per-app locales (Android 13+). |
| `cmd statusbar expand-notifications` / `collapse` | Expand/collapse the shade. |
| `cmd wifi set-wifi-enabled enabled|disabled` | Wi-Fi toggle (preferred to `svc wifi` on API 30+). |
| `cmd connectivity airplane-mode enable|disable` | Airplane-mode toggle on modern releases. |
| `cmd testharness enable` | Reset device to a hermetic test-harness state (Android 10+). |

### `svc` deprecation on API 30+

`svc data`, `svc wifi`, `svc bluetooth` are **no-ops on API 30+** unless invoked from a privileged shell (corpus §I.10). The preferred replacements are the `cmd` wrappers above.

## Patterns

### Pattern: WRONG — `svc wifi disable` on API 30+

```bash
# WRONG
adb shell svc wifi disable
# WRONG because: silently no-ops on API 30+ since the svc CLI was demoted from a privileged
# helper to an ordinary shell command. Exit code 0, Wi-Fi still on. The script proceeds
# under a wrong assumption — exactly the worst kind of failure mode.
```

```bash
# RIGHT
adb shell cmd wifi set-wifi-enabled disabled
adb shell cmd wifi status                       # verify
```

### Pattern: WRONG — `am force-stop` for a between-test reset

```bash
# WRONG
adb shell am force-stop com.example.app
./run-next-test.sh
# WRONG because: am force-stop kills the process but leaves shared prefs, databases, the
# image cache, and granted runtime permissions intact. The "next test" inherits state
# from the previous run.
```

```bash
# RIGHT
adb shell pm clear com.example.app
adb shell pm clear com.example.app.test         # also reset test process state
./run-next-test.sh
```

(Test Orchestrator with `clearPackageData: 'true'` is the equivalent inside Gradle — see `../../tests/running-instrumented-tests-via-adb/SKILL.md`.)

### Pattern: WRONG — `input text "hello world"`

```bash
# WRONG
adb shell input text "hello world"
# WRONG because: spaces in `input text` MUST be encoded as %s. The above types only "hello".
```

```bash
# RIGHT
adb shell input text 'hello%sworld'
```

### Pattern: hermetic preamble + Activity launch

```bash
adb shell settings put global window_animation_scale 0
adb shell settings put global transition_animation_scale 0
adb shell settings put global animator_duration_scale 0
adb shell pm clear com.example.app
adb shell am start -S -W -n com.example.app/.MainActivity
```

`-S` force-stops first; `-W` blocks until the launch completes.

### Pattern: TV / focus navigation via dpad

```bash
adb shell input dpad keyevent KEYCODE_DPAD_DOWN
adb shell input dpad keyevent KEYCODE_DPAD_DOWN
adb shell input dpad keyevent KEYCODE_DPAD_CENTER
```

Source-prefixing `dpad` ensures the events route to TV-style focus handlers, not the touchscreen pipeline.

## Mandatory rules

- **MUST** use `cmd wifi set-wifi-enabled` / `cmd connectivity airplane-mode` / `cmd bluetooth_manager enable` on API 30+. **MUST NOT** rely on `svc wifi` / `svc bluetooth` / `svc data` — they no-op silently on modern releases.
- **MUST** use `pm clear <pkg>` (not `am force-stop`) when the goal is a hermetic between-test reset. `force-stop` only kills processes; data is untouched.
- **MUST** encode spaces in `input text` as `%s`; **MUST** double-shell quote (ssh-style) any string with shell metacharacters. Platform Tools 23+ (developer.android.com/tools/adb#shellcommands).
- **MUST** call `wm size reset` and `wm density reset` in suite teardown when the suite overrode them. A leaked override breaks every subsequent run on that device.
- **MUST** set the three animation scales to `0` (or use `testOptions.animationsDisabled = true` in Gradle) before any UI-asserting test run.
- **MUST NOT** use `Thread.sleep` between `adb shell input` calls to "let the UI catch up". Inside an instrumentation test, use Compose's idle synchronisation (`../../../compose/synchronization/synchronizing-with-idle/SKILL.md`); from a host script, use `adb shell am start -W` to block on Activity launch and `adb shell wait-for-device` between reboots.
- **MUST NOT** assume `adb shell am instrument` returns a meaningful exit code without `-w`. See `../../tests/running-instrumented-tests-via-adb/SKILL.md` and `../../automation/scripting-adb-for-ci/SKILL.md`.
- **PREFERRED:** `am start -S -W -n pkg/.Activity` for cold-start launches in tests — `-S` resets, `-W` blocks until ready.

## Verification

- [ ] `adb shell settings get global window_animation_scale` returns `0` (or `0.0`) before tests run; same for `transition_animation_scale` and `animator_duration_scale`.
- [ ] No script in `scripts/` or `.github/workflows/` uses `svc wifi` / `svc bluetooth` / `svc data`.
- [ ] Every `am force-stop` between tests is paired with (or replaced by) `pm clear`.
- [ ] No `input text "<string with spaces>"` exists; all spaces are `%s`.
- [ ] Every `wm size <override>` in setup has a matching `wm size reset` in teardown.
- [ ] CI passes 50 consecutive runs without a "device entered prior config" flake.

## References

- developer.android.com/tools/adb — `am`, `pm`, `input`, `wm`, `settings`, intent specs (https://developer.android.com/tools/adb).
- developer.android.com/tools/adb#shellcommands — quoting through `adb shell` since Platform Tools 23.
- developer.android.com/reference/android/view/KeyEvent — `KEYCODE_*` constants table.
- developer.android.com/training/testing/instrumented-tests/stability — the canonical animation-scale-zero recipe.
- developer.android.com/studio/test/advanced-test-setup#use-gradle-managed-devices — `testOptions.animationsDisabled`.
- Research note `tasks/research/A2-adb-shell-commands.md` — full `input` / `wm` / `settings` / `am` / `pm` / `cmd` / `svc` tables and verbatim quotes.
- Sibling skill: `../../architecture/understanding-adb-architecture/SKILL.md` — three-piece architecture, server, daemon.
- Sibling skill: `../../devices/connecting-to-devices/SKILL.md` — `adb devices` states, `wait-for-device`.
- Sibling skill: `../../devices/connecting-over-wifi/SKILL.md` — `adb pair` / `adb connect`.
- Sibling skill: `../../apps/installing-and-managing-apps/SKILL.md` — full `pm install` / `pm uninstall` / `pm grant` surface.
- Sibling skill: `../../tests/running-instrumented-tests-via-adb/SKILL.md` — `am instrument -w -r`.
- Sibling skill: `../../capture/capturing-screenshots-and-screenrecord/SKILL.md` — capturing artefacts after driving the device.
- Sibling skill: `../../observability/extracting-logs-with-logcat/SKILL.md` — reading device logs.
- Sibling skill: `../../transfer/extracting-test-artifacts/SKILL.md` — `adb pull` / `adb push` / `run-as`.
- Sibling skill: `../../automation/scripting-adb-for-ci/SKILL.md` — bash idioms, retries, port forwarding.
- Cross-set: `../../../instrumentation/scenarios/launching-activities-with-activityscenario/SKILL.md` — launching from inside an instrumentation test.
- Cross-set: `../../../fundamentals/strategies/applying-testing-strategies/SKILL.md` — when to drive via `adb` vs instrumentation.
