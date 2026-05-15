# compose-test-skills — Authoring Spec

This file is the single source of truth for every `SKILL.md` in this repo. All skill authors **MUST** follow it.

---

## 1. Directory layout

```
compose-test-skills/
├── README.md                 # repo philosophy + quickstart
├── INDEX.md                  # symptom → skill map
├── LICENSE                   # Apache-2.0
└── <category>/<slug>/
    ├── SKILL.md              # the skill (required)
    └── references/           # optional, ONE level deep only
        └── <topic>.md
```

- `<category>` is lowercase, one of `setup`, `finders`, `assertions`, `actions`, `synchronization`, `patterns`, `interop`, `debug`, `audit`.
- `<slug>` is **gerund form** (`finding-nodes-by-tag-text-content`, `injecting-touch-gestures`). Lowercase, hyphenated, max 64 chars.
- `SKILL.md` is uppercase.
- `references/` is optional; use it for files the SKILL.md body explicitly links to (e.g. "See `references/touch-injection-coordinates.md`"). Never nest deeper — deep files get truncated by agents that preview with `head -100`.

## 2. Frontmatter template (copy exactly)

```yaml
---
name: <slug-matching-folder>
description: <trigger paragraph — 3rd person, 1-4 sentences, ≤1024 chars. Leads with "Use this skill to …" or a verb; ends with a "if the user mentions X, Y, Z" hammer. Must state BOTH what + when.>
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - <3-6 more specific keywords>
---
```

Rules:
- `name` MUST match the folder slug exactly.
- `description` MUST be ≤1024 chars, 3rd person, no first/second person ("I can…", "You can…" → forbidden). Include vocabulary variants the user would naturally say (e.g. "flaky test", "test rule won't compile", API names like `onNodeWithTag`, `performTouchInput`, `mainClock.advanceTimeBy`).
- NO reserved words: skill names must not contain `anthropic` or `claude`.
- `keywords` include both symptoms (`flaky-animation-test`, `test-times-out`, `tag-not-found`) and APIs (`compose-test-rule`, `state-restoration-tester`, `idling-resource`).

### Description quality checklist
- [ ] Starts with a verb or "Use this skill to …"
- [ ] Names ≥3 trigger vocabulary variants
- [ ] States the symptom/problem it addresses
- [ ] Closes with a "if the user mentions X, use this skill" line
- [ ] Under 1024 chars; reads in one breath
- [ ] No time-sensitive wording ("as of March 2026", "recently")
- [ ] Third person throughout

## 3. Body structure

Target **≤500 lines**. Longer content goes into `references/`. Use this section order:

```markdown
# <Title> — <one-line tagline>

<2-3 sentence orientation: what this skill fixes, who should care, approximate impact>

## When to use this skill

<3-5 bullets of concrete triggers. Mirror the vocabulary from frontmatter description.>

## When NOT to use this skill

<1-3 bullets where this skill does NOT apply — cross-reference sibling skills by relative path.>

## Prerequisites

<Bulleted. ui-test artifact versions, build flags, Activity setup, any sibling skills to run first.>

## Workflow

<Numbered steps OR markdown checkbox list. Every step has:
  - a concrete command or code snippet
  - a short rationale
  - (when relevant) a RIGHT / WRONG snippet pair>

## Patterns

### Pattern: <name>

<problem → fix as RIGHT / WRONG paired snippets:>

```kotlin
// WRONG
<code>
// WRONG because: <one line>
```

```kotlin
// RIGHT
<code>
```

<repeat for 2-5 patterns>

## Mandatory rules

- **MUST** …
- **MUST NOT** …
- **PREFERRED:** …

## Verification

<Checkbox list of commands/checks that prove the skill was applied correctly.
e.g. "- [ ] `./gradlew :app:connectedDebugAndroidTest --tests MyTest` passes" / "- [ ] No `Thread.sleep` remains in the test method">

## References

- Inline links to authoritative sources (Android Developers testing pages, androidx file paths, Chris Banes / Adam Powell posts).
- If you use `references/*.md`, list them here with one-line summaries.
```

Shorter skills can collapse `Workflow` into `Patterns`; index-style skills can omit `Workflow`.

## 4. Voice & tone

- Written **for Claude the agent** (reader is an LLM, not a human dev).
- **Imperative 2nd person** for instructions ("Run `./gradlew connectedDebugAndroidTest`", "Replace `performClick()` with `performTouchInput { longClick() }`").
- **3rd person** for the user ("the developer", "the user").
- Use **MUST / MUST NOT / DO NOT / PREFERRED** in bold caps for hard rules.
- Use paired **WRONG / RIGHT** blocks with one-line "WRONG because: …" rationale.
- No emojis. No filler ("Let's dive in!"). No apologies.
- No XML tags in skill content.
- No time-sensitive phrasing. Version-specific info goes in a clearly labeled section ("### Compose ≥1.9").

## 5. Skydoves-specific editorial directives (test edition)

Five non-negotiable hot takes carried across every skill. They appear as **MUST**/**MUST NOT** directives so the agent treats them as constraints, not advice.

