---
name: connecting-to-devices
description: >-
  Use this skill to attach a USB device or emulator to ADB, list transports with `adb devices` / `adb devices -l`, disambiguate among multiple devices using `-s SERIAL`, `-d` (single USB), `-e` (single TCP/IP), or `-t TRANSPORT_ID`, gate scripts on a transport with the canonical `adb wait-for[-TRANSPORT]-<state>` syntax (TRANSPORT in {usb, local, any}; state in {device, recovery, rescue, sideload, bootloader, disconnect}), interpret device states (`device`, `offline`, `unauthorized`, `no permissions`, `recovery`, `sideload`, `bootloader`, `rescue`), accept the RSA fingerprint dialog on first connect, and install Linux udev rules. Use when the user mentions `error: more than one device/emulator`, `error: device not found`, `unauthorized`, `no permissions`, `daemon not running`, "wait for device to boot", `wait-for-device-online` (which is not a real subcommand), or asks how to script around emulator startup.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - adb-devices
  - device-selector
  - transport-id
  - wait-for-device
  - unauthorized
  - no-permissions
  - udev-rules
  - usb-debugging
  - rsa-fingerprint
  - sys.boot_completed
---

# Connecting to Devices — Listing, Selecting, and Waiting on Transports

`adb devices` is the most-typed adb command. The follow-on traps are real: scripts run before the device is actually online, multi-device shells fail with `more than one device/emulator`, Linux misses `udev` rules, and tutorials reference a fictional `wait-for-device-online` form. This skill encodes the truth set.

## When to use this skill

- The user runs `adb shell` with two devices plugged in and gets `error: more than one device/emulator`.
- The user reports a device showing as `unauthorized`, `offline`, or `no permissions` in `adb devices`.
- The user wants a CI script to wait until a device finishes booting before running tests.
- The user copy-pasted `adb wait-for-device-online` from a blog post and it errors with usage text.
- The user is on Linux and a freshly plugged phone never appears in `adb devices`.

## When NOT to use this skill

- The user wants the high-level architecture / server lifecycle / env vars — use `../../architecture/understanding-adb-architecture/SKILL.md`.
- The user wants Wi-Fi / wireless debugging — use `../connecting-over-wifi/SKILL.md`.
- The user wants to install or clear app state on the connected device — use `../../apps/installing-and-managing-apps/SKILL.md`.

## Prerequisites

- Working ADB server (see `../../architecture/understanding-adb-architecture/SKILL.md`).
- Device with **USB debugging** enabled in `Settings → System → Developer options`. On Android 4.2+ developer options is hidden until "Build number" is tapped seven times in `About phone`.
- On Linux: ability to write `/etc/udev/rules.d/51-android.rules` with sudo.
- On Windows: a vendor USB driver (Google USB Driver via SDK Manager for Pixel/Nexus; OEM driver for other manufacturers).

## Workflow

- [ ] **1. Run `adb devices` first, every time.** It prints one line per attached transport in the form `<serial> <state> [details...]`. Add `-l` whenever scripting:
    ```bash
    adb devices -l
    # List of devices attached
    # emulator-5556  device  product:sdk_google_phone_x86_64 model:Android_SDK_built_for_x86_64 device:generic_x86_64 transport_id:1
    # 0a388e93       device  usb:1-1 product:razor model:Nexus_7 device:flo transport_id:2
    ```
    `-l` adds `product`, `model`, `device`, `transport_id`, and (for USB) the USB bus path.

- [ ] **2. Pick the right selector for the situation:**
    | Flag | Meaning | Fails when |
    |---|---|---|
    | `-s <serial>` | Explicit serial. Works for USB serials (`0a388e93`), emulators (`emulator-5554`), and TCP devices (`192.168.1.42:5555`). Overrides `$ANDROID_SERIAL`. | Serial not connected. |
    | `-d` | The single USB device. | Zero or more than one USB device. |
    | `-e` | The single TCP/IP device (covers emulators and `adb connect`-ed phones). | Zero or more than one TCP device. |
    | `-t <transport_id>` | Numeric transport ID from `adb devices -l`. Stable across re-plugs of the same port. | Transport ID not present. |

    Note: `-t` is the **transport ID**, not a timeout flag. Wrap with shell `timeout` (or `gtimeout` on macOS) if the goal is a per-command time limit.

