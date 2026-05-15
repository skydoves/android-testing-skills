---
name: picking-test-doubles
description: Use this skill to pick the right test double — fake, mock, stub, spy, dummy, or Robolectric shadow — for an Android test. Encodes Google's verbatim preference order ("fakes ... are preferred", "Fakes are preferred over stubs for simplicity", "Fakes or mocks are therefore preferred over spies"), gives Android examples (in-memory `FakeUserRepository`, `mockk<UserRepository>()`, `ShadowSystemClock`), and a decision matrix mapping intent to the correct double. Use when the user asks "fake or mock", "should I use Mockito here", "how do I replace this dependency in tests", "what's the difference between a stub and a mock", "spy vs mock", or mentions `mockk`, `every { } returns`, `verify { }`, `org.mockito`, `argumentCaptor`, `Robolectric` shadows, `ShadowSystemClock`, or `FakeRepository`.
license: Apache-2.0. See LICENSE for complete terms.
metadata:
  author: Jaewoong Eum (skydoves)
  keywords:
  - android-testing
  - test-doubles
  - fake-vs-mock
  - mockito
  - mockk
  - robolectric-shadows
  - dependency-injection
  - in-memory-repository
  - hilt-testing
  - test-design
---

# Picking Test Doubles — Prefer Fakes; Mocks for Interactions Only

Google has an unambiguous preference order for test doubles, stated three times on `/training/testing/fundamentals/test-doubles`. Most Android codebases over-mock and under-fake, which produces tests that pin implementation details (which methods were called) instead of behavior (what state the system landed in). This skill encodes the six doubles, the preference order verbatim, and a decision matrix for picking the right one. Framework-mechanics for the mocking libraries themselves live in sibling skills (`../../../jvm-tests/mocking/mocking-with-mockito/SKILL.md`, `../../../jvm-tests/mocking/mocking-with-mockk/SKILL.md`).

## When to use this skill

- The user asks "should I use a fake or a mock" / "Mockito or MockK" / "what's the difference between a stub and a spy".
- The user is replacing a dependency in a test and is unsure which double to reach for.
- The PR under review uses `verify(repo).getUser(any())` everywhere and the agent suspects over-mocking.
- The user mentions `Robolectric` "shadows" and asks where they fit in the test-double taxonomy.
- The user says "every { repo.getUser() } returns User('Alice')" with no behavior under test and the agent should suggest a fake.

## When NOT to use this skill

- The user already picked a fake / mock and wants framework mechanics (`whenever` / `every` / `verify` / `argumentCaptor`) — use `../../../jvm-tests/mocking/mocking-with-mockito/SKILL.md` or `../../../jvm-tests/mocking/mocking-with-mockk/SKILL.md`.
- The user is wiring Hilt to swap a binding — combine this skill with `../../strategies/applying-testing-strategies/SKILL.md` (Hilt section).
- The user is choosing what to test, not how — use `../../concepts/choosing-what-to-test/SKILL.md`.
- The user is debugging a Robolectric shadow that throws on a specific API — use `../../../jvm-tests/robolectric/using-robolectric-correctly/SKILL.md`.

## Prerequisites

- A test target whose dependencies are injected (constructor injection or interface-based DI). Doubles cannot be substituted into hard-wired singletons without refactor.
- `mockito-core` + `mockito-kotlin` or `io.mockk:mockk` on `testImplementation` if mocks/stubs/spies are needed. See `../../strategies/organizing-test-source-sets/SKILL.md`.
- For shadows: `org.robolectric:robolectric` on `testImplementation` plus `testOptions.unitTests.includeAndroidResources = true`.

## The six doubles (verbatim definitions)

All six definitions below — Fake, Mock, Stub, Spy, Dummy, and Shadow — are verbatim from `developer.android.com/training/testing/fundamentals/test-doubles`. Shadow is the Robolectric-specific double Google's page calls out alongside the classic five.

### Fake — preferred

> "**Fake**: A test double that has a 'working' implementation of the class, but it is implemented in a way that makes it good for tests but unsuitable for production. Example: an in-memory database. Fakes don't require a mocking framework and are lightweight. **They are preferred.**"
> — `developer.android.com/training/testing/fundamentals/test-doubles`

