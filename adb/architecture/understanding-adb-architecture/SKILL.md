---
name: understanding-adb-architecture
description: Use this skill to reason about the three-piece ADB topology (client CLI, host server on TCP 5037, on-device daemon `adbd`), the lifecycle commands `adb start-server` / `adb kill-server` / `adb reconnect`, ADB environment variables (`ADB_TRACE`, `ADB_VENDOR_KEYS`, `ANDROID_ADB_SERVER_PORT`, `ANDROID_SERIAL`, `ADB_LOCAL_TRANSPORT_MAX_PORT`, `ADB_MDNS_AUTO_CONNECT`, `ADB_MDNS_OPENSCREEN`, `ADB_LIBUSB`, `ADB_BURST_MODE`), the host RSA key pair under `~/.android/`, the server log location, and version mismatches between Android Studio's bundled `platform-tools` and a system-installed `adb`. Use when the user mentions `daemon not running; starting now`, `server version doesn't match`, port 5037 collisions, ADB_TRACE, vendor keys, mDNS Openscreen vs Bonjour, libusb regressions, "adb is being weird", or asks "what does adb actually do".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - adb
  - android-debug-bridge
  - platform-tools
  - adbd
  - port-5037
  - ADB_TRACE
  - ADB_VENDOR_KEYS
  - server-version-mismatch
  - rsa-adbkey
  - libusb-backend
---

# Understanding ADB Architecture — Client, Server, Daemon

ADB is one binary that wears three hats. Most "adb is being weird" reports come from misunderstanding which hat is misbehaving (the local CLI, the long-lived host server on port 5037, or the on-device `adbd`), or from two different `adb` binaries fighting over the same port. This skill grounds the mental model so the rest of the ADB skill set has a stable foundation.

## When to use this skill

- The user sees `* daemon not running; starting now at tcp:5037 *` and assumes it is an error.
- The user sees `adb server version (XX) doesn't match this client (YY); killing...` after Studio updates its bundled platform-tools.
- The user wants to enable verbose ADB logging (`ADB_TRACE`) or set up a CI runner with a preinstalled vendor key.
- The user asks why deleting `~/.android/adbkey` breaks every other paired device.
- The user is debugging port 5037 collisions, multiple adb installs, or libusb-related transport failures on Linux.

## When NOT to use this skill

- The user is troubleshooting a specific connected device (USB authorization, `unauthorized`, `no permissions`) — use `../../devices/connecting-to-devices/SKILL.md`.
- The user is setting up wireless / Wi-Fi ADB — use `../../devices/connecting-over-wifi/SKILL.md`.
- The user is running `adb shell am instrument` for tests — use `../../tests/running-instrumented-tests-via-adb/SKILL.md`.

## Prerequisites

- Android SDK Platform-Tools installed (path resolution rules below).
- Shell access to a workstation with `adb` on `$PATH`.
- For diagnostic flows: write access to `$TMPDIR` (macOS/Linux) or `%TEMP%` (Windows).

## Workflow

- [ ] **1. Internalize the three-piece model.** From developer.android.com/tools/adb (Overview):
    - **Client (`adb` CLI)** — invoked from the terminal. Serializes commands and ships them to the server over a local TCP socket.
    - **Server (`adb-server`)** — host-side background process. Multiplexes commands from many clients, tracks connected transports, forwards traffic to `adbd`. Binds **TCP port 5037** on `localhost` by default.
    - **Daemon (`adbd`)** — runs on the device. Spawned by `init` on userdebug/eng builds, and by the system on user builds. Receives commands from the host server over USB or TCP and executes them in the device's userspace.

- [ ] **2. Recognize the implicit `start-server` print.** The first `adb` command on a fresh shell emits:
    ```
    * daemon not running; starting now at tcp:5037
    * daemon started successfully
    ```
    This is **informational, not an error** — `adb` auto-starts the server when none is running. Skipping over this fact wastes hours of debugging.

