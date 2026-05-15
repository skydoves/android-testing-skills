---
name: capturing-preview-screenshots-in-ci
description: Use this skill to render every Jetpack Compose `@Preview` as a screenshot on a real Android device or emulator and publish a browsable HTML catalog from CI. Covers the Compose HotSwan Gradle compiler plugin (`com.github.skydoves.compose.hotswan.compiler`) and `debugImplementation("com.github.skydoves.compose.hotswan:preview")`, the `captureAllPreviews` task, capturing one preview over ADB via `HotSwanPreviewActivity` + `screencap`, the `hotSwanCompiler { preview { renderDelayMs; demoMode; sdkModeEnabled } }` DSL, per-preview `@PreviewScreenshot(renderDelay = …)`, running it on an emulator in GitHub Actions (`reactivecircus/android-emulator-runner` or Gradle Managed Devices) and deploying the catalog to GitHub Pages, and how device rendering differs from host-JVM tools (Paparazzi, Roborazzi). Use when the user mentions "captureAllPreviews", "HotSwan", "preview screenshots in CI", "screenshot catalog", "preview catalog on GitHub Pages", or "Paparazzi vs Roborazzi vs device screenshots".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - screenshot-testing
  - compose-preview
  - hotswan
  - captureAllPreviews
  - preview-catalog
  - github-actions
  - github-pages
  - paparazzi-roborazzi
---

# Capturing Preview Screenshots In CI — A Device-Rendered Catalog On Every Commit

A `@Preview` lives in one engineer's IDE, behind a recompile, on a renderer (layoutlib) that is not the device. This skill turns it into a shared artifact: render every `@Preview` on a real device/emulator (real ART, real fonts, real image loading, system bars), collect the PNGs into a searchable HTML catalog, regenerate it from CI on every commit, and serve it from GitHub Pages. The tool is Compose HotSwan's Gradle plugin. For structuring previews so this is worth doing, see `../developing-with-compose-previews/SKILL.md`.

## When to use this skill

- The user wants `./gradlew captureAllPreviews` — render all `@Preview`s and produce an HTML catalog.
- The user wants to screenshot one preview from a shell script over ADB without the IDE.
- The user wants a preview catalog deployed to GitHub Pages so designers/QA/PMs can browse components at a URL.
- A preview that loads a network image or settles an animation is captured half-painted — needs render-delay tuning.
- The user asks "Paparazzi vs Roborazzi vs device screenshots — which, and can I run both?".

## When NOT to use this skill

- The user wants pixel-diff golden-image regression gates (store a reference PNG, fail on diff). `captureAllPreviews` does not diff; that is Paparazzi's/Roborazzi's job. Run both if you need both — see the comparison table below.
- The user wants behavioral assertions on a Compose UI (`ComposeTestRule`, finders) — use `../../setup/choosing-test-rule-vs-runtest/SKILL.md` and the `compose/finders/` skills.
- The user is fixing why a preview won't render or keeps going "out of date" — that is preview structure; use `../developing-with-compose-previews/SKILL.md`.
- The user wants Compose live hot reload (edit code, see it on a running device) — that is the HotSwan IDE plugin's reload feature, a different surface from screenshot capture.

## Prerequisites