### Mock — for interaction verification

> "**Mock**: A test double that behaves how you program it to behave and that has expectations about its interactions. Mocks will fail tests if their interactions don't match the requirements that you define. Mocks are usually created with a mocking framework to achieve all this. Example: Verify that a method in a database was called exactly once."
> — `developer.android.com/training/testing/fundamentals/test-doubles`

### Stub — canned answers, no expectations

> "**Stub**: A test double that behaves how you program it to behave but doesn't have expectations about its interactions. Usually created with a mocking framework. **Fakes are preferred over stubs for simplicity.**"
> — `developer.android.com/training/testing/fundamentals/test-doubles`

### Spy — wraps a real object

> "**Spy**: A wrapper over a real object which also keeps track of some additional information, similar to mocks. They are usually avoided for adding complexity. **Fakes or mocks are therefore preferred over spies.**"
> — `developer.android.com/training/testing/fundamentals/test-doubles`

### Dummy — passed but never used

> "**Dummy**: A test double that is passed around but not used, such as if you just need to provide it as a parameter. Example: an empty function passed as a click callback."
> — `developer.android.com/training/testing/fundamentals/test-doubles`

### Shadow — Robolectric's fake-via-bytecode

> "**Shadow**: Fake used in Robolectric."
> — `developer.android.com/training/testing/fundamentals/test-doubles`

A shadow replaces an Android framework class with a JVM implementation at the bytecode level — `ShadowSystemClock`, `ShadowLooper`, `ShadowApplication`. It is a fake by intent (working implementation, unsuitable for production), but uniquely scoped to Android framework classes the developer cannot otherwise substitute. See `../../../jvm-tests/robolectric/using-robolectric-correctly/SKILL.md`.

## Preference order (cite this in PR reviews)

Three verbatim quotes from the same page form the preference order:

1. > "Fakes don't require a mocking framework and are lightweight. **They are preferred.**"
2. > "**Fakes are preferred over stubs** for simplicity."
3. > "**Fakes or mocks are therefore preferred over spies.**"

Synthesized:

```
Fake > Mock ≈ Stub > Spy
                 ↑ (mock when verifying interactions; stub when only canned data)
```

Dummies are not in the ordering — they are placeholders, not real test logic. Shadows are a fake-by-construction reserved for framework classes.

## Concrete Android examples

### Fake — `FakeUserRepository` with in-memory map

The `/test-doubles` page renders a sample for a fake. **MUST NOT** copy the page's example verbatim — it contains the typo `val const UserAlice = User("Alice")` (correct Kotlin is `const val`, not `val const`). Verified in `tasks/research/R8-android-fundamentals.md`. Use this corrected, expanded version:

```kotlin
// RIGHT — fake repository with an in-memory map
class FakeUserRepository : UserRepository {
    private val users = mutableMapOf<UserId, User>()

    override suspend fun get(id: UserId): User? = users[id]

    override suspend fun save(user: User) {
        users[user.id] = user
    }

    override fun observeAll(): Flow<List<User>> = flow {
        while (currentCoroutineContext().isActive) {
            emit(users.values.toList())
            delay(100)
        }
    }

    // Test-only seeding helpers; production interface does NOT expose these.
    fun seed(vararg users: User) {
        users.forEach { this.users[it.id] = it }
    }
    fun clear() = users.clear()
}

@Test
fun loadsSeededUser() = runTest {
    val repo = FakeUserRepository().apply { seed(User(UserId("u1"), "Alice")) }
    val vm = UserViewModel(repo)
    vm.load(UserId("u1"))
    assertEquals("Alice", vm.state.value.name)
}
```

The fake compiles into the test source set; production code never sees it. There are no `every { }` blocks, no mock initialization, no verification ceremony.

### Mock — `mockk<UserRepository>()` for interaction verification

Reach for a mock when *the interaction itself is the SUT*, not the resulting state. Example: a sync engine that must call `markDirty(id)` exactly once after a save.