1. **Tag from production, find by tag from tests.** `Modifier.testTag("…")` belongs in the production source under a constant. Tests find by tag, not by text. Text finders are i18n-fragile and churn with copy edits. (androidx/material3 itself: 1825 `onNodeWithTag` vs 424 `onNodeWithText` vs 46 `onNodeWithContentDescription`.)
2. **Default to `useUnmergedTree = false`.** Switch to `true` only when the matcher targets an inner element collapsed by the merge pass, or when verifying composition detail. Once flipped, the choice is sticky across navigation (`onChild`/`onParent` keep it).
3. **Animation tests require `mainClock.autoAdvance = false`.** Otherwise the installed `InfiniteAnimationPolicy` cancels infinite animations the moment they start, and your test never observes a determinate state. Step frames manually with `advanceTimeBy(durationMs)` or `advanceTimeUntil { state == … }`.
4. **`waitUntil` timeouts are wall clock; `advanceTimeUntil` is test clock.** Mixing them up produces flaky tests. Prefer `mainClock.advanceTimeUntil` whenever the awaited condition is observable through Compose state; reserve `waitUntil` for conditions outside Compose's snapshot system (a `Job.isCompleted`, an external counter from a `LaunchedEffect`).
5. **Funnel state mutations through `runOnIdle` or `runOnUiThread`.** Direct `var = value` from the test thread races with the recomposer. Use `runOnIdle { … }` when you need quiescence first; `runOnUiThread { … }` when you specifically do not (e.g. mid-animation pause with `autoAdvance = false`).

### Two more recurring directives

6. **Prefer the v2 entry points**: `androidx.compose.ui.test.junit4.v2.createComposeRule` and `androidx.compose.ui.test.v2.runComposeUiTest`. They use `StandardTestDispatcher` (matches `kotlinx.coroutines.test.runTest` semantics). The v1 forms use `UnconfinedTestDispatcher` and are deprecated `WARNING`.
7. **`Thread.sleep` is a smell**, except in screenshot tests waiting on the RenderThread. Anywhere else it indicates missing `waitUntil` / `mainClock.advanceTimeBy` / `IdlingResource`.

## 6. RIGHT/WRONG snippet quality bar

Every pattern SHOULD include a RIGHT/WRONG pair when the fix is a code transform. Format:

```kotlin
// WRONG
@Test
fun fadeAnimation() {
    rule.setContent { Box(Modifier.alpha(animateFloatAsState(target).value)) }
    Thread.sleep(500)
    rule.onNode(...).assertIsDisplayed()
}
// WRONG because: Thread.sleep desyncs from MainTestClock; the animation may not have made progress.
```

```kotlin
// RIGHT
@Test
fun fadeAnimation() {
    rule.mainClock.autoAdvance = false
    rule.setContent { Box(Modifier.alpha(animateFloatAsState(target).value)) }
    rule.mainClock.advanceTimeByFrame()                        // kick off
    rule.mainClock.advanceTimeBy(durationMillis = 500)          // step
    rule.onNode(...).assertIsDisplayed()
}
```

The WRONG snippet MUST be labeled, have the one-line "because" rationale, and the RIGHT snippet MUST compile (no `...` ellipses in the critical lines).

## 7. Cross-references

When referring to another skill, use a relative Markdown link.

- **Same category** (sibling slug under the same `<category>/`): one `..` up.
  ```
  See `../testing-animations-deterministically/SKILL.md` for the autoAdvance pattern.
  ```
- **Different category**: two `..`s up to repo root, then back into the target category.
  ```
  See `../../synchronization/testing-animations-deterministically/SKILL.md` for the autoAdvance pattern.
  ```

Do not link sibling skills by URL. Do not assume a shared index loads them — the reader may jump in mid-tree.

## 8. Mandatory rules for skill authors

- **MUST** keep the body ≤500 lines. Split into `references/` if longer.
- **MUST** include at least one RIGHT/WRONG snippet pair in any skill that teaches a code-level fix.
- **MUST** end with a Verification checklist.
- **MUST** cite authoritative URLs (Android Developers testing pages, androidx file paths, Chris Banes / Adam Powell posts, the skydoves CORPUS).
- **MUST** ground every API claim in either an androidx file path (from `docs/CORPUS.md` Section A) or an Android Developers doc URL.
- **MUST NOT** use first person ("I", "we", "let me") anywhere except the frontmatter's `metadata.author`.
- **MUST NOT** recommend `performGesture { }` — it is `@Deprecated`. Always `performTouchInput { }`.
- **MUST NOT** recommend `Thread.sleep` outside of screenshot/RenderThread waits.
- **MUST NOT** claim a feature without a version or doc URL.
- **MUST NOT** duplicate content between skills — cross-reference instead.

## 9. Post-write self-check

Before declaring a skill done:

- [ ] Frontmatter validates (name matches folder, description ≤1024 chars, no reserved words, has ≥4 keywords)
- [ ] Body ≤500 lines
- [ ] At least one RIGHT/WRONG pair (for code-level skills)
- [ ] Verification checklist present
- [ ] At least 3 authoritative reference URLs
- [ ] Skydoves hot-take surfaced when applicable (see §5)
- [ ] `MUST`/`MUST NOT` directives in bold caps
- [ ] No emojis, no first person, no time-sensitive phrasing
- [ ] Every API claim grounded in a file path (`docs/CORPUS.md`) or developer.android.com URL
