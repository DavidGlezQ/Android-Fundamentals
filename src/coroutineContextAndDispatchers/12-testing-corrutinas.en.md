# Coroutine Testing / Testing de corrutinas

[Versión en español](12-testing-corrutinas.es.md)

---

## 1. Why testing coroutines is different

A regular test runs synchronously. Coroutines suspend, use `delay`, and depend on a dispatcher - if you test with the real dispatcher (`Dispatchers.Main`, which doesn't exist on a test JVM), the test crashes or ends up waiting on real time (a `delay(5000)` would make the test actually take 5 seconds).

`kotlinx-coroutines-test` solves this with **test dispatchers** that control time virtually.

---

## 2. `runTest`: the replacement for `runBlocking` in tests

`runTest` creates a special coroutine scope where time is **virtual**: `delay` calls don't actually wait, they skip instantly, but the logical execution order is respected.

```kotlin
@Test
fun `when I load the user, the state becomes Success`() = runTest {
    val viewModel = UserViewModel(fakeRepo)
    viewModel.loadUser()
    advanceUntilIdle()   // advances virtual time until there's no pending work left
    assertEquals(UiState.Success(fakeUser), viewModel.state.value)
}
```

`runTest` also detects coroutines that got stuck (never finished) and fails the test explicitly, instead of leaving it hanging forever like `runBlocking` would.

**Virtual time control:**
- `advanceUntilIdle()` - advances until there's no more pending work in the queue.
- `advanceTimeBy(ms)` - advances a specific amount of virtual time.
- `runCurrent()` - runs tasks already scheduled for the current time, without advancing the clock.

---

## 3. `MainDispatcherRule`: replacing `Dispatchers.Main` in tests

Production code usually uses `viewModelScope`, which internally uses `Dispatchers.Main`. On a test JVM (without real Android) `Dispatchers.Main` isn't initialized and throws an exception. The fix is a JUnit Rule that replaces it with a test dispatcher before each test:

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class MainDispatcherRule(
    private val testDispatcher: TestDispatcher = StandardTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

Usage in a test:

```kotlin
class UserViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @Test
    fun `test with viewModelScope works correctly`() = runTest {
        // viewModelScope.launch now uses the test dispatcher, not the real one
    }
}
```

**Why it's a Rule and not loose code:** it guarantees `Dispatchers.Main` is reset after each test (in `finished`), preventing one test from leaving the test dispatcher installed and breaking the following tests.

---

## 4. `StandardTestDispatcher` vs `UnconfinedTestDispatcher`

Two ways to control how coroutines run in the test:

**`StandardTestDispatcher`**: launched coroutines **don't run immediately** - they stay queued until `advanceUntilIdle()` or similar is explicitly called. Gives precise control over ordering, useful to verify intermediate states (e.g. that the state goes through `Loading` before `Success`).

**`UnconfinedTestDispatcher`**: coroutines run **immediately**, eagerly, with no need to advance time manually. Simpler, but you lose the ability to easily verify intermediate states.

```kotlin
@Test
fun `with StandardTestDispatcher I can verify the Loading state`() = runTest {
    val viewModel = UserViewModel(fakeRepo)
    viewModel.loadUser()
    assertEquals(UiState.Loading, viewModel.state.value)   // haven't advanced time yet
    advanceUntilIdle()
    assertEquals(UiState.Success(fakeUser), viewModel.state.value)
}
```

`runTest` uses `StandardTestDispatcher` by default.

---

## 5. Testing Flows with Turbine

Manually collecting a `Flow` in a test (with `toList()` or similar) is awkward because you need to know when to stop collecting. **Turbine** provides an API built for this:

```kotlin
@Test
fun `the flow emits Loading and then Success`() = runTest {
    viewModel.state.test {
        assertEquals(UiState.Loading, awaitItem())
        viewModel.loadUser()
        assertEquals(UiState.Success(fakeUser), awaitItem())
        cancelAndIgnoreRemainingEvents()   // the flow is hot, doesn't finish on its own
    }
}
```

`awaitItem()` suspends until the next emission, `awaitComplete()` waits for the flow to finish, `awaitError()` waits for an exception. Since `StateFlow`/`SharedFlow` are hot and never finish on their own, you always need to explicitly close the collection with `cancelAndIgnoreRemainingEvents()` or similar at the end of the `test { }` block.

---

## 6. The AAA pattern applied to coroutines

**Arrange, Act, Assert** remains the structure, except "Act" usually needs `advanceUntilIdle()` before "Assert":

```kotlin
@Test
fun `when loading fails, the state becomes Error`() = runTest {
    // Arrange
    val fakeRepo = FakeUserRepository(shouldFail = true)
    val viewModel = UserViewModel(fakeRepo)

    // Act
    viewModel.loadUser()
    advanceUntilIdle()

    // Assert
    assertTrue(viewModel.state.value is UiState.Error)
}
```

---

## 7. JUnit 4 vs JUnit 5 (what changes for coroutines)

The `runTest`/`MainDispatcherRule` mechanics are the same in both, but how rules are declared changes:

- **JUnit 4**: `@get:Rule val mainDispatcherRule = MainDispatcherRule()` (uses `TestWatcher`, as shown above).
- **JUnit 5**: no `@Rule` - you use an `Extension` (`@ExtendWith(MainDispatcherExtension::class)`) that implements `BeforeEachCallback`/`AfterEachCallback` instead of `TestWatcher`.

In Android, JUnit 4 remains more common due to its integration with Android tooling (Robolectric, AndroidX Test), though JUnit 5 is supported.

---

## 8. Mocking dependencies: test doubles

To avoid depending on real Retrofit/Room in a unit test:

```kotlin
class FakeUserRepository(private val shouldFail: Boolean = false) : UserRepository {
    override suspend fun getUser(): User {
        if (shouldFail) throw IOException("network error")
        return User("1", "Ana")
    }
}
```

A **fake** (a simplified real implementation, like above) is usually preferred over a **mock** (`MockK`/`Mockito`) when the logic is simple, because it's more readable and doesn't depend on configuring expectations. For more complex cases (verifying a method was called N times, with certain arguments), `MockK` is the standard in Kotlin.

```kotlin
val repo = mockk<UserRepository>()
coEvery { repo.getUser() } returns fakeUser   // coEvery: MockK's version for suspend fun
```

---

## Summary - quick decision table

| I need... | Tool |
|---|---|
| Run a coroutine in a test without waiting real time | `runTest` |
| Make viewModelScope work in a JVM test | `MainDispatcherRule` + `Dispatchers.setMain` |
| Verify an intermediate state (e.g. Loading) | `StandardTestDispatcher` (manual time control) |
| Have coroutines run immediately without advancing time | `UnconfinedTestDispatcher` |
| Test emissions from a Flow/StateFlow | Turbine (`.test { awaitItem() }`) |
| Mock a suspend fun | `MockK` with `coEvery` |
| A simple test double without a mocking library | Fake (simplified real implementation) |

---

## Interview phrases

- "runTest uses virtual time: delays get skipped, but logical order is respected."
- "MainDispatcherRule replaces Dispatchers.Main with a TestDispatcher, because Main doesn't exist on a test JVM."
- "StandardTestDispatcher gives manual time control to verify intermediate states; UnconfinedTestDispatcher runs everything immediately."
- "Turbine avoids manually collecting a Flow - awaitItem suspends until the next emission."
- "A StateFlow is hot and never finishes on its own, so you have to explicitly close the collection in the test."

## Common interview traps

**"My test uses viewModelScope.launch and fails with an exception about Dispatchers.Main, why?"** - Because Dispatchers.Main isn't initialized on a test JVM without real Android. Fixed with MainDispatcherRule, which replaces it with a TestDispatcher.

**"Why use runTest instead of runBlocking in a test?"** - Because runTest uses virtual time (delays don't actually wait) and detects stuck coroutines, failing the test explicitly instead of leaving it hanging forever.

**"How do I test that a StateFlow emits Loading and then Success?"** - With Turbine: `viewModel.state.test { assertEquals(Loading, awaitItem()); ...; assertEquals(Success(x), awaitItem()) }`, closing with cancelAndIgnoreRemainingEvents because the flow is hot.