- [ ] **3. Use the lifecycle commands surgically:**
    | Command | Effect |
    |---|---|
    | `adb start-server` | Ensures a server is running on port 5037. If a compatible server is already running, it's a no-op (idempotent). If a different process is bound to 5037 (or an older incompatible adb server), `start-server` fails — see the port-collision pattern below. |
    | `adb kill-server` | Terminates the local server. The next `adb` command respawns it. |
    | `adb reconnect` | "Kick connection from host side to force reconnect." Useful for a stuck USB transport. |
    | `adb reconnect device` | "Kick connection from device side." Asks `adbd` to drop and re-handshake. |
    | `adb reconnect offline` | "Reset offline/unauthorized devices to force reconnect." Targets only the misbehaving transports. |
    | `adb -P PORT ...` | Override the server port (default 5037). Same as `ANDROID_ADB_SERVER_PORT`. |

    Stuck-transport recovery recipe:
    ```bash
    adb reconnect offline
    # if that fails:
    adb kill-server && adb start-server && adb devices
    ```

- [ ] **4. Resolve the binary path explicitly.** `adb` ships in **Android SDK Platform-Tools** (`android_sdk/platform-tools/`). Check what is actually loaded:
    ```bash
    adb --version
    which -a adb         # find shadowed copies
    ```
    Sample `adb --version` output decoded:
    ```
    Android Debug Bridge version 1.0.41         # wire-protocol version (stable for years)
    Version 35.0.2-12147458                      # platform-tools release + Google build number
    Installed as /opt/homebrew/bin/adb           # the actual file the OS resolved
    Running on Darwin 24.6.0 (arm64)
    ```
    The wire-protocol number is what matters for cross-compatibility: client and server must agree, or the first command kills the older server with `adb server version (XX) doesn't match this client (YY); killing...`.

- [ ] **5. Pick ONE canonical adb when Android Studio is involved.** Studio bundles and **auto-updates** its own `platform-tools/`. If a system / Homebrew adb is also on `$PATH`, three failure modes appear:
    1. **Version skew** — first command issued by either kills the running server.
    2. **Two competing servers** ping-ponging port 5037.
    3. **Different default backends** (libusb vs. native) producing inconsistent device enumeration.
    Resolution rule of thumb: `export PATH="$ANDROID_HOME/platform-tools:$PATH"` so CLI and Studio agree, then `adb kill-server` once.

- [ ] **6. Know the environment variables (verbatim, from `adb help`):**
    | Variable | Purpose |
    |---|---|
    | `ADB_TRACE` | Comma/space-separated debug categories. Tokens: `all,adb,sockets,packets,rwx,usb,sync,sysdeps,transport,jdwp,services,auth,fdevent,shell,incremental`. |
    | `ADB_VENDOR_KEYS` | Colon-separated paths to RSA key files/dirs. Use on CI to skip the "Allow USB debugging?" prompt. |
    | `ANDROID_SERIAL` | Default device serial (equivalent to `-s`). Overridden by `-s`. |
    | `ANDROID_ADB_SERVER_PORT` | Override the server port (default 5037). Equivalent to `-P PORT`. |
    | `ADB_LOCAL_TRANSPORT_MAX_PORT` | Highest odd port scanned for emulators. Default `5585` (16 emulators). |
    | `ADB_MDNS_AUTO_CONNECT` | CSV of mDNS service types eligible for auto-connect. Default: `adb-tls-connect`. |
    | `ADB_MDNS_OPENSCREEN` | `1`/`0`. Force the Openscreen mDNS backend. **Default Openscreen on Linux/Windows from ADB v34+ — the public doc lags.** |
    | `ADB_LIBUSB` | `1`/`0`. Force the libusb USB backend. |
    | `ADB_BURST_MODE` | `1`/`0`. Pipeline packets without ACK. Experimental, ADB 36.0.0+. |

    Diagnostic recipe with maximum verbosity:
    ```bash
    ADB_TRACE=all adb devices -l 2>~/adb-trace.log
    ```
    CI runner with a preinstalled key and custom port:
    ```bash
    export ADB_VENDOR_KEYS=/etc/adb/ci-key
    export ANDROID_ADB_SERVER_PORT=5038
    export ADB_LOCAL_TRANSPORT_MAX_PORT=5617
    adb start-server
    ```