- [ ] **3. Filter to "ready" devices in scripts:**
    ```bash
    adb devices | awk '$2=="device"{print $1}'
    ```
    Single-device shortcuts: `adb get-serialno` returns the serial, `adb get-state` returns one of `offline | bootloader | device`, `adb get-devpath` returns the USB path. For the full state set, parse `adb devices` directly.

- [ ] **4. Use the canonical `wait-for-*` form. The variants you may have seen are wrong.** From `adb help` (`scripting` section), the **only** valid syntax is:
    ```
    wait-for[-TRANSPORT]-STATE
        TRANSPORT ∈ { usb, local, any }            (transport defaults to any)
        STATE     ∈ { device, recovery, rescue, sideload, bootloader, disconnect }
    ```
    Common forms:
    | Command | Blocks until... |
    |---|---|
    | `adb wait-for-device` | Any transport, state `device`. The default — used at the start of CI scripts. |
    | `adb wait-for-usb-device` | A USB-attached device reaches state `device`. |
    | `adb wait-for-local-device` | A TCP/IP device (emulator or wireless) reaches state `device`. |
    | `adb wait-for-bootloader` | Device shows up in `bootloader` (fastboot) state. |
    | `adb wait-for-recovery` / `wait-for-sideload` / `wait-for-rescue` | Same idea for those modes. |
    | `adb wait-for-disconnect` | The currently selected device disconnects (e.g. after `adb reboot`). |

    The doc does **NOT** define a `wait-for-device-online` form. `wait-for-*` only checks the transport, not boot completion — pair it with a `sys.boot_completed` poll:

    ```bash
    adb wait-for-device
    until [[ "$(adb shell getprop sys.boot_completed | tr -d '\r')" == "1" ]]; do
      sleep 1
    done
    adb shell input keyevent 82   # unlock the AOSP slide-up screen
    ```

    Compound flow around a reboot:
    ```bash
    adb shell reboot
    adb wait-for-disconnect
    adb wait-for-device
    ```

- [ ] **5. Read device states correctly.** From the adb page (`Query for devices`) plus `adb help`:
    | State | Meaning | Recovery |
    |---|---|---|
    | `device` | Online, `adbd` is responsive. **Does not** imply boot complete — gate on `sys.boot_completed`. | n/a |
    | `offline` | Transport exists but `adbd` isn't talking. Common after suspend/resume or USB hubs. | `adb reconnect offline`, then re-plug if needed. |
    | `unauthorized` | Host RSA key not yet accepted on-device. | Unlock device, tap **Allow** on the on-device dialog. Try a different USB cable if no dialog appears (some "charge-only" cables block data). |
    | `recovery` | Booted into recovery. Only a small subset of `adb` works. | n/a |
    | `sideload` | OTA sideload mode. Use `adb sideload package.zip`. | n/a |
    | `bootloader` | At the bootloader (fastboot). `adb` cannot talk; use `fastboot`. | n/a |
    | `rescue` | Rescue Party mode (Android 10+). | n/a |
    | `connecting` | Transient TLS handshake state over wireless. | wait. |
    | `no permissions` | (Linux only) USB visible but blocked by missing udev rules. | install rules — see step 7. |

- [ ] **6. Accept the RSA fingerprint dialog on first connect.**
    > "When you connect a device running Android 4.2.2 (API level 17) or higher, the system shows a dialog asking whether to accept an RSA key that allows debugging through this computer." — developer.android.com/tools/adb
    Sequence:
    1. `adbd` sees a new client offering an RSA public key.
    2. The OS pops the "Allow USB debugging?" dialog showing the workstation's key fingerprint.
    3. Until the user taps **Allow**, `adb devices` reports `unauthorized`.
    4. After acceptance, the public key is appended to `/data/misc/adb/adb_keys` on the device. Tick **Always allow from this computer** so future connections from the same `~/.android/adbkey` skip the prompt.

    Revoke an old workstation's permission via `Settings → Developer options → Revoke USB debugging authorizations`.