```kotlin
@Test
fun save_marksDirty_exactlyOnce() = runTest {
    val repo = mockk<UserRepository>(relaxUnitFun = true)
    val syncQueue = mockk<SyncQueue>(relaxUnitFun = true)
    val sut = UserSaveOrchestrator(repo, syncQueue)

    sut.save(User(UserId("u1"), "Alice"))

    coVerify(exactly = 1) { syncQueue.markDirty(UserId("u1")) }
}
```

Mockito-kotlin equivalent (the canonical Android pattern per CORPUS Section G.3 — "400+ Kotlin files import org.mockito, zero import io.mockk" in the androidx checkout):

```kotlin
@Test
fun save_marksDirty_exactlyOnce() = runTest {
    val repo: UserRepository = mock()
    val syncQueue: SyncQueue = mock()
    val sut = UserSaveOrchestrator(repo, syncQueue)

    sut.save(User(UserId("u1"), "Alice"))

    verify(syncQueue).markDirty(UserId("u1"))
    verifyNoMoreInteractions(syncQueue)
}
```

### Stub — canned data, no verification

Use when the only thing the test needs is a return value, *and* you cannot conveniently write a fake.

```kotlin
@Test
fun loadsCachedAvatarUrl() = runTest {
    val avatarApi: AvatarApi = mock()
    whenever(avatarApi.getUrl(UserId("u1"))).thenReturn("https://cdn/u1.png")

    val sut = AvatarPresenter(avatarApi)
    sut.load(UserId("u1"))

    assertEquals("https://cdn/u1.png", sut.url.value)
    // Note: no verify {} — this is a stub, not a mock.
}
```

If `AvatarApi` had multiple methods or the test needed to seed many calls, swap the stub for a `FakeAvatarApi` per the preference order.

### Spy — wrap a real object (avoid)

Spies are listed as "usually avoided for adding complexity". A legitimate use is partial replacement of a hard-to-fake class where the developer cannot justify a full fake. Even then, refactor the class first if possible.

```kotlin
// WORKS but DISCOURAGED
val realCache = LruCache<String, Bitmap>(8)
val spied = spyk(realCache)
every { spied.evictAll() } answers { /* no-op for the test */ }
```

The spy lets the test override one method while keeping the rest real. The cost is hidden coupling: the test now relies on `LruCache`'s real behavior for everything except `evictAll`, and any refactor of either side breaks tests in non-obvious ways.

### Dummy — placeholder argument

```kotlin
val noOpClickHandler: () -> Unit = {}   // dummy — type-checks, never invoked
val sut = ClickableComponent(onClick = noOpClickHandler)
sut.bind(...)  // test does not interact with onClick at all
```

### Shadow — `ShadowSystemClock` as a fake-via-bytecode

When the SUT calls `SystemClock.uptimeMillis()` directly, abstracting it would require refactor; Robolectric ships a shadow that the test drives explicitly:

```kotlin
@RunWith(AndroidJUnit4::class)
class TimerTest {
    @Test
    fun reportsElapsedTime() {
        ShadowSystemClock.setCurrentTimeMillis(0)
        val timer = Timer()
        ShadowSystemClock.advanceBy(Duration.ofMillis(1500))
        assertEquals(1500L, timer.elapsed())
    }
}
```

`ShadowSystemClock` IS a fake (in-memory implementation, unsuitable for production) — it is just delivered as a Robolectric shadow because the production code calls a `final` Android framework method that no DI seam can intercept.

## Decision matrix

```
The intent of the test                                     Pick
----------------------------------------------------------------
Need a working substitute for a class with state           Fake
  (in-memory DB, in-memory user map, in-memory clock)
Need to verify a specific interaction happened             Mock
  ("markDirty was called once with id u1")
Need a canned return value, no behavior, no verification   Stub
  (and a Fake would be heavyweight for this one method)
Need partial real / partial fake on the same class         Spy (last resort; refactor preferred)
Need a placeholder so a parameter type-checks              Dummy
Need to substitute an Android framework class (final API)  Shadow (Robolectric)
```

If two doubles fit, pick the one earlier in the preference list.

## Patterns

### Pattern: WRONG vs RIGHT — fake vs mock for state

