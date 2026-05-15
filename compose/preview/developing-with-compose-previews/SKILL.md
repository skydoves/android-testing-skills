---
name: developing-with-compose-previews
description: Use this skill to make `@Preview` functions the primary feedback loop for Jetpack Compose UI work — preview-driven development. Covers hoisting state out of composables so a preview never needs a `ViewModel` or DI graph, `LocalInspectionMode` for conditional preview rendering (e.g. placeholder instead of Coil/Glide), `@PreviewParameter` + `PreviewParameterProvider` for rendering every UI state side by side, multi-configuration previews (`uiMode`, `fontScale`, `device`, `widthDp`, `locale`), `@Preview`-annotated annotation classes for reusable preview sets, and the anti-patterns that make previews rot (screen-level previews, unwrapped theme, zero-width `fillMaxWidth` previews, Lorem-ipsum data). Use when the user mentions "preview won't render", "preview needs a ViewModel", "@PreviewParameter", "LocalInspectionMode", "preview out of date", "preview-driven development", "previews keep breaking", or "how to structure Compose previews".
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - jetpack-compose
  - ui-testing
  - compose-preview
  - preview-driven-development
  - PreviewParameter
  - PreviewParameterProvider
  - LocalInspectionMode
  - state-hoisting
  - multipreview
  - preview-anti-patterns
---

# Developing With Compose Previews — Preview-Driven Development

`@Preview` is the cheapest verification loop in Compose: edit, see the render in ~1 s, no build/install/navigate cycle. It only pays off if composables are structured so previews stay alive — hoisted state, no `ViewModel` in the signature, realistic sample data. This skill encodes that structure and the anti-patterns that cause teams to abandon previews. For taking those previews as on-device screenshots in CI, see `../capturing-preview-screenshots-in-ci/SKILL.md`.

## When to use this skill

- A `@Preview` "fails to render" or crashes because the composable takes a `ViewModel`, repository, or `Context`-bound dependency.
- The user wants to see loading / empty / error / overflow-text / RTL states without running the app — `@PreviewParameter` territory.
- A composable loads images with Coil/Glide and the preview shows nothing — needs `LocalInspectionMode`.
- The user asks "how should I structure Compose previews so they don't keep breaking" or mentions "preview-driven development".
- Previews exist but drift out of date / show the "out of date" banner constantly and the team is about to delete them.

## When NOT to use this skill

- The user wants to capture previews as screenshots on a device or in CI (HotSwan `captureAllPreviews`, GitHub Pages catalog) — use `../capturing-preview-screenshots-in-ci/SKILL.md`.
- The user wants pixel-diff golden-image regression tests (Paparazzi / Roborazzi) — that is a different tool; see `../capturing-preview-screenshots-in-ci/SKILL.md` for where it fits relative to device rendering.
- The user is writing behavioral UI tests (`ComposeTestRule`, finders, assertions) — start at `../../setup/configuring-test-dependencies/SKILL.md` and `../../finders/finding-nodes-by-tag-text-content/SKILL.md`.

## Prerequisites

- `androidx.compose.ui:ui-tooling-preview` on the main classpath (provides `@Preview`, `@PreviewParameter`, `PreviewParameterProvider`; see `compose/ui/ui-tooling-preview/`), plus `debugImplementation("androidx.compose.ui:ui-tooling")` so Android Studio can render.
- `LocalInspectionMode` ships in `androidx.compose.ui:ui` (`compose/ui/ui/.../platform/InspectionMode.kt` — `staticCompositionLocalOf { false }`), no extra dependency.
- A Compose-enabled module (the `org.jetbrains.kotlin.plugin.compose` plugin or `buildFeatures.compose = true`).

## Workflow

- [ ] **1. Decide what to preview: the component, not the screen.** Screen-level composables pull in `ViewModel`s, navigation, DI — they are the hardest to preview and the least worth it. Preview the leaf and mid-level composables that take plain parameters; the screen composable wires those together and does not need its own `@Preview`.

- [ ] **2. Hoist state until the previewable composable takes only plain values.** If a composable's signature mentions a `ViewModel`, split it: a stateful wrapper at the call site that owns the `ViewModel`, and a stateless composable that takes the data. Preview the stateless one. (See Pattern: `ViewModel` in the signature.)

- [ ] **3. Wrap the preview body in your app theme.** A composable that reads `MaterialTheme.colorScheme` / `.typography` renders with default Material values unless wrapped — the preview then lies about how it looks shipped. Always `AppTheme { … }` (or your equivalent) inside the `@Preview` function.

- [ ] **4. Give the preview a realistic viewport when the composable expands.** A composable using `Modifier.fillMaxWidth()` collapses to zero width in a wrap-content preview. Add `@Preview(widthDp = 360)` (and `heightDp` if it fills height) so it renders at a believable size.

- [ ] **5. Use realistic sample data, not `"Lorem ipsum"`.** Sample data should match the production model's shape and edge cases (long names, empty strings, zero-item lists). When the model changes, the preview should fail to compile — that compile error is the feature: it tells you the preview (and the doc it represents) is stale.