- [ ] **7. (Linux) install udev rules** so a freshly plugged phone is not stuck at `no permissions`. Canonical file: `/etc/udev/rules.d/51-android.rules`. Minimal example (one line per OEM):
    ```
    # Google
    SUBSYSTEM=="usb", ATTR{idVendor}=="18d1", MODE="0660", GROUP="plugdev", TAG+="uaccess"
    # Samsung
    SUBSYSTEM=="usb", ATTR{idVendor}=="04e8", MODE="0660", GROUP="plugdev", TAG+="uaccess"
    ```
    Apply without reboot:
    ```bash
    sudo udevadm control --reload-rules
    sudo udevadm trigger
    # unplug and re-plug the device
    ```
    Most distros also ship a comprehensive `android-udev` / `android-sdk-platform-tools-common` package that drops the same file in `/lib/udev/rules.d/`.

- [ ] **8. (Windows) install the vendor USB driver.** Pixel / Nexus / generic AOSP devices use the **Google USB Driver** from `SDK Manager → SDK Tools → Google USB Driver` (files at `android_sdk\extras\google\usb_driver\`). Other OEMs require manufacturer drivers from `https://developer.android.com/tools/extras/oem-usb`. macOS and Linux do not need vendor drivers.

- [ ] **9. Multi-device fan-out.** ADB has no built-in `--all` flag. Two patterns:
    ```bash
    # Sequential
    for s in $(adb devices | awk '$2=="device"{print $1}'); do
      adb -s "$s" install -r app.apk
    done

    # Parallel (4 jobs at a time)
    adb devices | awk '$2=="device"{print $1}' \
      | xargs -I{} -P 4 adb -s {} install -r app.apk
    ```
    For sharded CI runners that each own one phone, lock the server to a single device:
    ```bash
    adb --one-device 0a388e93 start-server
    ```

## Patterns

### Pattern: WRONG vs RIGHT — `more than one device/emulator`

```bash
# WRONG
adb shell getprop ro.product.model
# adb: more than one device/emulator
# WRONG because: with two transports attached (e.g. an emulator + a USB phone),
# every non-server adb command needs an explicit selector. The command did not
# fail; it never even ran.
```

```bash
# RIGHT
adb -s emulator-5554 shell getprop ro.product.model
# Or, if there is exactly one USB phone:
adb -d shell getprop ro.product.model
# Or set $ANDROID_SERIAL once and stop typing -s:
export ANDROID_SERIAL=emulator-5554
adb shell getprop ro.product.model
```

### Pattern: WRONG vs RIGHT — fictional `wait-for-device-online`

```bash
# WRONG
adb wait-for-device-online
# error: usage: ...
# WRONG because: this command does not exist. The canonical syntax is
# wait-for[-TRANSPORT]-<state> with state in {device, recovery, rescue,
# sideload, bootloader, disconnect}. There is no -online state.
```

```bash
# RIGHT
adb wait-for-device                  # any transport, state=device
# Then poll boot complete (transport up != system booted):
until [[ "$(adb shell getprop sys.boot_completed | tr -d '\r')" == "1" ]]; do
  sleep 1
done
```

### Pattern: WRONG vs RIGHT — racing emulator startup

```bash
# WRONG
emulator -avd Pixel_API_34 &
adb install -r app.apk            # races: emulator may not be online yet
# Often: error: device 'emulator-5554' not found
```

```bash
# RIGHT
adb start-server                  # important: see corner case below
emulator -avd Pixel_API_34 &
adb -s emulator-5554 wait-for-device
until [[ "$(adb -s emulator-5554 shell getprop sys.boot_completed | tr -d '\r')" == "1" ]]; do
  sleep 1
done
adb -s emulator-5554 install -r app.apk
```

> Corner case (verbatim from the adb page): "running emulators [may] not show up in `adb devices` ... when **all** of the following are true: the adb server is not running; you use the `emulator` command with the `-port` or `-ports` option with an odd-numbered port between 5554 and 5584; ... You start the adb server after you start the emulator." Workaround: `adb start-server` BEFORE `emulator`.

### Pattern: WRONG vs RIGHT — `transport_id` vs `-t` confusion

```bash
# WRONG
adb -t 60 shell getprop ro.product.model
# WRONG because: -t is transport_id, not a 60-second timeout. Either there is
# no transport with id=60 (error: device not found) or the command runs against
# the wrong device.
```

```bash
# RIGHT
# Real timeout via the shell tool:
timeout 60 adb -s emulator-5554 shell getprop ro.product.model     # GNU timeout
gtimeout 60 adb -s emulator-5554 shell getprop ro.product.model    # macOS (coreutils via brew)
# Use -t only with a transport_id from adb devices -l:
adb -t 2 shell getprop ro.product.model
```

## Mandatory rules

- **MUST** add a selector (`-s`/`-d`/`-e`/`-t`) on every adb command when more than one transport is attached, OR set `$ANDROID_SERIAL`.
- **MUST** use the documented `wait-for[-TRANSPORT]-<state>` syntax. There is no `wait-for-device-online`.
- **MUST** poll `getprop sys.boot_completed` after `wait-for-device` when the script needs the OS fully booted (e.g. before installing an APK or invoking `am instrument`).
- **MUST NOT** confuse `-t TRANSPORT_ID` with a timeout flag. Wrap with `timeout` / `gtimeout` for actual timeouts.
- **MUST NOT** delete `~/.android/adbkey*` to "fix" `unauthorized` — see `../../architecture/understanding-adb-architecture/SKILL.md` for the correct recovery.
- **PREFERRED:** use `adb devices -l` and `transport_id` for scripts that may see two devices with the same serial (rare hardware bug or duplicate AVDs).
- **PREFERRED:** install the distro's `android-udev` package on Linux instead of hand-maintaining `51-android.rules`.

## Verification

- [ ] `adb devices -l` lists every connected transport with state `device` (no `offline`, `unauthorized`, or `no permissions`).
- [ ] `adb devices | awk '$2=="device"{print $1}'` returns one line per ready device.
- [ ] `adb -s <serial> get-state` prints `device`.
- [ ] `adb wait-for-device` returns immediately when at least one transport is in state `device`.
- [ ] `adb shell getprop sys.boot_completed` prints `1` after a fresh boot.
- [ ] `adb -d shell echo ok` (or `-e`, or `-s`) succeeds without `more than one device/emulator`.
- [ ] On Linux, `lsusb` shows the device and `adb devices` reports `device` (not `no permissions`) after udev rules apply.

## References

- ADB user guide (`adb devices`, `wait-for-*`, RSA dialog): https://developer.android.com/tools/adb
- Run-on-device setup (Linux udev rules and Windows drivers): https://developer.android.com/studio/run/device
- OEM USB drivers (Windows): https://developer.android.com/tools/extras/oem-usb
- AOSP `adb` man page: https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/docs/user/adb.1.md
- `tasks/research/A1-adb-architecture-devices.md` — verbatim `wait-for-*` table, the `wait-for-device-online` correction, full state set, udev rule recipe.
- `docs/CORPUS.md` §I.3 (state truth set) and §I.10 (research findings).
- Sibling skills:
  - High-level architecture: `../../architecture/understanding-adb-architecture/SKILL.md`
  - Wireless ADB: `../connecting-over-wifi/SKILL.md`
  - Install / clear apps: `../../apps/installing-and-managing-apps/SKILL.md`
  - Run instrumented tests: `../../tests/running-instrumented-tests-via-adb/SKILL.md`
- Cross-set neighbours:
  - Run instrumented tests with `AndroidJUnit4`: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`
  - Configure JUnit4 on Android: `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`
  - Source-set strategy: `../../../fundamentals/strategies/organizing-test-source-sets/SKILL.md`
