---
name: connecting-over-wifi
description: Use this skill to connect ADB to an Android 11+ device wirelessly with the modern `adb pair` flow (pairing code or QR via `Settings → Developer options → Wireless debugging`), then `adb connect <host:port>` and `adb disconnect`, plus mDNS auto-discovery via `adb mdns check` / `adb mdns services` and the `_adb-tls-pairing._tcp` / `_adb-tls-connect._tcp` service types. Covers the ADB v34+ default mDNS backend (Openscreen on Linux/Windows, not Bonjour as the public doc says), the legacy `adb tcpip <port>` + `adb connect` path that pre-dates pairing, security caveats (corporate Wi-Fi blocks p2p; flat LAN exposure), version requirements (Android 11/API 30 for phones, Android 13/API 33 for TV+Wear), and the `ADB_MDNS_OPENSCREEN` / `ADB_MDNS_AUTO_CONNECT` environment variables. Use when the user mentions `adb pair`, `adb connect`, "wireless debugging", "QR code pairing", `_adb-tls-connect`, "Openscreen vs Bonjour", `adb tcpip 5555`, or "device not connecting after pairing successfully".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - adb-pair
  - adb-connect
  - wireless-debugging
  - mdns
  - openscreen
  - tls-pairing
  - tcpip
  - android-11
  - ADB_MDNS_OPENSCREEN
  - corporate-wifi
---

# Connecting Over Wi-Fi — `adb pair` (Modern) and `adb tcpip` (Legacy)

Android 11 introduced a paired wireless debugging flow that replaces the legacy `adb tcpip` dance. The modern path encrypts and authenticates over TLS using a pairing-code-derived shared secret and uses ephemeral ports; the legacy path leaves an unauthenticated TCP listener on a well-known port. This skill covers both, plus the mDNS plumbing the host server uses to discover paired devices.

## When to use this skill

- The user wants to connect a phone to ADB without a USB cable.
- The user mentions `adb pair`, QR-code pairing, the "Wireless debugging" toggle in Developer options, or pairing codes.
- The user copies `adb tcpip 5555 && adb connect 192.168.1.100:5555` from an old blog and asks why a modern phone treats that as insecure.
- The user sees `connection refused` after pairing and is on a corporate Wi-Fi.
- The user sees mDNS Bonjour vs Openscreen confusion in `adb mdns check`.

## When NOT to use this skill

- The user wants USB device basics (selector flags, RSA dialog, udev rules) — use `../connecting-to-devices/SKILL.md`.
- The user wants the architecture / env-var cheat sheet — use `../../architecture/understanding-adb-architecture/SKILL.md`.
- The user wants to run a test on the wirelessly connected device — use `../../tests/running-instrumented-tests-via-adb/SKILL.md`.

## Prerequisites

- Workstation and device on the **same Wi-Fi network**. Corporate networks frequently block p2p — see step 6.
- **Phones / tablets:** Android 11 (API 30)+. **TV / Wear OS:** Android 13 (API 33)+.
- Working ADB server (see `../../architecture/understanding-adb-architecture/SKILL.md`).
- Wireless debugging enabled on-device: `Settings → System → Developer options → Wireless debugging`.

## Workflow

- [ ] **1. Prefer the modern `adb pair` flow on Android 11+.** From developer.android.com/tools/adb#wireless:
    > "Android 11 (API level 30) and higher support deploying and debugging your app wirelessly from your workstation using Android Debug Bridge (adb). For example, you can deploy your debuggable app to multiple remote devices without ever needing to physically connect your device via USB."

    User flow with a pairing code:
    1. On the device, open `Settings → Developer options → Wireless debugging` and toggle it on.
    2. Tap **Pair device with pairing code**. The device shows an `IP:PORT` and a six-digit pairing code.
    3. On the workstation:
       ```bash
       adb pair 192.168.1.42:42999
       # Enter pairing code: 123456
       # Successfully paired to 192.168.1.42:42999 [guid=adb-...-RQAZWM]
       ```

    QR-code pairing is the same protocol, driven by Android Studio's "Pair Devices Using Wi-Fi" UI.

- [ ] **2. Connect to the **separate** debug port shown after pairing.** The pairing port is one-shot; the device exposes a different `IP:PORT` for actual debugging:
    ```bash
    adb connect 192.168.1.42:39555
    adb devices
    # 192.168.1.42:39555  device
    ```
    `adb disconnect [<host:port>]` drops a single host or, with no argument, **all** TCP devices.

