# Cancellation / Cancelacion

[Versión en español](08-concurrencia-cancelacion.es.md)

---

## 1. Why cancellation matters

When the user closes a screen, navigates away, or explicitly cancels an action, in-progress tasks should stop. If they don't: network/CPU is wasted, the app might try to update a UI that no longer exists (crash), or resources leak.

Structured concurrency (see [06 - Scopes](06-concurrencia-scopes.en.md)) already solves part of this: when `viewModelScope` is cancelled, it cancels all its children. But **how fast and correctly** a coroutine is cancelled depends on how well-written the code inside it is. That's what this topic covers.

---

## 2. Cancellation is cooperative

This is the central idea. Cancelling a coroutine **doesn't kill it instantly** like a forced `Thread.interrupt()`. Instead, Kotlin marks the `Job` as cancelled, and it's **the responsibility of the code inside the coroutine** to check that state and stop.

```kotlin
val job = launch {
    repeat(1000) { i ->
        println("working $i")
        Thread.sleep(500)   // does NOT check cancellation, does NOT suspend
    }
}
delay(1300)
job.cancel()   // requests cancellation...
// ...but the loop above NEVER notices it, keeps running forever
```

If the code inside never suspends or checks the state, cancellation **never actually happens** even if it was "requested." That's why cancellation is said to be cooperative: the coroutine has to cooperate by checking whether it was cancelled.

---

## 3. Where it DOES cooperate: suspension points

The suspending functions from `kotlinx.coroutines` (`delay`, `yield`, and the builder functions like `withContext`) **already check cancellation automatically** at each suspension point. If the Job was cancelled, they throw a `CancellationException` right there.

```kotlin
val job = launch {
    repeat(1000) { i ->
        println("working $i")
        delay(500)   // DOES check cancellation at each suspension
    }
}
delay(1300)
job.cancel()   // now it DOES stop the loop, at the next delay()
```

**The practical rule:** if the code inside a coroutine does long work with no suspension point at all (a heavy CPU loop, a long computation with no `delay`/suspend calls), that coroutine **isn't cancellable** until that stretch finishes.

---

## 4. Cancelling code that doesn't suspend: `isActive` and `ensureActive`

For a CPU loop with no natural suspension point, you have to check the state manually.

```kotlin
val job = launch(Dispatchers.Default) {
    var i = 0
    while (isActive) {       // manual check: exits the loop if cancelled
        i++
        // pure CPU work, no suspend
    }
}
```

`isActive` is an extension property available inside a coroutine (comes from `CoroutineScope`/`Job`). It returns `false` once cancellation has been requested.

The alternative `ensureActive()` does the same but **throws** `CancellationException` instead of just returning a boolean - useful to cut off immediately without wrapping everything in an `if`:

```kotlin
val job = launch(Dispatchers.Default) {
    repeat(1_000_000) { i ->
        ensureActive()       // throws if cancelled, cuts off right here
        heavyComputation(i)
    }
}
```

---

## 5. Cleaning up resources on cancellation: `finally` and `NonCancellable`

When a coroutine is cancelled, Kotlin throws a `CancellationException` at the suspension point. That means a `try/finally` block **does run**, just like with any exception - it's the right place to release resources (close a file, a socket, log that it was cancelled).

```kotlin
val job = launch {
    try {
        repeat(1000) { i ->
            println("working $i")
            delay(500)
        }
    } finally {
        println("cleanup: closing resources")   // DOES run on cancellation
    }
}
```

**The problem:** inside that `finally`, the coroutine is already in the process of being cancelled, so any `suspend` function called there **will also immediately throw** `CancellationException` (because the Job is already cancelled). If the cleanup needs to do something suspendable (e.g. a final write to disk with a suspend function), wrap it in `withContext(NonCancellable)`:

```kotlin
val job = launch {
    try {
        repeat(1000) { i -> delay(500) }
    } finally {
        withContext(NonCancellable) {
            delay(1000)              // this DOES run, even though the Job is cancelled
            saveStateBeforeExit()    // a suspend fun that needs to run anyway
        }
    }
}
```

`NonCancellable` is a special context that ignores the parent Job's cancellation state - it should only be used for brief, critical cleanup, never to keep doing normal work.

