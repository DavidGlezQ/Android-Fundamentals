# CoroutineContext & Dispatchers / CoroutineContext y Dispatchers a fondo

🌐 [Versión en español](07-concurrencia-context.md)

---

## 1. What a CoroutineContext is

A `CoroutineContext` is the **set of data that defines how a coroutine behaves**: which thread it runs on, who its Job is, its name, how it handles errors. It's a **collection of elements**, like a map where each element has a unique key.

The four main elements:
- **`Job`** → controls the lifecycle (cancel, wait, state).
- **`CoroutineDispatcher`** → which thread/pool it runs on (Main, IO, Default).
- **`CoroutineName`** → a name, useful for debugging.
- **`CoroutineExceptionHandler`** → how it handles uncaught exceptions.

```kotlin
launch(Dispatchers.IO + CoroutineName("fetch") + SupervisorJob()) {
    // context: IO dispatcher, name "fetch", a supervisor job
}
```

---

## 2. The `+` operator: how elements combine

Elements combine with `+`. Each element has a **unique key**, so combining two of the same type makes the **right one win** (it replaces the left one).

```kotlin
val context = Dispatchers.IO + CoroutineName("A")
val newContext = context + CoroutineName("B")  // "B" replaces "A"
// Result: Dispatchers.IO + CoroutineName("B")
```

Adding two dispatchers **doesn't combine them** — the second (same key) wins. Adding a dispatcher and a name do coexist (different keys).

---

## 3. Context inheritance: how a child inherits from the parent

When you launch a child coroutine, it **inherits the parent's context**, with two caveats:

1. Elements you **don't** specify in the child → are inherited from the parent.
2. The ones you **do** specify → override the parent's.
3. **Exception: the `Job` is NEVER inherited** — each coroutine creates its own child `Job` (to keep the parent-child hierarchy).

```kotlin
viewModelScope.launch(Dispatchers.IO) {   // parent: IO dispatcher
    launch {                               // child with no dispatcher specified
        // INHERITS IO from the parent
    }
    launch(Dispatchers.Default) {          // child with its own dispatcher
        // uses Default (overrides inheritance)
    }
}
```

That's why `viewModelScope.launch { }` runs on Main: `viewModelScope` has `Dispatchers.Main` in its context, and the child inherits it.

---

## 4. How `withContext` works underneath

`withContext(Dispatchers.IO)` **takes the current context and replaces the dispatcher** (by the `+` rule), runs the block in that modified context, and when done **returns to the previous context**.

```kotlin
suspend fun getData() {
    // current context: Main (viewModelScope)
    withContext(Dispatchers.IO) {
        // here the dispatcher is IO — the rest of the context is kept
        api.fetch()
    }
    // we're back on Main automatically
}
```

It doesn't create a new coroutine (no new child Job like in `launch`) — it just changes the context for that block and suspends until done.

---

## 5. The CoroutineDispatcher in depth

Technically a **`ContinuationInterceptor`** — it intercepts the coroutine every time it resumes and places it on the right thread.

- **`Dispatchers.Main`** → main thread (UI). Backed by the main looper's `Handler`.
- **`Dispatchers.IO`** → pool for blocking/I/O, up to 64 threads (shares the physical pool with Default).
- **`Dispatchers.Default`** → pool for CPU, as many threads as cores.
- **`Dispatchers.Unconfined`** → starts on the calling thread, jumps to whichever thread resumes it. Almost never used in production.

`Main.immediate`: if you're already on the main thread, it executes immediately without re-dispatching (avoids an unnecessary jump).

**IO/Default relationship:** they share the same physical thread pool; the difference is the **parallelism quota** (Default = cores, IO = up to 64). They aren't two separate pools, they're two quotas over the same pool.

---

## 6. Job inside the context

```kotlin
val job = launch {
    val myJob = coroutineContext[Job]
    println("active? ${myJob?.isActive}")
}
job.cancel()
```

`coroutineContext[Job]` uses the `Job` key to pull that element out. The context is queryable by key.

---

## 7. Real Android example

```kotlin
class DataViewModel(private val repo: Repo) : ViewModel() {

    fun processLargeDataset(data: List<Item>) {
        viewModelScope.launch {                    // inherits Main from viewModelScope
            _state.value = Loading

            val processed = withContext(Dispatchers.Default) {
                data.map { heavyTransform(it) }    // runs on Default (CPU)
            }
            // back on Main automatically

            _state.value = Success(processed)      // update UI on Main, safe
        }
    }
}
```

---

## 8. Dispatcher injection (senior detail)

Never hardcode `Dispatchers.IO` directly — inject dispatchers for testability:

```kotlin
interface DispatcherProvider {
    val main: CoroutineDispatcher
    val io: CoroutineDispatcher
    val default: CoroutineDispatcher
}

class Repo(private val dispatchers: DispatcherProvider) {
    suspend fun getData() = withContext(dispatchers.io) {  // injected, not hardcoded
        api.fetch()
    }
}
```

In tests, you pass a `StandardTestDispatcher` and control virtual time. See [12 — Testing](12-testing-corrutinas.en.md).

---

## 9. Common mistakes

**a) Thinking adding two dispatchers combines them.** `Dispatchers.IO + Dispatchers.Default` doesn't do both — the second (Default) wins because they share a key.

**b) Expecting the Job to be inherited.** It isn't; each coroutine has its own child Job.

**c) Hardcoding dispatchers.** Breaks testability.

**d) Forgetting that `withContext` returns to the previous context.** After a `withContext(IO)`, you go back to the previous dispatcher — you don't stay on IO.

---

## 10. Interview phrases

- *"The context is a map of elements indexed by key: Job, Dispatcher, Name, ExceptionHandler."*
- *"They combine with `+`; same type, the right one wins."*
- *"The child inherits everything from the parent except the Job, which is always its own."*
- *"withContext replaces the dispatcher for one block and returns to the previous one — doesn't create a new coroutine."*
- *"Inject dispatchers, don't hardcode them — that's what makes the code testable."*
- *"IO and Default are two quotas over the same thread pool, not two separate pools."*

---

## 11. Typical interview traps

**"What happens if you do `withContext(Dispatchers.IO + Dispatchers.Default)`?"**
Default wins, because both are dispatchers (same key) and the right one overrides. They don't run on both.

**"How does `withContext` work internally?"**
It takes the current context, replaces the element you pass, runs the block, and returns to the previous context. It doesn't create a new coroutine, that's why it's cheaper than `async`/`launch` for changing threads.