- [ ] **7. Find the server log when something is wrong.** The server keeps a persistent log distinct from per-command `ADB_TRACE` output and from `logcat`:
    | OS | Path |
    |---|---|
    | macOS | `$TMPDIR/adb.$UID.log` (e.g. `/var/folders/.../T/adb.501.log`) |
    | Linux | `$TMPDIR/adb.$UID.log` (e.g. `/tmp/adb.1000.log`) |
    | Windows | `%TEMP%\adb.log` |
    Useful flow:
    ```bash
    adb kill-server
    ADB_TRACE=adb,transport,auth adb start-server
    adb devices -l
    ls -lh "$TMPDIR/adb.$UID.log"
    ```

- [ ] **8. Treat `~/.android/` as load-bearing.** RSA keypair: `~/.android/adbkey` (private), `~/.android/adbkey.pub` (public). Generated on first `adb` invocation. Every accepted "Allow USB debugging?" dialog appends this public key to `/data/misc/adb/adb_keys` on the device. Wiping the local key invalidates every device the user previously authorized — see the WRONG/RIGHT pair below.

- [ ] **9. Accept the libusb v36 regression on Linux as background context.** ADB v36.0.0 (Apr 2025) shipped a rewritten libusb backend with hot-plug. ADB v36.0.2 (Sep 2025) **reverted libusb as the Linux default** because of instability. CI runners pinned to specific platform-tools versions need to know which side of that revert they sit on; force the backend explicitly with `ADB_LIBUSB=0` or `=1` if reproducibility matters.

## Patterns

### Pattern: WRONG vs RIGHT — recovering from `unauthorized` or `offline` device

```bash
# WRONG
rm -rf ~/.android/adbkey ~/.android/adbkey.pub
adb kill-server
adb start-server
# WRONG because: this rotates the host RSA keypair. Every previously authorized
# device now sees an unknown public key and re-prompts the "Allow USB debugging?"
# dialog. CI runners and headless test farms break silently because nobody is
# there to tap "Allow". Worse, the key on /data/misc/adb/adb_keys for the old
# pubkey lingers as orphan data.
```

```bash
# RIGHT
adb reconnect offline           # try a targeted reset first
adb devices                     # confirm the state changed
# if still offline:
adb kill-server && adb start-server
# if still unauthorized: unlock the device, re-tap "Allow USB debugging?"
```

The keypair only needs regenerating when it is genuinely lost or corrupted; in that case, `adb keygen ~/.android/adbkey` rebuilds it deliberately and the user accepts that every device must re-authorize.

### Pattern: WRONG vs RIGHT — two competing adb installs

```bash
# WRONG
brew install android-platform-tools     # installs adb to /opt/homebrew/bin
# Studio also auto-updates ~/Library/Android/sdk/platform-tools/adb
# PATH puts /opt/homebrew/bin first.
# Studio's "Run" button uses ~/Library/Android/sdk/platform-tools/adb (different version).
# First command issued by either side prints:
#   adb server version (41) doesn't match this client (40); killing...
# WRONG because: every Run-button invocation kills the server you started,
# yanks transports out from under Logcat, and reissues "daemon not running".
```

```bash
# RIGHT
# Pick one canonical adb. Easiest: prefer Studio's bundled platform-tools.
export ANDROID_HOME="$HOME/Library/Android/sdk"
export PATH="$ANDROID_HOME/platform-tools:$PATH"
adb kill-server                  # clear stale state once
adb --version                    # confirm path matches Studio's
```

### Pattern: WRONG vs RIGHT — port 5037 already in use

```bash
# WRONG
# Some other tool (nc, a proxy, a stale adb-server zombie) is bound to 5037.
adb start-server   # silently fails; subsequent adb devices hangs.
```