---

## 6. `CancellationException` is not a regular error

When a coroutine is cancelled, a `CancellationException` is thrown internally. This is **intentional and propagates specially**: parent and child coroutines recognize it as "normal cancellation," not a real failure, so it **does not** trigger the `CoroutineExceptionHandler` or get treated as a crash.

**The trap:** if you catch `Exception` generically without rethrowing the `CancellationException`, you break cancellation:

```kotlin
// BAD: swallows the CancellationException, the coroutine never really cancels
try {
    delay(1000)
} catch (e: Exception) {
    log(e)   // this also catches CancellationException and hides it
}

// GOOD: catch specifically what you expect, let CancellationException pass through
try {
    delay(1000)
} catch (e: IOException) {
    log(e)
}
// or rethrow it explicitly if you catch Exception for some reason:
catch (e: Exception) {
    if (e is CancellationException) throw e
    log(e)
}
```

This is one of the most common interview traps: a generic `catch (e: Exception)` around a suspend call silently breaks cooperative cancellation.

---

## 7. Explicit user-triggered cancellation (real Android example)

Typical case: a search that gets cancelled if the user types something new before the previous one finishes.

```kotlin
class SearchViewModel(private val repo: SearchRepository) : ViewModel() {

    private var searchJob: Job? = null

    fun onQueryChanged(query: String) {
        searchJob?.cancel()              // cancel the previous search if still running
        searchJob = viewModelScope.launch {
            delay(300)                    // debounce
            val results = repo.search(query)
            _state.value = SearchState.Success(results)
        }
    }
}
```

Keeping a reference to the `Job` and manually cancelling it before launching a new one is the standard pattern for "the latest user action invalidates the previous one." (This same case is also solved with the `collectLatest` operator in Flows, which cancels automatically - see [10 - Flows](10-flows.en.md).)

---

## 8. Cascading cancellation: cancelling the parent

By structured concurrency, cancelling a parent scope automatically cancels all its active children - no need to cancel each one manually.

```kotlin
class MyViewModel : ViewModel() {
    fun loadEverything() {
        viewModelScope.launch { taskA() }   // child 1
        viewModelScope.launch { taskB() }   // child 2
    }
}
// When the ViewModel is destroyed, viewModelScope.cancel() is called automatically (onCleared),
// which cancels taskA and taskB automatically - no extra code needed
```

---

## Summary - quick decision table

| Situation | Tool |
|---|---|
| Code with `delay`/regular suspend calls | Cancels on its own, nothing to do |
| Pure CPU loop with no suspension points | `isActive` (manual check) or `ensureActive()` (throws) |
| Releasing resources on cancellation | `try/finally` |
| Cleanup that needs suspend code | `withContext(NonCancellable)` inside the `finally` |
| Catching exceptions without breaking cancellation | Don't catch generic `Exception` without rethrowing `CancellationException` |
| Invalidating a previous task with a new one | Store the `Job` and call `.cancel()` before relaunching |
| Cancelling several related tasks at once | Cancel the parent scope (structured concurrency) |

---

## Interview phrases

- "Cancellation is cooperative: it doesn't kill the coroutine instantly, it marks the Job and the code must check it."
- "Suspension points like delay check cancellation on their own; a pure CPU loop needs isActive or ensureActive."
- "finally runs on cancellation, but a suspend fun inside it still throws CancellationException - that's what NonCancellable is for."
- "A generic catch of Exception without rethrowing CancellationException breaks cooperative cancellation."

## Common interview traps

**"I have a heavy CPU loop inside a coroutine and job.cancel() doesn't stop it, why?"** - Because the loop has no suspension point where cancellation gets checked. You need to add `isActive` or `ensureActive()` inside the loop.

**"Why can a try/catch(Exception) break cancellation?"** - Because `CancellationException` inherits from `Exception`, and catching it without rethrowing makes the coroutine hierarchy think the task is still alive, breaking structured concurrency.

**"I need to save something to disk when a coroutine is cancelled, how do I do it if disk access is a suspend fun?"** - By wrapping that call in `withContext(NonCancellable)` inside a `finally` block, so it runs even though the Job is already cancelled.