- [ ] **3. Use mDNS auto-discovery for hands-off connections.** The ADB server can discover paired devices and auto-connect:
    ```bash
    adb mdns check
    # mdns daemon version [openscreen discovery 0.0.0]

    adb mdns services
    # List of discovered mdns services
    # adb-XXXXXXXX-RQAZWM   _adb-tls-pairing._tcp  192.168.1.42:42999
    # adb-XXXXXXXX-RQAZWM   _adb-tls-connect._tcp  192.168.1.42:39555
    ```
    Service types:
    - `_adb._tcp` — legacy `adb tcpip`.
    - `_adb-tls-pairing._tcp` — the ephemeral pairing port (after tapping "Pair device with pairing code").
    - `_adb-tls-connect._tcp` — the post-pairing debug port. By default these are auto-connected (`ADB_MDNS_AUTO_CONNECT=adb-tls-connect`).

- [ ] **4. Know which mDNS backend is in use.** The doc says Bonjour is the default, but **from ADB v34+ the default flipped to Openscreen on Linux/Windows** (the public doc lags). Force a backend explicitly with the env var when reproducibility matters:
    ```bash
    ADB_MDNS_OPENSCREEN=1 adb start-server     # force Openscreen
    ADB_MDNS_OPENSCREEN=0 adb start-server     # force Bonjour
    ```
    See `../../architecture/understanding-adb-architecture/SKILL.md` for the full env-var table (`ADB_MDNS_AUTO_CONNECT`, `ADB_MDNS_OPENSCREEN`).

- [ ] **5. Use the legacy `adb tcpip` path only when pairing is not available.** Pre-Android-11 (and still supported on 11+ as a fallback):
    ```bash
    # 1. USB-attach the device, accept the RSA dialog, confirm with `adb devices`.
    adb tcpip 5555
    # 2. Unplug USB.
    adb connect 192.168.1.42:5555
    ```
    Why it is the legacy path:
    - Requires physical USB to bootstrap, defeating the "no cable" appeal.
    - `5555` is well-known and exposed without any pairing — anyone on the LAN can reach it until reboot.
    - The device drops back to USB-only on reboot (`adb usb` resets it manually).
    - No mDNS auto-discovery; the user must know the IP.

    The modern `adb pair` flow encrypts and authenticates over TLS using a pairing-code-derived shared secret, and uses ephemeral ports — much safer.

- [ ] **6. Diagnose pairing/connection failures in this order:**
    > "Device not connecting after pairing successfully: `adb` relies on mDNS to discover and automatically connect to paired devices. If your network or device configuration does not support mDNS or has disabled it, then you need to manually connect to the device using `adb connect ip:port`."
    1. **Same Wi-Fi?** Confirm both ends are on the same SSID and the same VLAN/subnet.
    2. **Corporate Wi-Fi?** Verbatim:
       > "Secure Wi-Fi networks, such as corporate Wi-Fi networks, may block p2p connections and not let you connect over Wi-Fi. Try connecting with a cable or another (non-corp) Wi-Fi network."
    3. **mDNS disabled?** Run `adb mdns check`. If "mDNS is not running", fall back to manual `adb connect <ip:port>` using the connect-port from the device's Wireless debugging screen.
    4. **`connection refused` after `adb connect`?** Wireless debugging port closed (toggled off, device rebooted, or pairing port used instead of the connect port). Re-enable wireless debugging; re-pair if needed; copy the `_adb-tls-connect._tcp` port from `adb mdns services`.
    5. **Auto-disconnect after Wi-Fi switch?** Verbatim:
       > "`adb` over Wi-Fi sometimes turns off automatically: This can happen if the device either switches Wi-Fi networks or disconnects from the network. To resolve, re-connect to the network."

- [ ] **7. Treat wireless ADB as a "trust the LAN" feature.**
    - Anyone with network reach plus the pairing code can pair.
    - Once paired, only the cryptographic key gates access — but a flat office LAN means the attack surface is everyone-on-the-Wi-Fi.
    - **Disable wireless debugging when not actively using it.** The Settings toggle kills the listening sockets immediately.
    - Avoid wireless ADB on **public / café / hotel Wi-Fi** entirely.
    - The legacy `adb tcpip` mode is a flat unauthenticated TCP listener — prefer `adb pair` whenever the device supports it.

## Patterns

### Pattern: WRONG vs RIGHT — using `adb tcpip` on Android 13

```bash
# WRONG
adb tcpip 5555
adb connect 192.168.1.100:5555
# WRONG because: on Android 11+ the modern `adb pair` flow is available and
# encrypts/authenticates over TLS. The tcpip path bypasses pairing, exposes a
# well-known unauthenticated TCP listener until reboot, and is often blocked
# on managed Wi-Fi. Many corporate networks drop port 5555 traffic outright.
```

```bash
# RIGHT
# On the device: Settings -> Developer options -> Wireless debugging -> ON
# Then "Pair device with pairing code" — note the displayed IP:PORT and code.
adb pair 192.168.1.100:42999
# Enter pairing code: 123456
# Then connect on the device's separate connect-port (also shown in Wireless debugging):
adb connect 192.168.1.100:39555
adb devices
# 192.168.1.100:39555  device
```

