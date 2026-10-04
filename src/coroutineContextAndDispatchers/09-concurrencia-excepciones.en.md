# Exception Handling / Manejo de excepciones

[Versión en español](09-concurrencia-excepciones.es.md)

---

## 1. Why exception handling in coroutines is different

In regular code, an exception bubbles up the call stack until something catches it. With coroutines there's an extra layer: exceptions also propagate through the **structured concurrency hierarchy** (see [06 - Scopes](06-concurrencia-scopes.en.md)), and the behavior changes depending on the builder (`launch` vs `async`) and the type of Job (`Job` vs `SupervisorJob`).

---

## 2. Regular `try/catch`: works, with caveats

A `try/catch` around a suspend call works exactly like in synchronous code:

```kotlin
viewModelScope.launch {
    try {
        val data = repo.getData()   // suspend fun that can throw
        _state.value = Success(data)
    } catch (e: IOException) {
        _state.value = Error(e.message)
    }
}
```

**The trap:** `CancellationException` is also an `Exception`. A generic `catch (e: Exception)` catches it by mistake and breaks cooperative cancellation (see [08 - Cancellation](08-concurrencia-cancelacion.en.md) section 6). It's always better to catch specific types, or rethrow `CancellationException` if you catch `Exception`.

---

## 3. `launch` vs `async`: where the exception explodes

This is the central distinction of the topic.

**`launch`: the exception propagates immediately**, as soon as it happens, up to the parent scope - no `await` or anything similar is needed for it to appear.

```kotlin
viewModelScope.launch {
    throw RuntimeException("boom")   // explodes right now, propagates to the parent
}
```

**`async`: the exception is stored inside the `Deferred`** and only thrown when `.await()` is called. If `await` is never called, the exception doesn't disappear - it still propagates to the parent via structured concurrency, but the point where a `try/catch` would catch it is different.

```kotlin
val deferred = async { throw RuntimeException("boom") }
// nothing exploded yet...
deferred.await()   // the exception comes out HERE
```

| | `launch` | `async` |
|---|---|---|
| When thrown | Immediately | When `.await()` is called |
| Where to catch it | `try/catch` around the `launch` or the code inside | `try/catch` around the `.await()` |

---

## 4. Propagation through the hierarchy: regular `Job`

With a regular `Job` (the one `coroutineScope` creates), an uncaught exception in a child **cancels the parent and all its siblings**, then gets rethrown upward.

```kotlin
coroutineScope {
    launch { throw Exception("boom") }       // fails
    launch { delay(1000); doWork() }         // gets cancelled, NEVER reaches doWork()
}
// the exception keeps propagating past this coroutineScope
```

This is "all or nothing": one failure brings down the whole group. It's the correct behavior when the tasks form an indivisible unit.

---

## 5. Propagation with `SupervisorJob`

With `SupervisorJob` (the one `viewModelScope` and `supervisorScope` use), a child's failure does **not** cancel its siblings - each one is independent.

```kotlin
supervisorScope {
    launch { throw Exception("boom") }       // fails, but only this child
    launch { delay(1000); doWork() }         // DOES get to run
}
```

This is what enables the pattern of "several sibling `launch` calls, each with its own state" seen in previous topics: one failing (e.g. balance didn't load) doesn't bring down the others (profile and transactions keep loading fine).

**Important:** even though the sibling survives, the failed child's exception **still needs to be handled** - if it isn't caught with `try/catch` inside that `launch`, it eventually reaches the `CoroutineExceptionHandler` (see next section), or crashes the app if there isn't one.

---

## 6. `CoroutineExceptionHandler`: the final safety net

It's an element of the `CoroutineContext` (see [07 - Context and Dispatchers](07-concurrencia-context.en.md)) that catches **unhandled** exceptions that finish propagating through the whole hierarchy, before they crash the app. It's the last catch point, not a replacement for `try/catch`.

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    Log.e("MyApp", "Unhandled exception: ${exception.message}")
}

viewModelScope.launch(handler) {
    throw RuntimeException("boom")   // no try/catch, falls into the handler
}
```

**Key rules:**
- Only works with `launch`, not with `async` - in `async` the exception lives in the `Deferred` and is thrown at `.await()`, the handler doesn't intercept it there.
- Only catches exceptions that **reach the root** of the coroutine tree (usually installed on the top-level scope, not on each child `launch`).
- Doesn't prevent the `Job` from being cancelled - it just gives a centralized place to log/report before the exception is lost.

---

## 7. `runCatching` as a functional alternative

To avoid nested try/catch blocks, it's common to wrap a suspend call in `runCatching`, which returns a `Result<T>`:

```kotlin
viewModelScope.launch {
    val result = runCatching { repo.getData() }
    _state.value = result.fold(
        onSuccess = { Success(it) },
        onFailure = { Error(it.message) }
    )
}
```

**Watch out:** `runCatching` also catches `CancellationException` by default (it inherits from `Throwable`, not treated specially). If used inside a coroutine that can be cancelled, it's better to rethrow it:

```kotlin
val result = runCatching { repo.getData() }
    .onFailure { if (it is CancellationException) throw it }
```

---

## 8. Real Android example: putting it all together

```kotlin
class AccountViewModel(private val repo: AccountRepository) : ViewModel() {

    private val handler = CoroutineExceptionHandler { _, e ->
        Log.e("AccountVM", "Unhandled error", e)
    }

    private val _balance = MutableStateFlow<BalanceState>(BalanceState.Loading)
    val balance = _balance.asStateFlow()

    fun loadBalance() {
        // viewModelScope's implicit SupervisorJob: this launch doesn't bring down others
        viewModelScope.launch(handler) {
            try {
                _balance.value = BalanceState.Success(repo.getBalance())
            } catch (e: IOException) {
                _balance.value = BalanceState.Error(e.message)
            }
            // CancellationException is NOT caught here because it doesn't match IOException,
            // so cooperative cancellation keeps working fine
        }
    }
}
```

---

## Summary - quick decision table

| I need... | Tool |
|---|---|
| Catch an expected error (network, IO) in a launch | Specific `try/catch` around the code |
| Catch the error from a task with a result | `try/catch` around the `.await()` |
| A failed child not to bring down its siblings | `SupervisorJob` / `supervisorScope` / `viewModelScope` |
| A final safety net for uncaught errors | `CoroutineExceptionHandler` (only with `launch`) |
| Avoiding nested try/catch | `runCatching` + `fold` (watch out for CancellationException) |

---

## Interview phrases

- "launch propagates the exception immediately; async stores it in the Deferred until the await."
- "With a regular Job, a failure cancels the whole group; with SupervisorJob, it only affects the failed child."
- "CoroutineExceptionHandler is the final safety net, only works with launch, doesn't replace try/catch."
- "CancellationException inherits from Exception - a generic catch without rethrowing it breaks cooperative cancellation."

## Common interview traps

**"I have a generic try/catch(Exception) around a suspend call, what's the problem?"** - It also catches `CancellationException`, breaking the coroutine's cooperative cancellation. You should catch specific types or rethrow it.

**"Why doesn't a CoroutineExceptionHandler placed on an async work?"** - Because in async the exception is stored in the Deferred and only thrown at await - the handler never intercepts it there.

**"Three sibling launch calls in viewModelScope, one throws an uncaught exception, what happens?"** - Since viewModelScope uses SupervisorJob, the other two keep running normally; the failed one's exception propagates up to the CoroutineExceptionHandler if one is installed, or crashes the app if there isn't one.