```bash
# RIGHT
lsof -nP -iTCP:5037 -sTCP:LISTEN     # find the holder (macOS/Linux)
# kill the squatter, OR run a second adb server on a different port:
ANDROID_ADB_SERVER_PORT=5038 adb start-server
ANDROID_ADB_SERVER_PORT=5038 adb devices
```

## Mandatory rules

- **MUST** treat `* daemon not running; starting now at tcp:5037 *` as informational, not an error.
- **MUST** keep ONE `adb` binary first on `$PATH`. Mixing Studio-bundled and system-installed `adb` of different versions causes server ping-pong.
- **MUST** preserve `~/.android/adbkey` and `~/.android/adbkey.pub` across machine moves; wiping them silently breaks every previously authorized device.
- **MUST NOT** delete `~/.android/adbkey*` as a "fix" for `unauthorized` or `offline` devices — use `adb reconnect offline` first.
- **MUST NOT** run two `adb` servers on port 5037 simultaneously. Use `ANDROID_ADB_SERVER_PORT` (or `-P PORT`) for sharded CI.
- **PREFERRED:** put `export PATH="$ANDROID_HOME/platform-tools:$PATH"` in the shell profile so the CLI and Studio resolve the same binary.
- **PREFERRED:** when reporting a transport bug, attach the server log from `$TMPDIR/adb.$UID.log` (macOS/Linux) or `%TEMP%\adb.log` (Windows) plus an `ADB_TRACE=adb,transport,auth` reproduction.

## Verification

- [ ] `adb --version` prints a single binary path; `which -a adb` lists no shadowed copies.
- [ ] `adb start-server` is idempotent — running it twice does not change anything.
- [ ] `adb kill-server && adb start-server && adb devices` recovers a stuck transport.
- [ ] `~/.android/adbkey` and `~/.android/adbkey.pub` exist and are not world-readable (`chmod 600 ~/.android/adbkey`).
- [ ] `ADB_TRACE=adb adb devices -l` produces verbose transport logs without errors.
- [ ] If running multiple adb servers, each has a distinct `ANDROID_ADB_SERVER_PORT` value and `lsof -iTCP:5037` shows exactly one (or zero) listeners.
- [ ] Server log file exists at `$TMPDIR/adb.$UID.log` (macOS/Linux) or `%TEMP%\adb.log` (Windows) and is recent.

## References

- ADB user guide (architecture and lifecycle): https://developer.android.com/tools/adb
- Platform-tools release notes (version deltas, libusb default): https://developer.android.com/tools/releases/platform-tools
- AOSP `adb` man page (canonical env-var reference): https://android.googlesource.com/platform/packages/modules/adb/+/refs/heads/main/docs/user/adb.1.md
- Studio + AGP testing on the command line: https://developer.android.com/studio/test/command-line
- Source-of-truth research notes (CORPUS §I, A1 report):
  - `tasks/research/A1-adb-architecture-devices.md` — three-piece model, env-var table, libusb v36 revert, Openscreen-vs-Bonjour default flip in ADB v34+, server log paths, RSA keypair handling.
  - `docs/CORPUS.md` §I.2 (three-piece architecture) and §I.10 (critical findings: `force-stop` ≠ `pm clear`, exit codes only with `-w`).
- Sibling skills:
  - Connect a single device: `../../devices/connecting-to-devices/SKILL.md`
  - Wireless ADB (Android 11+): `../../devices/connecting-over-wifi/SKILL.md`
  - Install / manage apps: `../../apps/installing-and-managing-apps/SKILL.md`
  - Run instrumented tests via `am instrument`: `../../tests/running-instrumented-tests-via-adb/SKILL.md`
- Cross-set neighbours:
  - Configure JUnit4 on Android: `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`
  - Run instrumented tests with `AndroidJUnit4`: `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`
  - Source-set strategy: `../../../fundamentals/strategies/organizing-test-source-sets/SKILL.md`
