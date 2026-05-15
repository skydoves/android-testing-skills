---
name: understanding-the-testing-pyramid
description: Use this skill to size an Android test suite using Google's small / medium / big scope vocabulary and the qualitative pyramid. Explains why most apps should hold "many small tests and relatively few big tests", how scope (small/medium/big) is orthogonal to execution location (local vs instrumented), and where the older 70/20/10 numeric ratio actually comes from. Use when the user asks "how many unit vs UI tests should I write", "what's the testing pyramid", "all my tests are instrumented and CI is slow", "test pyramid 70 20 10", "Android small medium large tests", "Robolectric counts as which layer", or mentions Unit / Component / Feature / Application / Release Candidate framing from the strategies page.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - testing-pyramid
  - test-scope
  - small-medium-big
  - test-strategy
  - flaky-tests
  - ci-slow
  - instrumented-vs-local
  - robolectric
  - junit4
---

# Understanding the Testing Pyramid — Pick the 3-Layer Framing and Stick to It

Android test suites slow CI and rot when teams ship one big instrumented test per behavior and skip unit coverage of the underlying logic. Google's `/training/testing/fundamentals` page frames test sizing with three scopes — **small, medium, big** — and the `/strategies` page sketches a qualitative pyramid on top of that. This skill encodes the 3-layer framing, names the alternative 5-layer framing so authors do not blend them, and is honest about where the famous numeric ratios come from. Subsequent skills (`../choosing-what-to-test/SKILL.md`, `../../strategies/applying-testing-strategies/SKILL.md`) assume this vocabulary.

## When to use this skill

- The user asks "how many unit vs UI tests should I write" or quotes a 70/20/10 / 80/15/5 ratio and asks if it is in Google's docs.
- The user calls every test "an integration test" / "an end-to-end test" and the agent needs a vocabulary to sort them.
- The user reports "CI is slow" / "the suite takes 40 minutes" and wants to know whether to delete instrumented tests.
- The user asks whether Robolectric is "a unit test" or "an integration test" — it is a *medium-local* test in Google's framing.
- The user is starting a new module and asks where to begin (answer: small tests first, per `/strategies`).

## When NOT to use this skill

- The user wants to know *what* to put in each test (which screens, which ViewModels) — use `../choosing-what-to-test/SKILL.md`.
- The user wants to wire `testImplementation` vs `androidTestImplementation` Gradle configurations — use `../../strategies/organizing-test-source-sets/SKILL.md`.
- The user wants to pick between `Fake` / `Mock` / `Stub` / `Spy` — use `../../doubles/picking-test-doubles/SKILL.md`.
- The user is debugging a single flaky test and wants to fix it — start with `../../../compose/synchronization/synchronizing-with-idle/SKILL.md` or `../../../jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md`.

## Prerequisites

- A module with at least one test class so the discussion is concrete.
- Familiarity with the `src/test/` (local, JVM) and `src/androidTest/` (instrumented) split. If unfamiliar, read `../../strategies/organizing-test-source-sets/SKILL.md` first.
- An understanding that "scope" (small/medium/big) and "execution location" (local/instrumented) are two independent axes per Google.

## The two framings (pick exactly one per skill / per design doc)

Google publishes two framings on adjacent pages. They are not contradictory; they are different views.

### Framing A — Small / Medium / Big (3 layers)  ← this skill recommends this one

From `/training/testing/fundamentals` ("Types of tests in Android" → "By Scope"):

> "Tests are also classified by their scope, or how much of the code they cover. There are three categories of test scope:
> - **Unit tests** or **small tests** only verify a very small portion of the app, such as a method or class.
> - **End-to-end tests** or **big tests** verify larger parts of the app at the same time, such as a whole screen or user flow.
> - **Medium tests** are in between and check the integration between two or more units."
> — `developer.android.com/training/testing/fundamentals`

