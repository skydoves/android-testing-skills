# Android Testing Skills

<img width="1116" height="444" alt="cover" src="https://github.com/user-attachments/assets/0cb8280f-aa4c-425a-95a6-c17995e6b33d" />

A library of [Agent Skills](https://agentskills.io) for **Android testing**, covering the full
testing surface that an Android engineer touches, from unit-test fundamentals to ADB-driven E2E.
The skills are written for AI coding agents (Claude Code, Android Studio Agent mode, Gemini) but
read fine for humans too: every API claim is grounded in an `androidx/` file path or an
authoritative external doc, and every behavior note is sourced from primary research.

54 skills across 7 sets:

| Set | Skills | What it covers |
|---|---:|---|
| [`compose/`](compose/) | 25 | Jetpack Compose UI testing: finders, semantics matchers, assertions, gesture/key injection, the test clock, idling resources, state restoration, Espresso interop, accessibility checks, audit, preview-driven development, on-device preview screenshot catalogs in CI, stability validation. |
| [`fundamentals/`](fundamentals/) | 5 | Testing pyramid, what to test, test doubles (fake/stub/mock/spy/dummy), Given-When-Then strategies, source set organisation. |
| [`kotlin/`](kotlin/) | 1 | Kotlin-native test tooling: the multiplatform `kotlin.test` library (`@Test`/`@BeforeTest`, `assert*`, `assertFailsWith`, `Asserter`, the `kotlin("test")` JUnit4/JUnit5 capability). |
| [`jvm-tests/`](jvm-tests/) | 6 | JVM unit tests: JUnit4 setup, Mockito + mockito-kotlin, MockK, kotlinx-coroutines-test (`runTest`), Turbine for flows, Robolectric. |
| [`instrumentation/`](instrumentation/) | 6 | On-device tests: `AndroidJUnit4` runner, `ActivityScenario`, `FragmentScenario`, Espresso, UiAutomator, Gradle Managed Devices. |
| [`platform/`](platform/) | 1 | Platform-shipped test tooling: migrating off the deprecated `android.test.*` classes (`ActivityInstrumentationTestCase2`, `AndroidTestCase`, `ServiceTestCase`, `MoreAsserts`, `TouchUtils`, `android.test.mock.*`) onto AndroidX Test. |
| [`adb/`](adb/) | 10 | ADB-driven E2E: architecture, USB & wireless connection, app management, `am instrument`, input injection, screencap/screenrecord, logcat, file transfer, CI scripting. |

> Another skills: [**compose-performance-skills**](https://github.com/skydoves/compose-performance-skills)
> covers the *performance* side of Compose: stability, recomposition, lazy layouts, modifiers, side
> effects, measurement, baseline profiles, R8, hot reload. Same authoring spec; install both for full
> coverage. See [Attribution](#attribution) for details.

## 💝 Sponsors

The sponsors listed below made it possible for this project to be released as open source. Many thanks to all of them for their support!

<a href="https://cline.gg/jaewoong"><img src="art/logo-cline.png" alt="Cline" width="280"/></a>

**[Cline](https://cline.gg/jaewoong)** is an open-source autonomous coding agent, and with Cline 2.0 the runtime that powers it across VS Code, JetBrains, and the CLI is now a programmable TypeScript SDK (`npm i @cline/sdk`). The agent loop is decoupled from providers (Anthropic, OpenAI, Google, Bedrock, and more) and extended via plugins, custom tools, hooks, and hub-backed sessions that outlive a single UI: inspectable, forkable, and built to be built upon.

## What is a Skill

A Skill is a single Markdown file (`SKILL.md`) plus optional `references/` material that teaches
an agent how to perform one focused task. It declares trigger vocabulary in YAML frontmatter and a
numbered workflow in the body. The agent reads the frontmatter, decides whether the skill applies
to the current task, then follows the workflow inside `SKILL.md` step by step.

A skill is not documentation for humans. It is operational instructions for an LLM. That changes
the writing voice: terse, imperative, RIGHT and WRONG snippet pairs, MUST and MUST NOT directives
in bold caps, and a Verification checklist that proves the work was done. Prose is kept short.
Code samples carry the load.

## Directory layout

```
android-testing-skills/
├── README.md                     # this file
├── INDEX.md                      # symptom + API + workflow lookup tables
├── LICENSE                       # Apache-2.0
├── docs/
│   ├── SPEC.md                   # authoring spec (frontmatter, voice, RIGHT/WRONG, post-write self-check)
│   └── CORPUS.md                 # research corpus (file paths, API tables, primary URLs)
├── scripts/install-skills.sh     # symlinks each skill into the agent's flat skills dir
├── tasks/research/               # 11 deep research reports (gitignored)
└── <set>/<category>/<slug>/
    ├── SKILL.md                  # the skill (required)
    └── references/               # optional, one level deep
```

Conventions:

- **Set**: lowercase, one of `compose`, `fundamentals`, `kotlin`, `jvm-tests`, `instrumentation`, `platform`, `adb`.
- **Category**: lowercase, varies per set (e.g. `compose/setup`, `adb/architecture`, `fundamentals/concepts`).
- **Slug**: gerund form, lowercase, hyphenated, max 64 chars. Unique repo-wide.
- **Body length**: every skill stays under 500 lines. Longer material moves into `references/`.
- **Cross links**: relative Markdown links. Same category uses `..`; different category in same set uses `../..`; different set uses `../../..`. See [`docs/SPEC.md`](docs/SPEC.md) §7.

## Installation

Skill loaders (Claude Code, Android Studio Agent mode, Gemini) expect a **flat layout** of
`<install-dir>/<slug>/SKILL.md`. This repo organizes skills under `<set>/<category>/<slug>/SKILL.md`
for human readability. A plain `git clone` into `~/.claude/skills` will NOT surface the skills
directly. Use the bundled install script, which symlinks each skill folder by its slug into a flat
target directory.

> Why this matters. If you skip the install script, the agent loader sees `compose/`, `fundamentals/`, etc. as
> top-level entries and never finds any `SKILL.md`. The script flattens the tree. **Always run
> it after cloning** and after every `git pull`, since new skills only become discoverable
> once their slugs are symlinked.

### Claude Code

```bash
git clone https://github.com/skydoves/android-testing-skills.git \
  ~/.claude/skills-sources/android-testing-skills

~/.claude/skills-sources/android-testing-skills/scripts/install-skills.sh
```

The script is idempotent. It walks the repo, finds every `SKILL.md`, and creates one symlink per
skill at `~/.claude/skills/<slug>/`. It does NOT touch other entries in `~/.claude/skills/`, so it
coexists with `android skills add ...` (Google's first-party catalog) and with other community
libraries.

```bash
~/.claude/skills-sources/android-testing-skills/scripts/install-skills.sh /path/to/agent/skills
~/.claude/skills-sources/android-testing-skills/scripts/install-skills.sh --uninstall
```

After install, restart Claude Code so it re-scans the skills directory.

### Android Studio Agent mode and Gemini

These agents discover skills at runtime by scanning project-local directories per Google's
[Android skills documentation](https://developer.android.com/tools/agents/android-skills). Use the
same install script, targeting the project's agent skills directory:

```bash
cd <your-android-project>
git clone https://github.com/skydoves/android-testing-skills.git \
  .android-testing-skills-source

./.android-testing-skills-source/scripts/install-skills.sh .agent/skills
```

Add `.android-testing-skills-source/` to `.gitignore` so the project doesn't track upstream.
Updates are a single `git -C .android-testing-skills-source pull && ./.android-testing-skills-source/scripts/install-skills.sh .agent/skills`.

End-to-end behavior with Android Studio Agent mode and Gemini has not been independently verified
by the author. Reports of working or broken integration are welcome at the
[issue tracker](https://github.com/skydoves/android-testing-skills/issues).

### Android CLI (first-party catalog only)

The [Android CLI](https://developer.android.com/tools/agents/android-cli) manages only Google's
first-party skill catalog (`android skills add --skill r8-analyzer`, etc.). Community libraries
such as this repo are outside that catalog and cannot be installed through the CLI. Use both
together: `android skills add` for the first-party catalog, `install-skills.sh` for this library.

### Claude.ai and the Claude API

For Claude.ai workspaces, upload the relevant `SKILL.md` files as Agent Skill attachments. For
direct Anthropic API integrations, reference skills inline in the system prompt or load them
through the Agent Skills file API. The skills are self-contained: no external runtime is required
beyond a working Android toolchain on the developer's machine for the verification steps.

### Other agent runtimes

Any agent runtime that can read a Markdown file and follow its instructions can use these skills.
The frontmatter is plain YAML. The body is plain Markdown. Treat the `SKILL.md` files as system
prompts for the task, attach the linked `references/` files when the body cites them, and let the
agent run the Verification checklist before declaring the work done.

## Quickstart examples

### Diagnose a flaky animation test

> My LazyColumn animation test passes locally but fails on CI. There's a `Thread.sleep(500)` mid-test.

The agent matches on `flaky`, `animation`, `Thread.sleep`, `LazyColumn` and loads
[`compose/synchronization/testing-animations-deterministically/SKILL.md`](compose/synchronization/testing-animations-deterministically/SKILL.md),
which walks Claude through the `mainClock.autoAdvance = false` recipe and chains into
[`compose/synchronization/synchronizing-with-idle/SKILL.md`](compose/synchronization/synchronizing-with-idle/SKILL.md)
to swap `Thread.sleep` for `mainClock.advanceTimeBy(durationMs)`.

### Stand up a JVM test suite for a new module

> I'm adding unit tests to a Kotlin library module. Pick the right runner, mock framework, and coroutine harness.

The agent loads [`jvm-tests/runner/configuring-junit4-on-android/SKILL.md`](jvm-tests/runner/configuring-junit4-on-android/SKILL.md)
for the AndroidX dep matrix and `AndroidJUnit4` runner choice, then chains into
[`jvm-tests/mocking/mocking-with-mockito/SKILL.md`](jvm-tests/mocking/mocking-with-mockito/SKILL.md)
(or `mocking-with-mockk`) and [`jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md`](jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md)
for the coroutine test harness.

### Wire ADB into a CI pipeline

> Run instrumented tests across 4 emulators in parallel and capture screenshots + logcat on failure.

The agent loads [`adb/automation/scripting-adb-for-ci/SKILL.md`](adb/automation/scripting-adb-for-ci/SKILL.md)
for the parallel-fan-out idiom, [`adb/tests/running-instrumented-tests-via-adb/SKILL.md`](adb/tests/running-instrumented-tests-via-adb/SKILL.md)
for the `am instrument -w -r -e numShards/-e shardIndex` invocation, and
[`adb/capture/capturing-screenshots-and-screenrecord/SKILL.md`](adb/capture/capturing-screenshots-and-screenrecord/SKILL.md)
+ [`adb/observability/extracting-logs-with-logcat/SKILL.md`](adb/observability/extracting-logs-with-logcat/SKILL.md)
for the capture-on-failure recipe.

The full symptom-to-skill lookup is in [INDEX.md](INDEX.md).

## Editorial position

Cross-cutting hot takes that appear as **MUST** / **MUST NOT** directives across the relevant skills:

1. **Tag from production, find by tag from tests.** `Modifier.testTag` (Compose) or `android:id` (View) belongs in production source under a constant. Text finders are i18n-fragile and churn with copy edits.
2. **Default to `useUnmergedTree = false`** in Compose. Switch only when the matcher targets an inner element collapsed by the merge pass.
3. **Animation tests require `mainClock.autoAdvance = false`** in Compose. Otherwise `InfiniteAnimationPolicy` cancels infinite animations and your test never observes a determinate state.
4. **`waitUntil` timeouts are wall clock; `advanceTimeUntil` is test clock** in Compose. Same split applies to `runTest` virtual time vs Turbine's wall-clock timeouts.
5. **Funnel state mutations through `runOnIdle` or `runOnUiThread`.** Direct `var = value` from the test thread races with the recomposer.
6. **Prefer the v2 entry points** for Compose (`androidx.compose.ui.test.junit4.v2.createComposeRule`, `androidx.compose.ui.test.v2.runComposeUiTest`). They use `StandardTestDispatcher`, matching `kotlinx.coroutines.test.runTest`.
7. **`Thread.sleep` is a smell**, except in screenshot tests waiting on the RenderThread for ripples.
8. **`am instrument` exit codes only matter with `-w`.** Without it, `$?` is meaningless and CI silently passes broken tests.
9. **`pm clear` for hermetic reset, not `am force-stop`.** Force-stop kills the process but leaves SharedPreferences, DB, and files intact.
10. **`androidx.test:orchestrator` ships via `androidTestUtil`, NOT `androidTestImplementation`.** The latter is a silent no-op.

## Scope

In scope:

- Jetpack Compose UI testing (the original `compose/` set).
- JVM unit tests on Android: JUnit4, Mockito + mockito-kotlin, MockK, kotlinx-coroutines-test, Turbine, Robolectric.
- On-device instrumented tests: `AndroidJUnit4`, `ActivityScenario`, `FragmentScenario`, Espresso, UiAutomator.
- ADB as the device-control surface for E2E and CI: connection, app management, instrumentation invocation, input injection, capture, logcat, transfer, automation.
- Cross-cutting fundamentals: pyramid, what to test, test doubles, strategies, source sets.

Out of scope (intentionally; root-foundations focus first):

- Screenshot/golden testing libraries (Paparazzi, Roborazzi, Shot). May be added later.
- Code coverage tooling (JaCoCo, Kover).
- Static analysis (lint, ktlint, detekt).
- Performance and stability, covered in the sister repo [compose-performance-skills](https://github.com/skydoves/compose-performance-skills).
- Compose Multiplatform iOS/Desktop/Web specifics. Most Compose skills apply because `ComposeUiTest` is multiplatform, but iOS/Desktop/Web targets are not validated here.

## Contributing

Contributions are welcome. Before opening a pull request:

1. Read [`docs/SPEC.md`](docs/SPEC.md) end to end. Frontmatter template, body structure, voice, mandatory rules, post-write self-check: all non-negotiable.
2. Pull facts and code samples from [`docs/CORPUS.md`](docs/CORPUS.md) so vocabulary stays consistent across skills.
3. Verify cross-references resolve. The convention is `..` for same-category siblings, `../..` for same-set different category, `../../..` for cross-set.
4. Run the post-write self-check (SPEC §9). Frontmatter validates, body ≤500 lines, ≥1 RIGHT/WRONG pair for code-level skills, Verification checklist, ≥3 authoritative reference URLs, no first person, no emojis, no time-sensitive phrasing.
5. Add the new skill to [INDEX.md](INDEX.md) under each lookup section it belongs to.

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for the full procedure.

## Find this repository useful? :heart:
Support it by joining __[stargazers](https://github.com/skydoves/android-testing-skills/stargazers)__ for this repository. :star: <br>
Also __[follow](https://github.com/skydoves)__ me for my next creations! 🤩

# License
```xml
Designed and developed in 2026 by skydoves (Jaewoong Eum)

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