### Pattern: WRONG vs RIGHT — pairing port confused with connect port

```bash
# WRONG
adb pair 192.168.1.100:42999
# (succeeds)
adb connect 192.168.1.100:42999
# error: failed to connect to '192.168.1.100:42999': Connection refused
# WRONG because: the pairing port is one-shot. After successful pairing, the
# device exposes a different IP:PORT (the "Pair devices over Wi-Fi" main screen
# shows it under the device's IP). Trying to connect to the pairing port fails.
```

```bash
# RIGHT
# Read the connect port from the Wireless debugging main screen (NOT the
# "Pair device with pairing code" dialog), or:
adb mdns services
# adb-...  _adb-tls-connect._tcp  192.168.1.100:39555
adb connect 192.168.1.100:39555
```

### Pattern: WRONG vs RIGHT — mDNS Openscreen vs Bonjour expectations

```bash
# WRONG (script makes assumptions about mdnsResponder / Bonjour being default)
# CI script greps for "bonjour" in `adb mdns check` output and aborts when it
# does not find it.
adb mdns check | grep -q bonjour || exit 1
# WRONG because: ADB v34+ defaults to the Openscreen backend on Linux/Windows.
# Output is "mdns daemon version [openscreen discovery 0.0.0]" on those hosts.
# The script breaks on every modern CI runner.
```

```bash
# RIGHT
# Either accept either backend, or pin one with the env var:
ADB_MDNS_OPENSCREEN=1 adb start-server
adb mdns check                       # confirm "openscreen discovery"
# Or for environments that still need Bonjour:
ADB_MDNS_OPENSCREEN=0 adb start-server
```

## Mandatory rules

- **MUST** prefer `adb pair` + `adb connect` on Android 11 (API 30) phones / Android 13 (API 33) TV+Wear. The `adb tcpip` path is the legacy fallback.
- **MUST** use the **connect** port (from `_adb-tls-connect._tcp` or the Wireless debugging main screen) for `adb connect`, not the **pairing** port shown by the pairing-code dialog.
- **MUST** disable wireless debugging on the device when not actively using it. The Settings toggle kills the listening sockets immediately.
- **MUST NOT** assume the mDNS backend is Bonjour. ADB v34+ defaults to Openscreen on Linux/Windows. Pin with `ADB_MDNS_OPENSCREEN=0|1` if reproducibility is required.
- **MUST NOT** run `adb tcpip 5555` on shared/public Wi-Fi. The result is an unauthenticated open TCP listener until reboot.
- **PREFERRED:** plug into the office Ethernet or use a personal hotspot when corporate Wi-Fi blocks p2p.
- **PREFERRED:** `adb mdns check` and `adb mdns services` are the first diagnostic step when wireless debugging "just stops working."

## Verification

- [ ] `adb pair <host:port>` prints `Successfully paired to <host:port>`.
- [ ] `adb connect <host:port>` returns `connected to <host:port>` and `adb devices` lists the IP transport with state `device`.
- [ ] `adb mdns check` prints a daemon version (either Openscreen or Bonjour) without errors.
- [ ] `adb mdns services` lists at least one `_adb-tls-connect._tcp` entry for a paired device.
- [ ] Toggling Wireless debugging off on-device removes the IP transport from `adb devices` within a few seconds.
- [ ] On API 30+ phones, the developer's preferred path is `adb pair` (not `adb tcpip`).
- [ ] When debugging fails, the user has run `adb mdns check` before regenerating the host RSA key.

## References

- Wireless ADB (developer.android.com/tools/adb#wireless): https://developer.android.com/tools/adb#wireless
- ADB user guide root: https://developer.android.com/tools/adb
- Platform-tools release notes (mDNS backend defaults): https://developer.android.com/tools/releases/platform-tools
- AOSP `adb` man page: https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/docs/user/adb.1.md
- `tasks/research/A1-adb-architecture-devices.md` — ADB v34+ Openscreen-vs-Bonjour default flip on Linux/Windows, mDNS service-type taxonomy, and the verbatim "Device not connecting after pairing successfully" guidance.
- `docs/CORPUS.md` §I.4 (wireless ADB).
- Sibling skills:
  - High-level architecture and env vars: `../../architecture/understanding-adb-architecture/SKILL.md`
  - USB device basics: `../connecting-to-devices/SKILL.md`
  - App install/clear: `../../apps/installing-and-managing-apps/SKILL.md`
  - Run instrumented tests over a wireless transport: `../../tests/running-instrumented-tests-via-adb/SKILL.md`
- Cross-set neighbours:
  - Run instrumented tests with `AndroidJUnit4`: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`
  - Configure JUnit4 on Android: `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`
  - Source-set strategy: `../../../fundamentals/strategies/organizing-test-source-sets/SKILL.md`