| Scope | What it covers | Typical runner | Typical execution |
|---|---|---|---|
| **Small** | One method or class | JUnit4 + Mockito/MockK + fakes | `src/test/` (JVM) |
| **Medium** | 2+ classes integrating, possibly Android framework via Robolectric | JUnit4 + Robolectric or in-process Android | `src/test/` (host) OR `src/androidTest/` (instrumented) |
| **Big** | A whole screen, user flow, or release-build smoke | AndroidJUnitRunner + Espresso/Compose-test/UiAutomator | `src/androidTest/` |

### Framing B — Unit / Component / Feature / Application / Release Candidate (5 layers)

From `/training/testing/fundamentals/strategies` (the "scalable strategy" table):

| Level | Scope | Network |
|---|---|---|
| Unit | Single method or class with minimal dependencies | None |
| Component | Module or component | None |
| Feature | Multiple components / modules | "supports mocked network access" |
| Application | Whole app, debuggable binary | n/a |
| Release Candidate | Whole release build, minified | n/a |

Most teams stay with the 3-layer Framing A because it is the simpler, more cited shape. Reach for Framing B only when designing a multi-team CI strategy where the extra granularity earns its keep. **MUST NOT** blend the two in a single doc — pick one and stick with it.

## Scope is orthogonal to execution location

The single most important quote on the page, and the one the agent should cite when a developer says "all my tests are instrumented":

> "Not all unit tests are local, and not all end-to-end tests run on a device. For example:
> - **Big local test**: You can use an Android simulator that runs locally, such as Robolectric.
> - **Small instrumented test**: You can verify that your code works well with a framework feature, such as a SQLite database."
> — `developer.android.com/training/testing/fundamentals`

Two axes:

```
                 LOCAL (JVM)              INSTRUMENTED (device)
SMALL    pure JUnit + fakes          unit test of SQLite via real DB
MEDIUM   Robolectric component      component test on real Android
BIG      Robolectric flow           Espresso / Compose UI flow
```

Robolectric is a **local** Android simulator. It is not "an instrumented test" — it runs on the JVM under `src/test/`. See `../../strategies/organizing-test-source-sets/SKILL.md` and the cross-category skill `../../../jvm-tests/robolectric/using-robolectric-correctly/SKILL.md`.

## The qualitative pyramid

The `/strategies` page only commits to a qualitative shape:

> "Most apps should have many small tests and relatively few big tests, forming a pyramid shape."
> — `developer.android.com/training/testing/fundamentals/strategies`

> "**Key Point:** In general, you should try to add tests as soon as possible in the development cycle. That typically means starting with small tests."
> — `developer.android.com/training/testing/fundamentals/strategies`

That is the entirety of Google's current quantitative claim: "many small, few big". No ratio.

## Where the 70/20/10 ratio actually comes from

The "70 percent unit, 20 percent integration, 10 percent end-to-end" rule is widely cited but is **NOT** present on the current `/training/testing/fundamentals` or `/training/testing/fundamentals/strategies` pages (as of 2026-05-06; verified in `tasks/research/R8-android-fundamentals.md`). The number is from Google's own engineering literature — *Software Engineering at Google*, chapter 11 — not the Android training pages.