- A connected device or running emulator with the app installed and **launched at least once** in the current run — `captureAllPreviews` drives the *running* app's process so previews render with the real dependency graph (Hilt/Koin, network, DB) and need no mocks.
- The HotSwan Gradle compiler plugin and the preview library (versions from the project's catalog; `1.3.2` at time of writing):

```kotlin
// libs.versions.toml
[plugins]
hotswan-compiler = { id = "com.github.skydoves.compose.hotswan.compiler", version = "1.3.2" }

// root build.gradle.kts
plugins { alias(libs.plugins.hotswan.compiler) apply false }

// app module build.gradle.kts
plugins { alias(libs.plugins.hotswan.compiler) }
dependencies {
    debugImplementation("com.github.skydoves.compose.hotswan:preview:1.3.2")  // debug only — out of release builds
}
```

- For CI: an emulator step (`reactivecircus/android-emulator-runner@v2`) or Gradle Managed Devices (see `../../../instrumentation/managed-devices/running-tests-on-gradle-managed-devices/SKILL.md`). KVM enabled on the runner for an accelerated emulator.

## Workflow

- [ ] **1. Verify why this needs a device.** Host-JVM renderers (Paparazzi, Roborazzi) run layoutlib without a device — fast, but the pixels are an approximation. Device rendering goes through ART: device font metrics, real Coil/Glide loading (placeholder, network image, crossfade all execute), GPU/AGSL/blur on the real path, and the full screen including status and navigation bars. The cost is an emulator in CI. Accept that trade only if catalog fidelity matters; otherwise stick with host-JVM diffing.

- [ ] **2. Capture one preview over ADB (the primitive).** With the debug app built and launched once on the device, the HotSwan preview library exposes `HotSwanPreviewActivity`, which takes the fully-qualified `@Preview` function name as a `composable` string extra:

```bash
adb shell am start -S \
  -n com.example.app/com.skydoves.compose.hotswan.preview.HotSwanPreviewActivity \
  --es composable "com.example.app.feature.home.PokedexHomePreview"
sleep 1                                  # let it render
adb exec-out screencap -p > home-preview.png
```

  Replace the package with the `applicationId` and the extra with the FQN of the preview function. This is the unit every higher-level capture is built on; doing it preview-by-preview does not scale, so use step 3.

- [ ] **2a. (Optional) interactive loop in the IDE.** With the HotSwan IDE plugin installed, the gutter icon next to a `@Preview` launches it on device in under half a second with no rebuild, and resolves `@PreviewParameter` providers via reflection. The IDE plugin is only needed for that interactive flow and for live hot reload — `captureAllPreviews` below needs only the Gradle plugin.

- [ ] **3. Capture every preview at once.** With the app running on a connected device:

```bash
./gradlew captureAllPreviews
```

  The task: scans every `.kt` under `src/` for `@Preview` functions and resolves each one's FQN and Gradle module; enables Android System UI Demo Mode so the clock/battery/signal are pinned (diffs stay about your UI, not the time of day); for each preview launches `HotSwanPreviewActivity`, waits the render delay, runs `screencap`; restores the status bar and returns the device to the app's main Activity; generates `index.html`. Everything lands in `.hotswan/preview-captures/` at the project root. The catalog has search, module grouping, dark/light toggle, fullscreen view, and per-shot device model + timestamp. With `sdkModeEnabled = true` it also traces each preview to the composable it wraps and renders a KDoc + parameter table (a design-system reference that cannot drift). No test code anywhere — the task reads `@Preview` annotations directly.

- [ ] **4. Tune the render delay: small global default, per-preview overrides.** A shot is taken a fixed number of ms after the composable launches. Too short and a screen loading a network image / running a query / starting an animation is captured half-painted; too long and a big catalog wastes minutes. Set a low global default near the floor and pay extra only where it is earned:

```kotlin
// app module build.gradle.kts
hotSwanCompiler {
    preview {
        renderDelayMs.set(1000L)        // ships at 2500L; most previews are static layout — keep the baseline low
        demoMode.set(true)              // pin status bar for deterministic shots
        sdkModeEnabled.set(false)       // true => KDoc + parameter tables in the catalog (design systems)
    }
}
```

```kotlin
// per-preview override for the slow minority (network/DB/animation): 3000–5000 ms is typical
@Preview
@PreviewScreenshot(renderDelay = 4000)
@Composable
private fun PokemonDetailPreview() { AppTheme { PokemonDetail(pokemon = samplePokemon) } }
```

  A preview with no `@PreviewScreenshot` falls back to the global value, so fast stays fast. If most of your previews are data-loading feature screens, the other valid choice is a higher global and overrides on the static minority — match the baseline to the majority.

- [ ] **5. Run it in GitHub Actions on an emulator.** `captureAllPreviews` is a normal Gradle task driving ADB, so it runs anywhere a managed Android device exists. The app must be installed and launched before the task runs:

```yaml
# .github/workflows/preview-screenshots.yml
name: Preview Screenshots
on:
  workflow_dispatch:
  push: { branches: [ main ], paths: [ '**/*.kt', '**/*.xml' ] }
permissions: { contents: write, pages: write, id-token: write }
concurrency: { group: preview-screenshots, cancel-in-progress: true }

jobs:
  capture:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: zulu, java-version: 17 }
      - name: Enable KVM
        run: |
          echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' | sudo tee /etc/udev/rules.d/99-kvm4all.rules
          sudo udevadm control --reload-rules && sudo udevadm trigger --name-match=kvm
      - run: ./gradlew :app:assembleDebug
      - run: rm -rf .hotswan/preview-captures        # so a deleted preview leaves the catalog
      - name: Run emulator and capture
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 31
          arch: x86_64
          profile: pixel_6
          emulator-options: -no-snapshot-save -no-window -gpu swiftshader_indirect -noaudio -no-boot-anim
          disable-animations: true
          script: |
            adb install -r app/build/outputs/apk/debug/app-debug.apk
            adb shell am start -n com.example.app/.MainActivity
            sleep 10
            ./gradlew :app:captureAllPreviews
      - uses: actions/upload-artifact@v4
        if: always()                                  # keep whatever rendered even if one preview throws
        with: { name: preview-screenshots, path: .hotswan/preview-captures/, retention-days: 30 }
```

  (Gradle Managed Devices is the equivalent if you prefer Google's built-in option — see the GMD skill. The runner action is just less config for a single device.)

- [ ] **6. Deploy the catalog to GitHub Pages.** A zip you download is not a team asset; a URL is. A second job publishes the captured directory:

```yaml
  deploy:
    needs: capture
    runs-on: ubuntu-latest
    environment: { name: github-pages, url: '${{ steps.deployment.outputs.page_url }}' }
    steps:
      - uses: actions/download-artifact@v4
        with: { name: preview-screenshots, path: preview-captures }
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with: { path: preview-captures }
      - id: deployment
        uses: actions/deploy-pages@v4
```

  Push to `main` → emulator boots → every `@Preview` is rendered and screenshotted → the HTML catalog is built → Pages serves the new version.

## Patterns

### Pattern: capturing in CI without the app running

```bash
# WRONG — captureAllPreviews on a fresh emulator with nothing installed
- uses: reactivecircus/android-emulator-runner@v2
  with:
    api-level: 31
    script: ./gradlew :app:captureAllPreviews
# WRONG because: HotSwanPreviewActivity renders inside the *running app's* process. With no APK
# installed and no Activity started, every launch fails and the catalog comes out empty.
```

```bash
# RIGHT — build, install, launch, settle, then capture
    script: |
      adb install -r app/build/outputs/apk/debug/app-debug.apk
      adb shell am start -n com.example.app/.MainActivity
      sleep 10
      ./gradlew :app:captureAllPreviews
```

### Pattern: one global render delay set high "to be safe"

```kotlin
// WRONG
hotSwanCompiler { preview { renderDelayMs.set(5000L) } }
// WRONG because: every preview now costs 5 s. A 60-preview catalog wastes ~4 extra minutes per run
// to accommodate the handful that actually load remote data.
```

```kotlin
// RIGHT — low global, opt into the cost where the pixels need it
hotSwanCompiler { preview { renderDelayMs.set(1000L); demoMode.set(true) } }

@Preview @PreviewScreenshot(renderDelay = 4000) @Composable
private fun NetworkImagePreview() { AppTheme { UserAvatar(imageUrl = "https://example.com/p.jpg") } }
```

### Pattern: treating `captureAllPreviews` as a regression gate

```kotlin
// WRONG — expecting captureAllPreviews to fail the build on a visual change
- run: ./gradlew captureAllPreviews   # then... assume it diffs against last run? it does not.
// WRONG because: captureAllPreviews produces a catalog; it does not store golden images or pixel-diff.
// Nothing fails when a screenshot changes.
```

```kotlin
// RIGHT — pick the tool for the job; run both if you need both
// Pixel-diff regression gate, no emulator, fast: Paparazzi or Roborazzi (test class per composable, golden PNGs).
// Browsable device-rendered catalog for docs/design review: captureAllPreviews.
// Many projects run Paparazzi/Roborazzi on PRs as a gate AND captureAllPreviews on main for the catalog.
```

## HotSwan `captureAllPreviews` vs Paparazzi / Roborazzi

| Aspect | `captureAllPreviews` (HotSwan) | Paparazzi / Roborazzi |
|---|---|---|
| Rendering | Real device or emulator (ART) | Host JVM, layoutlib |
| System bars | Included (status + navigation) | Not included |
| Runtime behavior | Full — network, image loading, DB, DI all execute | UI shell only |
| Test code required | None — scans `@Preview` directly | One test class per composable |
| Output | PNGs + searchable HTML catalog (module grouping, KDoc, params) | PNGs |
| Pixel-diff regression | Not built in | Yes — the core feature |
| CI setup | Emulator required (GitHub Actions / GMD supported) | No emulator needed |
| Best for | Visual cataloging, documentation, design review | Golden-image regression testing |

## Mandatory rules

- **MUST** build, install, and launch the debug app on the device/emulator **before** `captureAllPreviews` — it renders in the running app's process.
- **MUST** keep the preview library on `debugImplementation` so `HotSwanPreviewActivity` never ships in release builds.
- **MUST** set a low global `renderDelayMs` and use `@PreviewScreenshot(renderDelay = …)` only on previews that load remote data or settle animations — not a blanket high default.
- **MUST** clear `.hotswan/preview-captures/` before a CI run so deleted previews disappear from the catalog instead of lingering, and use `if: always()` on the artifact upload.
- **MUST NOT** treat `captureAllPreviews` as a pixel-diff regression gate — it has no golden-image comparison. Pair it with Paparazzi/Roborazzi if you need that.
- **MUST NOT** rely on host-JVM screenshot tools (Paparazzi/Roborazzi) when the screenshot must match device fonts, real image loading, GPU/AGSL effects, or system chrome — those need device rendering.
- **PREFERRED:** enable `demoMode` so the status bar is deterministic across shots; enable `sdkModeEnabled` for design-system modules so the catalog carries KDoc + parameter tables.
- **PREFERRED:** deploy the catalog to GitHub Pages so it has a stable URL non-engineers can open.

## Verification

- [ ] `./gradlew captureAllPreviews` with the app running produces `.hotswan/preview-captures/index.html` and one PNG per `@Preview`.
- [ ] The single-preview ADB recipe (`am start … HotSwanPreviewActivity --es composable <FQN>` + `screencap`) returns a non-empty PNG of that composable.
- [ ] `debugImplementation("com.github.skydoves.compose.hotswan:preview:…")` (not `implementation`/`releaseImplementation`); a release build does not contain `HotSwanPreviewActivity`.
- [ ] The CI job installs + launches the app before invoking `captureAllPreviews`; the uploaded artifact is non-empty on a green run.
- [ ] The GitHub Pages deploy job runs `needs: capture` and the published URL shows the latest catalog after a push to `main`.
- [ ] Slow previews (network/DB/animation) carry `@PreviewScreenshot(renderDelay = …)`; the global `renderDelayMs` is near the floor for the static majority.

## References

- hotswan.dev/blog/compose-preview-screenshots-ci — skydoves, "Compose Preview Screenshots in CI: A Real Device Catalog on Every Commit" (single-preview ADB capture, `captureAllPreviews`, render-delay strategy, GitHub Actions + emulator-runner, GitHub Pages deploy, Paparazzi/Roborazzi comparison).
- hotswan.dev/blog/compose-preview-driven-development — skydoves, "Compose Preview Driven Development with Instant Feedback" (Preview Runner, `captureAllPreviews`, why device rendering, CI workflow).
- github.com/skydoves/compose-hotswan — the HotSwan Gradle compiler plugin (`com.github.skydoves.compose.hotswan.compiler`), the `preview` artifact, `HotSwanPreviewActivity`, the `captureAllPreviews` task, and the `hotSwanCompiler { preview { … } }` / `@PreviewScreenshot` API.
- github.com/skydoves/pokedex-compose — a reference project running this exact setup (the `screenshot.yml` workflow and the published preview catalog).
- developer.android.com/develop/ui/compose/tooling/previews — `@Preview` and `@PreviewParameter` (the annotations `captureAllPreviews` scans).
- github.com/ReactiveCircus/android-emulator-runner — the GitHub Actions emulator step used to host a real Android image in CI.
- Sibling skill: `../developing-with-compose-previews/SKILL.md` — structuring `@Preview` functions so capturing them is worthwhile (state hoisting, `@PreviewParameter`, anti-patterns).
- Cross-set: `../../../instrumentation/managed-devices/running-tests-on-gradle-managed-devices/SKILL.md` — the Gradle Managed Devices alternative to `android-emulator-runner` for hosting the device in CI.