```kotlin
// WRONG — using a mock to provide state
@Test
fun loadsAndDisplaysFirstUser() = runTest {
    val repo = mockk<UserRepository>()
    coEvery { repo.get(UserId("u1")) } returns User(UserId("u1"), "Alice")
    coEvery { repo.get(UserId("u2")) } returns User(UserId("u2"), "Bob")
    coEvery { repo.observeAll() } returns flowOf(
        listOf(User(UserId("u1"), "Alice"), User(UserId("u2"), "Bob"))
    )

    val vm = UserViewModel(repo)
    vm.load(UserId("u1"))
    assertEquals("Alice", vm.state.value.name)
}
// WRONG because: the test uses a mocking framework purely to seed data. Per /test-doubles:
// "Fakes are preferred over stubs for simplicity" and stubs are at least the lighter form
// of mock-as-data-source. A FakeUserRepository compiles, has no every {} ceremony, and
// the seed helper is reusable across tests.
```

```kotlin
// RIGHT — fake provides state; ViewModel test asserts state
@Test
fun loadsAndDisplaysFirstUser() = runTest {
    val repo = FakeUserRepository().apply {
        seed(User(UserId("u1"), "Alice"), User(UserId("u2"), "Bob"))
    }
    val vm = UserViewModel(repo)
    vm.load(UserId("u1"))
    assertEquals("Alice", vm.state.value.name)
}
```

### Pattern: WRONG vs RIGHT — verification on a fake

```kotlin
// WRONG — verifying a method call on a fake
@Test
fun save_callsRepository() = runTest {
    val repo = FakeUserRepository()
    val sut = UserSaveOrchestrator(repo)
    sut.save(User(UserId("u1"), "Alice"))

    // Now the test wants to verify save() was called. Fakes don't track calls.
    // Tempting fix: switch to spyk(repo). DON'T.
    val tracking = spyk(repo)
    coVerify { tracking.save(any()) }
}
// WRONG because: this confuses "did the right state result?" (fake territory) with
// "was the right method called?" (mock territory). Pick one. If the test is verifying
// state, assert on repo.get(UserId("u1")) instead. If the test is verifying interaction,
// drop the fake and use a mock.
```

```kotlin
// RIGHT — fake answers state assertions
@Test
fun save_persistsUser() = runTest {
    val repo = FakeUserRepository()
    val sut = UserSaveOrchestrator(repo)
    sut.save(User(UserId("u1"), "Alice"))

    assertEquals(User(UserId("u1"), "Alice"), repo.get(UserId("u1")))
}

// RIGHT — mock answers interaction assertions (separate test)
@Test
fun save_emitsSyncSignal() = runTest {
    val repo: UserRepository = mock()
    val syncQueue: SyncQueue = mock()
    val sut = UserSaveOrchestrator(repo, syncQueue)
    sut.save(User(UserId("u1"), "Alice"))

    verify(syncQueue).markDirty(UserId("u1"))
}
```

### Pattern: WRONG vs RIGHT — overusing a spy

```kotlin
// WRONG
val cache = spyk(LruCache<String, Bitmap>(8))
every { cache.put(any(), any()) } answers {
    // intercept and log
    println("put: ${invocation.args[0]}")
    callOriginal()
}
```
WRONG because: per `/test-doubles`, spies "are usually avoided for adding complexity. Fakes or mocks are therefore preferred over spies". The interceptor here belongs in a fake LRU implementation that records calls in a list the test can assert on.

```kotlin
// RIGHT
class RecordingCache : Cache<String, Bitmap> {
    val puts = mutableListOf<Pair<String, Bitmap>>()
    override fun put(key: String, value: Bitmap) { puts += key to value }
    override fun get(key: String): Bitmap? = null
}
```

### Pattern: WRONG vs RIGHT — Hilt + fake binding

```kotlin
// WRONG — production module with no test override
@Module @InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds abstract fun bind(impl: RealUserRepository): UserRepository
}

@HiltAndroidTest
class HomeScreenTest {
    @Inject lateinit var repo: UserRepository      // real repo, real DB, slow + non-hermetic
    // ...
}
```