**MUST NOT** attribute "70/20/10" to `developer.android.com`. The numeric ratio is a community heuristic popularised in *Software Engineering at Google* (https://abseil.io/resources/swe-book, ch. 11) — paraphrase carefully, do NOT present any specific sentence as a verbatim quote unless you confirm it line-by-line in the book. The book frames the split as a heuristic, not a law: different domains warrant different ratios; see "When to break the pyramid" below.

## Cost / speed tradeoff

The reason the pyramid bottom is wide:

| Layer | Wall-clock per test | Failure signal | Maintenance cost |
|---|---|---|---|
| Small | 1-50 ms | Pinpoints a class/method | Low |
| Medium (Robolectric) | 50-500 ms | Pinpoints a component | Medium |
| Big (instrumented) | 5-30 s | Says "the screen is broken" | High (flake, emulator state, animation) |

A flat or inverted pyramid (lots of big tests, few small) produces:

- Slow CI (10x to 100x slower than a unit-heavy suite of equivalent coverage).
- High flake rate; flake correlates with test size and external dependencies (`/strategies` notes "no network access" for unit and component layers).
- Vague failure signal — "screen X broken" instead of "function Y returned the wrong value for input Z".
- Painful refactors — UI tests assert on rendered text and break on every copy change.

`/training/testing/instrumented-tests` makes the price explicit:

> "We recommend using instrumented tests only in cases where you must test against the behavior of a real device."
> — `developer.android.com/training/testing/instrumented-tests`

That is Google's "use big tests as a last resort" stance, in their own words.

## When to break the pyramid intentionally

The pyramid is a default, not a law. Break it when:

1. **The product IS the integration.** A payments SDK that primarily wires three vendor SDKs together has more value in feature/application tests than in unit tests of glue code. The unit tests would be tautological mock verifications.
2. **The framework owns the logic.** A pure CRUD screen with a DataStore-backed `ViewModel` and stock Material widgets has very little to unit-test. Big tests catch wiring; small tests would just be testing `runBlocking { dataStore.data.first() }`.
3. **Determinism is the point.** A render-correctness suite (screenshot tests, animation timing) lives at the big-test layer because that is where the bug surfaces. See the cross-category skill `../../../compose/audit/auditing-compose-test-suite/SKILL.md`.
4. **Cost economics flip.** On Firebase Test Lab / Gradle Managed Devices the per-minute cost of a big test may be small enough that the maintenance cost dominates anyway — and unit tests still win on flake.

What does NOT justify breaking the pyramid:

- "Unit tests are too hard to set up" — that is a signal of bad architecture, not a reason to pile on big tests. See `developer.android.com/training/testing/fundamentals` ("With a testable app architecture …").
- "We can't mock X" — use a fake instead of a mock. See `../../doubles/picking-test-doubles/SKILL.md`.
- "It works on my machine, just ship more device tests" — the flake will follow the suite. Diagnose with `../../../adb/observability/extracting-logs-with-logcat/SKILL.md`.

## Patterns

### Pattern: WRONG vs RIGHT — citing the 70/20/10 ratio

```markdown
<!-- WRONG -->
> Per Google's developer documentation, the testing pyramid recommends
> a 70/20/10 split between unit, integration, and end-to-end tests.
> Source: developer.android.com/training/testing/fundamentals/strategies
```
WRONG because: that ratio is not on the strategies page as of 2026-05-06. The page only says "many small, few big" qualitatively. Citing developer.android.com here is fabrication.

```markdown
<!-- RIGHT -->
> Google's Android docs commit to a qualitative pyramid only:
> "Most apps should have many small tests and relatively few big tests"
> (developer.android.com/training/testing/fundamentals/strategies).
> The familiar 70/20/10 split comes from Software Engineering at Google,
> ch. 11 (https://abseil.io/resources/swe-book), not the Android docs.
```

### Pattern: WRONG vs RIGHT — labeling Robolectric

```markdown
<!-- WRONG -->
> Robolectric tests are instrumented tests because they exercise the
> Android framework. Put them in src/androidTest/.
```
WRONG because: Robolectric runs on the JVM with no Android device or emulator. Per `/training/testing/fundamentals` ("Big local test: You can use an Android simulator that runs locally, such as Robolectric"), Robolectric is a *local* test, in `src/test/`, regardless of scope.

```markdown
<!-- RIGHT -->
> Robolectric is a local (JVM) Android simulator. By scope it can be small,
> medium, or big depending on what the test exercises. Place it in src/test/.
> Use src/androidTest/ only for tests that require a real device.
```

### Pattern: WRONG vs RIGHT — pyramid framing in one design doc

```markdown
<!-- WRONG -->
> Our suite has 200 small tests, 40 medium tests, 8 feature tests, and
> 3 release-candidate tests, which is a healthy pyramid.
```
WRONG because: this blends Framing A ("small/medium") with Framing B ("feature/release candidate"). The reader cannot tell whether "small" means Framing A's small or Framing B's unit.

```markdown
<!-- RIGHT — pick Framing A and stick with it -->
> Our suite has 200 small tests, 40 medium (Robolectric component) tests,
> and 11 big (instrumented Compose-test + Espresso) tests. Per Google's
> small/medium/big framing on /training/testing/fundamentals, the ratio
> is roughly 80/16/4, in line with the pyramid recommendation.
```

## Decision matrix — what scope should a new test be?

```
Question                                         → Scope
----------------------------------------------------------
"Does it depend on the Android framework?"
  No                                             → Small (src/test/)
  Yes, simulated via Robolectric                 → Medium (src/test/)
  Yes, must be a real device                     → Big (src/androidTest/)
"Does it exercise more than one class?"
  No                                             → Small
  Yes, but no I/O / no UI                        → Medium
  Yes, with UI rendering or system services      → Big
"Does it cross a network or process boundary?"
  No                                             → Small or Medium
  Yes (mocked)                                   → Medium (Feature in Framing B)
  Yes (real)                                     → Big (Application/RC in Framing B)
```

## Mandatory rules

- **MUST** use the 3-layer "small / medium / big" framing in default skill copy and design docs. Reach for the 5-layer Unit/Component/Feature/Application/Release Candidate framing only when explicitly invoking the `/strategies` page.
- **MUST NOT** blend the two framings in a single document. Pick one.
- **MUST NOT** attribute the "70/20/10" numeric ratio to `developer.android.com`. Cite *Software Engineering at Google* (https://abseil.io/resources/swe-book).
- **MUST NOT** describe Robolectric as "an instrumented test". It is a *local* Android simulator that may carry any scope.
- **MUST** label every new test with its scope in a comment or naming convention (`@SmallTest`, `@MediumTest`, `@LargeTest` from `androidx.test.filters`) so CI can shard accordingly.
- **PREFERRED:** start a new module with small tests first, per Google's "starting with small tests" guidance, and only add medium/big tests where small ones cannot reach.
- **PREFERRED:** when CI is slow, delete redundant big tests before adding more small ones — the pyramid tightens by removing the top, not just by widening the base.

## Verification

- [ ] Every test class in the module is reachable via one of `@SmallTest`, `@MediumTest`, `@LargeTest` (or by source-set placement).
- [ ] No test in `src/androidTest/` could equivalently run as a Robolectric host test in `src/test/` without losing fidelity. (If yes, move it.)
- [ ] CI logs report unit-test wall time < instrumented-test wall time per layer per change.
- [ ] No design doc cites "70/20/10" attributed to `developer.android.com`.
- [ ] No design doc blends Framing A and Framing B vocabulary.
- [ ] The team's testing strategy doc names exactly one framing in its glossary.

## References

- `developer.android.com/training/testing/fundamentals` — small/medium/big scope, "Not all unit tests are local" quote, testable architecture rationale.
- `developer.android.com/training/testing/fundamentals/strategies` — qualitative pyramid ("many small, few big"), 5-level Unit/Component/Feature/Application/Release Candidate table, "starting with small tests" guidance.
- `developer.android.com/training/testing/instrumented-tests` — "instrumented tests only in cases where you must test against the behavior of a real device".
- *Software Engineering at Google*, ch. 11 — origin of the 70/20/10 ratio. https://abseil.io/resources/swe-book
- `tasks/research/R8-android-fundamentals.md` — verbatim quotes, version-sensitive notes, the "70/20/10 not on the page" finding.
- `androidx.test.filters` — `@SmallTest` / `@MediumTest` / `@LargeTest` annotations for runtime sharding.
- Sibling skills: `../choosing-what-to-test/SKILL.md`, `../../doubles/picking-test-doubles/SKILL.md`, `../../strategies/applying-testing-strategies/SKILL.md`, `../../strategies/organizing-test-source-sets/SKILL.md`.
- Cross-category: `../../../jvm-tests/robolectric/using-robolectric-correctly/SKILL.md`, `../../../jvm-tests/runner/configuring-junit4-on-android/SKILL.md`, `../../../instrumentation/runner/running-instrumented-tests-with-androidjunit4/SKILL.md`, `../../../compose/audit/auditing-compose-test-suite/SKILL.md`.