- [ ] **6. Cover every UI state with `@PreviewParameter`.** Render loading / content / empty / error / overflow side by side from one function via a `PreviewParameterProvider`. (See Pattern: one state per preview.)

- [ ] **7. Fan out across configurations with repeated `@Preview`.** `@Preview` is `@Repeatable` — stack `uiMode = UI_MODE_NIGHT_YES`, `fontScale = 1.5f`, `device = "spec:width=320dp,height=640dp"`, `locale = "ar"` (RTL). Each annotation is one render. For sets you reuse across many composables, define a custom annotation class annotated with the `@Preview`s and apply that one annotation ("multipreview").

- [ ] **8. Quarantine unavoidable dependencies behind `LocalInspectionMode`.** Image loaders need a `Context` and network access. Branch on `LocalInspectionMode.current` to draw a placeholder in preview. Use sparingly — if you reach for it everywhere, the composable still needs better state hoisting.

## Patterns

### Pattern: `ViewModel` in the signature

```kotlin
// WRONG
@Composable
fun ProfileScreen(viewModel: ProfileViewModel) {
    val user by viewModel.user.collectAsStateWithLifecycle()
    Column { Text(user.name); Text(user.bio) }
}

@Preview
@Composable
fun ProfileScreenPreview() {
    ProfileScreen(viewModel = ???)   // can't: needs repository, network, DB
}
// WRONG because: the preview environment has no DI graph. Studio must instantiate
// ProfileViewModel and its whole dependency chain, so the preview fails to compile
// or crashes at render time.
```

```kotlin
// RIGHT — stateless composable takes plain values; ViewModel stays at the call site
@Composable
fun ProfileScreen(viewModel: ProfileViewModel = hiltViewModel()) {
    val user by viewModel.user.collectAsStateWithLifecycle()
    ProfileContent(name = user.name, bio = user.bio, onEditClick = viewModel::onEdit)
}

@Composable
fun ProfileContent(name: String, bio: String, onEditClick: () -> Unit = {}) {
    Column { Text(name); Text(bio) }
}

@Preview(widthDp = 360)
@Composable
private fun ProfileContentPreview() {
    AppTheme { ProfileContent(name = "Jane Doe", bio = "Android developer") }
}
```

### Pattern: one state per preview vs `@PreviewParameter`

```kotlin
// WRONG — three near-duplicate preview functions, easy to let one rot
@Preview @Composable fun UserCardLoadingPreview()  { AppTheme { UserCard(UiState.Loading) } }
@Preview @Composable fun UserCardContentPreview()  { AppTheme { UserCard(UiState.Content(sampleUser)) } }
@Preview @Composable fun UserCardErrorPreview()    { AppTheme { UserCard(UiState.Error("offline")) } }
// WRONG because: each state is a separate function with its own copy of the theme wrapper;
// adding a new state means remembering to add another function. Edge cases get skipped.
```

```kotlin
// RIGHT — one function, every state generated from the provider
class UserStateProvider : PreviewParameterProvider<UiState> {
    override val values = sequenceOf(
        UiState.Loading,
        UiState.Content(User(name = "Jane Doe", bio = "Short bio")),
        UiState.Content(User(name = "A very long display name that overflows the row", bio = "")),
        UiState.Empty,
        UiState.Error("network unavailable"),
    )
}

@Preview(widthDp = 360)
@Composable
private fun UserCardPreview(@PreviewParameter(UserStateProvider::class) state: UiState) {
    AppTheme { UserCard(state) }
}
```

### Pattern: image loader with no `Context`

```kotlin
// WRONG
@Composable
fun UserAvatar(imageUrl: String) {
    AsyncImage(model = imageUrl, contentDescription = "Avatar", modifier = Modifier.size(48.dp))
}
// WRONG because: AsyncImage needs an ImageLoader (Context + network). In a preview it renders
// empty, so a composable built around the avatar looks broken in Studio for no real reason.
```

```kotlin
// RIGHT — branch on LocalInspectionMode for the preview-only path
@Composable
fun UserAvatar(imageUrl: String) {
    if (LocalInspectionMode.current) {
        Box(Modifier.size(48.dp).clip(CircleShape).background(MaterialTheme.colorScheme.surfaceVariant))
    } else {
        AsyncImage(
            model = imageUrl,
            contentDescription = "Avatar",
            modifier = Modifier.size(48.dp).clip(CircleShape),
        )
    }
}
```

### Pattern: reusable preview set as a multipreview annotation

```kotlin
// WRONG — paste the same four @Preview lines onto every composable
@Preview(name = "phone")           // duplicated across dozens of files;
@Preview(name = "dark", uiMode = UI_MODE_NIGHT_YES)
@Preview(name = "font 1.5x", fontScale = 1.5f)
@Preview(name = "RTL", locale = "ar")
@Composable fun FooPreview() { AppTheme { Foo() } }
// WRONG because: when the standard set changes (add a tablet size, drop a font scale) you
// have to edit every composable. The set is policy; it belongs in one place.
```