```kotlin
// RIGHT — replace at the module boundary
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [RepositoryModule::class],
)
abstract class FakeRepositoryModule {
    @Binds abstract fun bind(fake: FakeUserRepository): UserRepository
}
```

The fake compiles only into the test source set; production graph is untouched. See `../../strategies/applying-testing-strategies/SKILL.md` for the full Hilt rule-ordering pattern.

## Mandatory rules

- **MUST** prefer fakes over mocks/stubs for substituting *state-bearing* dependencies. Per `/test-doubles`: "Fakes ... are preferred".
- **MUST** prefer mocks over stubs/spies for *interaction* assertions. Stubs are mocks-without-verification; if no interaction is being verified, switch to a fake.
- **MUST NOT** use a spy unless every other double is impractical and the cost of the partial-real coupling is documented in the test.
- **MUST NOT** verify framework or library calls. (See `../../concepts/choosing-what-to-test/SKILL.md` "What NOT to test".)
- **MUST NOT** copy the `FakeUserRepository` snippet from `/test-doubles` verbatim — it contains the syntax error `val const UserAlice = User(...)`. Use `private val` or `const val` (in that order).
- **PREFERRED:** keep one canonical `FakeXRepository` per repository interface in a `testFixtures` source set or a shared `core-testing` module; reuse it across ViewModel tests.
- **PREFERRED:** when a class is hard to fake, refactor it (extract an interface) before reaching for a spy.
- **PREFERRED:** Mockito-kotlin (`mock<T>()`, `whenever`, `argumentCaptor`) for new code on Android — it is the de-facto standard in `androidx/` itself per CORPUS §G.3. Reach for MockK only when Kotlin-specific features (coroutines, top-level functions, sealed mocking) demand it.

## Verification

- [ ] Every `mockk<X>()` / `mock<X>()` in the suite either has a `verify { }` / `verify(x)` call (genuine mock) or has been replaced by a `FakeX` (was actually a fake-as-mock anti-pattern).
- [ ] No test uses `spyk(x)` / `spy(x)` without an inline comment justifying why a fake or mock is impractical.
- [ ] Each repository interface has at most one canonical `FakeXRepository` shared across tests.
- [ ] No production code references `Fake*` classes (the test source set never leaks into release).
- [ ] Hilt-based UI tests use `@TestInstallIn` or `@UninstallModules` + `@BindValue` to swap real bindings for fakes — never `every { }` on a real injected instance.
- [ ] No copy of the broken `val const UserAlice` Kotlin from the `/test-doubles` page exists in the repo.

## References

- `developer.android.com/training/testing/fundamentals/test-doubles` — verbatim definitions, the three preference quotes, the Hilt cross-link.
- `developer.android.com/training/testing/local-tests` — "Caution: Complex mocks should be avoided. Instead, you can use different types of test doubles such as fakes, or Robolectric shadows".
- `developer.android.com/training/dependency-injection/hilt-testing` — `@TestInstallIn`, `@UninstallModules`, `@BindValue`, `HiltAndroidRule` ordered to execute first.
- `tasks/research/R8-android-fundamentals.md` — verbatim test-double quotes plus the noted typo (`val const UserAlice`) on Google's page.
- `developer.android.com/develop/ui/compose/testing` — Compose-side test doubles (cross-cutting).
- *Software Engineering at Google*, ch. 13 ("Test Doubles") — deeper rationale for preferring fakes. https://abseil.io/resources/swe-book
- Sibling skills: `../../concepts/understanding-the-testing-pyramid/SKILL.md`, `../../concepts/choosing-what-to-test/SKILL.md`, `../../strategies/applying-testing-strategies/SKILL.md`, `../../strategies/organizing-test-source-sets/SKILL.md`.
- Cross-category (framework mechanics): `../../../jvm-tests/mocking/mocking-with-mockito/SKILL.md`, `../../../jvm-tests/mocking/mocking-with-mockk/SKILL.md`, `../../../jvm-tests/robolectric/using-robolectric-correctly/SKILL.md`, `../../../jvm-tests/coroutines/testing-coroutines-with-runtest/SKILL.md`, `../../../jvm-tests/coroutines/testing-flows-with-turbine/SKILL.md`.
