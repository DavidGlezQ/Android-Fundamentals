# Coroutine Builders / Builders de corrutinas

🌐 [Versión en español](05-concurrencia-builders.md)

---

## 1. What a builder is

A **builder** is a function that **creates and launches** a coroutine. It's the bridge between regular code and coroutine code: you can't call a `suspend fun` from regular code, but a builder opens that door.

---

## 2. `launch` — fire and forget

Launches a coroutine that **returns no result**. Used for side effects: saving something, updating state, triggering an action. Returns a `Job`, a **handle** to control the coroutine (cancel it, wait for it), not its result.

```kotlin
fun main() = runBlocking {
    val job: Job = launch {
        delay(1000)
        println("Task done")
    }
    println("Coroutine launched")
    job.join()   // waits until it finishes (optional)
}
// Output: "Coroutine launched" ... (1s) ... "Task done"
```

What the `Job` gives you:
- `job.join()` → suspends until the coroutine finishes.
- `job.cancel()` → cancels it.
- `job.isActive` / `job.isCompleted` / `job.isCancelled` → its state.

---

## 3. `async` — when you need a result

`async` launches a coroutine that **does return a value**. Returns a `Deferred<T>` (a `Job` that also carries a result), and you get the value with `.await()`.

```kotlin
fun main() = runBlocking {
    val deferred: Deferred<Int> = async {
        delay(1000)
        42
    }
    println("Calculating...")
    val result = deferred.await()   // suspends until the result is ready
    println("Result: $result")
}
```

But the real value of `async` **isn't** launching a single task — `withContext` is better for that. Its point is **running several tasks overlapped and getting their results**.

```kotlin
suspend fun loadDashboard() = coroutineScope {
    val user = async { api.getUser() }      // starts now
    val posts = async { api.getPosts() }    // starts now, overlapping
    val notifications = async { api.getNotifications() }

    // all three are already running; now collect the results
    Dashboard(user.await(), posts.await(), notifications.await())
}
```

If each call takes 1s, sequentially it would be 3s; this way it's ~1s.

**Important:** if those three tasks are network calls, this is **I/O concurrency**, not CPU parallelism — it doesn't need `Dispatchers.Default`. The `IO` dispatcher lives inside the repository (main-safe), not in the `async`. `Default` would only be used if the `async` did CPU computation. See [04 — Fundamentals](04-concurrencia-fundamentos.en.md) sections 4 and 12.

---

## 4. The `launch` vs `async` contrast

| | `launch` | `async` |
|---|---|---|
| Returns | `Job` | `Deferred<T>` |
| Gives a result? | No | Yes (with `.await()`) |
| Use | Side effect | Concurrent work with a result |
| Exception | Propagates immediately to the scope | Thrown when `.await()` is called |

`launch` vs `async` does **not** change whether something is concurrent — that's decided by how you launch them (see 04-fundamentals). It changes **whether you need to collect the result**:

```kotlin
// launch: DON'T need a result (side effect)
launch { analytics.track("dashboard_opened") }

// async: DO need the result
val user = async { api.getUser() }
render(user.await())
```

---

## 5. `runBlocking` — the bridge from blocking code

`runBlocking` **blocks the current thread** until the coroutine inside finishes. It's the philosophical opposite of everything else (blocks instead of suspending).

```kotlin
fun main() = runBlocking {   // blocks the main thread until everything inside finishes
    launch { delay(1000); println("A") }
    launch { delay(500); println("B") }
}
```

Where it IS used:
- In a console app's `main()` function.
- In **tests** (though `runTest` is preferred today — see [12 — Testing](12-testing-corrutinas.en.md)).
- Occasional bridges with legacy blocking code.

Where it's NOT: **never in Android production**, because it would block the main thread and freeze the UI. In Android you use `viewModelScope`/`lifecycleScope`.

---

## 6. `withContext` — changing context, not creating a coroutine

`withContext` changes the context (typically the dispatcher) to execute a block and returns its result **sequentially** — suspends until it finishes. It doesn't create a new child coroutine like `launch`/`async`; it just changes the context for that block and suspends.

```kotlin
suspend fun getUser(): User = withContext(Dispatchers.IO) {
    api.fetchUser()
}
```

**`withContext` vs `async` for changing context:** if you just want to move a block to another dispatcher and use its result right away, `async` + immediate `await()` is an antipattern — all the concurrency machinery without gaining any concurrency.

```kotlin
// ANTIPATTERN: async + immediate await to change context
val user = async(Dispatchers.IO) { api.getUser() }.await()

// CORRECT: withContext for the same thing
val user = withContext(Dispatchers.IO) { api.getUser() }
```

`async` earns its place when you launch **several** tasks to run in parallel/concurrently and collect the results later.