```kotlin
// RIGHT — define once, apply once. @Preview can annotate an annotation class.
@Preview(name = "phone", showBackground = true)
@Preview(name = "dark", uiMode = Configuration.UI_MODE_NIGHT_YES, showBackground = true)
@Preview(name = "font 1.5x", fontScale = 1.5f, showBackground = true)
@Preview(name = "RTL", locale = "ar")
annotation class AppPreviews

@AppPreviews
@Composable
private fun FooPreview() { AppTheme { Foo() } }
```

## Anti-patterns to flag in review

- **Screen-level `@Preview`.** A `@Preview` on a composable that takes a `ViewModel`/`NavController`. Preview the components it composes instead.
- **Theme not wrapped.** A preview body that calls a `MaterialTheme`-dependent composable without `AppTheme { … }` — it renders with default Material values, not the app's.
- **Zero-width preview.** A composable using `fillMaxWidth()`/`fillMaxSize()` previewed without `widthDp`/`heightDp` — it collapses.
- **`"Lorem ipsum"` / fabricated data.** Sample data that does not mirror the production model's shape and edge cases; it hides overflow and empty-state bugs and never forces a recompile when the model changes.
- **Mocking inside a preview.** Mockito/MockK objects in a `@Preview` body. The need to mock means state was not hoisted; fix the composable, not the preview.

## Mandatory rules

- **MUST** keep previewable composables free of `ViewModel`, repository, `NavController`, and `Context`-bound parameters. Hoist that state to a stateful wrapper at the call site.
- **MUST** wrap every `@Preview` body in the app theme when the composable reads `MaterialTheme` colors/typography/shapes.
- **MUST** use `@PreviewParameter` + a `PreviewParameterProvider` to cover the full set of UI states (loading, content, empty, error, overflow) rather than one preview function per state.
- **MUST** add `widthDp`/`heightDp` to `@Preview` for composables that fill width/height, so the render has a realistic viewport.
- **MUST** use sample data that mirrors the production model — a model change should break the preview's compile, and that is intended.
- **MUST NOT** put mocks (Mockito/MockK) in a `@Preview` body; **MUST NOT** sprinkle `LocalInspectionMode` checks as a substitute for hoisting state — reserve it for genuinely environment-bound calls (image loading, sensors).
- **PREFERRED:** mark preview functions `private`; they are tooling entry points, not API.
- **PREFERRED:** extract the team's standard configuration set into one `@Preview`-annotated annotation class (multipreview) and apply that.

## Verification

- [ ] No previewable composable takes a `ViewModel`/repository/`Context` parameter; `git grep -nE '@Preview' -A3 -- '**/*.kt'` shows preview bodies calling stateless composables only.
- [ ] Every `@Preview` body that uses `MaterialTheme.*` is wrapped in the app theme.
- [ ] State-bearing composables have a `PreviewParameterProvider` covering loading/content/empty/error/overflow; `grep -rn "PreviewParameterProvider" src/` is non-empty for those modules.
- [ ] Composables using `fillMaxWidth`/`fillMaxSize` have `@Preview(widthDp = …)`.
- [ ] No `mock(`/`mockk(` appears under a `@Preview` function.
- [ ] Building the module after a model field rename causes the affected previews to fail compilation (proof the sample data is real, not Lorem).

## References

- developer.android.com/develop/ui/compose/tooling/previews — `@Preview` parameters, `@PreviewParameter`/`PreviewParameterProvider`, multipreview annotation classes, `device` spec strings.
- `androidx/compose/ui/ui-tooling-preview/src/commonMain/kotlin/androidx/compose/ui/tooling/preview/Preview.kt` — the `@Preview` annotation (`@Repeatable`; `name`, `group`, `widthDp`, `heightDp`, `locale`, `fontScale`, `showSystemUi`, `showBackground`, `backgroundColor`, `uiMode`, `device`, `wallpaper`).
- `androidx/compose/ui/ui-tooling-preview/src/commonMain/kotlin/androidx/compose/ui/tooling/preview/PreviewParameter.kt` — `PreviewParameterProvider<T>` (`values: Sequence<T>`, `getDisplayName`) and `@PreviewParameter(provider, limit)`.
- `androidx/compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/platform/InspectionMode.kt` — `LocalInspectionMode = staticCompositionLocalOf { false }`.
- developer.android.com/develop/ui/compose/state-hoisting — state hoisting, stateful vs stateless composables.
- hotswan.dev/blog/compose-preview-driven-development — skydoves, "Compose Preview Driven Development with Instant Feedback" (state hoisting for previewability, `LocalInspectionMode`, `@PreviewParameter`, anti-patterns, the preview-rebuild friction).
- Sibling skill: `../capturing-preview-screenshots-in-ci/SKILL.md` — rendering these previews as device screenshots and a CI catalog; where Paparazzi/Roborazzi fit.
- Sibling skill: `../../setup/configuring-test-dependencies/SKILL.md` — the Gradle matrix for Compose test/tooling artifacts.