```kotlin
// async shines here: two calls running CONCURRENTLY
suspend fun loadScreen() = coroutineScope {
    val user = async(Dispatchers.IO) { api.getUser() }
    val posts = async(Dispatchers.IO) { api.getPosts() }
    Screen(user.await(), posts.await()) // both ran at the same time
}
// with withContext they'd be sequential: getUser FINISHES before getPosts starts
```

| | `withContext` | `async` |
|---|---|---|
| Returns | The result directly | `Deferred<T>` (wait with `await`) |
| Execution | Sequential (suspends until done) | Concurrent (continues without waiting) |
| Creates new coroutine | No, just changes context | Yes, a child coroutine |
| Ideal use | Changing dispatcher for one block | Parallelizing/concurrency of several tasks |
| Overhead | Lower | Higher (coordinates a child) |

**Mental rule:** one task, change context, use the result right away → `withContext`. Several tasks you want overlapping → `async` + `await`. If you write `async { ... }.await()` on the same line, it should almost always be `withContext`.

---

## 7. Real Android example

`launch` is the builder you use 90% of the time in Android:

```kotlin
class OrderViewModel(private val repo: OrderRepository) : ViewModel() {

    private val _state = MutableStateFlow<OrderState>(OrderState.Idle)
    val state = _state.asStateFlow()

    // launch: side effect (update state), returns nothing
    fun placeOrder(order: Order) {
        viewModelScope.launch {
            _state.value = OrderState.Loading
            try {
                repo.submit(order)
                _state.value = OrderState.Success
            } catch (e: Exception) {
                _state.value = OrderState.Error(e.message)
            }
        }
    }
}
```

`async` when you need to **combine results** from several calls to build a screen:

```kotlin
fun loadProfileScreen() {
    viewModelScope.launch {                    // launch: the container
        _state.value = ProfileState.Loading
        try {
            // async: the two calls overlap inside the launch
            val profile = async { repo.getProfile() }
            val activity = async { repo.getRecentActivity() }
            _state.value = ProfileState.Success(
                profile.await(),
                activity.await()
            )
        } catch (e: Exception) {
            _state.value = ProfileState.Error(e.message)
        }
    }
}
```

Idiomatic pattern: **`launch` as container** (the side effect of updating state), and **`async` inside** to overlap the network calls.

And the repository, main-safe inside:

```kotlin
class Repo(private val api: Api) {
    suspend fun getProfile(): Profile = withContext(Dispatchers.IO) {
        api.getProfile()   // the dispatcher lives here, not in the ViewModel's async
    }
}
```

---

## 8. Common mistakes

**a) `async` + immediate `await()` instead of `withContext`.**

```kotlin
// Antipattern
val user = async { repo.getUser() }.await()
// Correct
val user = withContext(Dispatchers.IO) { repo.getUser() }
```

**b) Using `async` for tasks that are actually sequential (dependent).**

```kotlin
// Doesn't parallelize anything: posts NEEDS the user's id
val user = async { repo.getUser() }.await()
val posts = async { repo.getPosts(user.id) }.await()
// Better sequential and clear:
val user = repo.getUser()
val posts = repo.getPosts(user.id)
```

**c) Forgetting that `async`'s exception waits for `await()`.** With `launch`, an exception propagates immediately. With `async`, the exception is "stored" and **explodes when you call `.await()`**.

```kotlin
val deferred = async { throw RuntimeException("boom") }
// nothing exploded yet...
deferred.await()  // ← the exception throws HERE
```

**d) `runBlocking` on Android's main thread.** Freezes the UI. Never.

**e) Using `Dispatchers.Default` for network calls.** It's I/O concurrency, not CPU parallelism — the dispatcher goes in the repo with `IO`.

---

## 9. Interview phrases

- *"`launch` returns a Job (control); `async` returns a Deferred (result)."*
- *"The value of `async` is running overlapped tasks and getting results, not launching a single one."*
- *"The exception from `async` explodes at `await`, not when launched."*
- *"`runBlocking` blocks the thread — for main() and tests, never in Android production."*
- *"withContext changes context and suspends sequentially; async launches concurrency and returns a Deferred."*
- *"async + immediate await is an antipattern: all the concurrency machinery without gaining any concurrency — that's withContext."*

---

## 10. Typical interview traps

**"When does `async` add nothing?"**
With a single task + immediate `await()` (use `withContext`), and with sequentially dependent tasks (nothing to overlap). It only shines with independent tasks running at the same time.

**"You're shown `async(IO) { fetch() }.await()`, what would you improve?"**
Change it to `withContext(IO) { fetch() }`, because the immediate `async`+`await` adds no concurrency and does add the cost of creating and coordinating a child coroutine.

**"Does launch vs async change whether it's concurrent?"**
No. Concurrency comes from launching them without waiting between one and the other — two `launch` calls also overlap. The difference is whether you need to collect the result (async) or not (launch).
