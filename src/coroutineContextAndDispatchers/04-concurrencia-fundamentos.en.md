# Concurrency & Coroutines — Fundamentals / Concurrencia y Corrutinas — Fundamentos

[Versión en español](04-concurrencia-fundamentos.md)

---

## 1. The four concepts and how they relate

Not synonyms. Different layers of the same question: "how do I do several things?"

| Concept | What it is | Level |
|---|---|---|
| **Concurrency** | Several tasks in progress, taking turns | Model / structure |
| **Parallelism** | Several tasks at the exact same instant (multiple cores) | Model / execution |
| **Asynchronous code** | Not blocking while waiting | Technique to achieve concurrency |
| **Coroutines** | Kotlin's tool that uses async to give cheap concurrency | Implementation |

Rob Pike's summary:
> **Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once.**

---

## 2. The three ways to execute tasks (the key mental map)

This is what gets confused most. When you want to "do several things," there are **three** distinct scenarios, not two. The dividing line is two questions: **do the tasks depend on each other?** and **is the work waiting or computing?**

```
"Do several tasks"
│
├── Does one depend on the other's result?
│   │
│   ├── YES → SEQUENTIAL (you wait for one before launching the other, forced)
│   │
│   └── NO → they can overlap... what kind of work?
│            │
│            ├── WAITING (I/O: network, disk) → CONCURRENCY
│            │   E.g.: async { api.getUser() } + async { api.getPosts() }
│            │   Dispatcher: IO (or main-safe suspend)
│            │   Works on 1 core ✓
│            │
│            └── COMPUTATION (CPU) → PARALLELISM
│                E.g.: async(Default) { applyFilter(img) }
│                Dispatcher: Default
│                Needs multiple cores ✓
```

**The mental test to tell concurrency from parallelism:**
> *"Could this work the same on a single core?"*
> - Yes → it's **concurrency** (overlapping waits).
> - No, needs multiple cores → it's **parallelism** (overlapping real computation).

---

## 3. Concurrency

**Precise definition:** multiple tasks **in progress during the same period**, interleaving, but not necessarily executing at the same instant.

The key word isn't "at the same time" but **"in progress at once."** Several tasks are started and not yet finished; the system alternates between them. On a single thread, at any exact instant only one is executing code; the rest are suspended waiting their turn.

**Analogy:** a cook with three pots. Three dishes are in progress, but at that exact instant the cook is only stirring one pot; the other two are cooking on their own (suspended).

**Important clarifications:**
- Concurrency is **not** synonymous with asynchronous code. One is the concept, the other the technique that achieves it.
- Concurrency requires **independent** tasks (if they depend on each other, it's sequential).
- Concurrency does **not** require simultaneous execution. That's parallelism.
- It can happen on a **single core**.

---

## 4. Parallelism

**Definition:** multiple tasks executing **literally at the same instant**. Requires **multiple cores**.

Parallelism is a *way* of executing concurrent work when there are multiple cores. It needs two things together:
1. **CPU work** (real computation, not waiting).
2. **Multiple threads spread across multiple cores** (e.g. `Dispatchers.Default`).

If either is missing, you have concurrency at most, not parallelism.

**Analogy:** three cooks, each with their own pot. Several people, several tasks at the same time.

**The typical mistake:** believing several network calls with `async` are "parallelism." They're not — they're **I/O concurrency**. The calls are *waiting* at the same time, not *executing code* at the same time. Waiting isn't executing. They could run on a single core and work the same → by definition it's not parallelism.

**Parallelism without using YOUR CPU:** parallelism always needs CPU executing, but not necessarily yours. When you do concurrent I/O, your CPU is freed (suspended) while other hardware — a remote server, disk (DMA), GPU — does real parallel work. From your app's view it's concurrency; in the overall system there's parallelism, just not on your processor.

---

## 5. Sequential

**Definition:** one thing after another. You start task 1, wait for it to **finish completely**, and only then start task 2. There are never two tasks in progress at once.

```kotlin
suspend fun sequential() {
    val user = api.getUser()      // (1) start and WAIT for it to finish — 1s
    val posts = api.getPosts()    // (2) only starts now — another 1s
    // Total: 2 seconds
}
```

**Suspending is NOT the same as being concurrent.** Even if `getUser()` is `suspend` and frees the thread, the code is still sequential: line 2 doesn't start until line 1 returns its value.

> Suspending is about **the thread** (you don't block it). Sequential vs concurrent is about **your flow** (do you wait for one task before launching the other?). You can suspend and still be sequential.

**Sequential is NOT always a mistake.** It's correct and mandatory when there's **dependency** between tasks:

```kotlin
// MANDATORY sequential: posts needs the user's id
val user = api.getUser()              // need the user first
val posts = api.getPosts(user.id)     // because I use its id here → can't parallelize
```

**The rule:**
- **Independent** tasks → concurrent (launch all, wait after) — *optional*, you can still choose sequential for simplicity.
- **Dependent** tasks (one uses the other's result) → sequential, no choice.

**Watch the relationship between concurrency and dependency:** they're **opposites**.
- Concurrency happens when tasks **don't depend** on each other.
- Dependency is what **forces sequential**.
- If there's dependency, there's NO concurrency possible.

**Independence doesn't force using `async`.** It *enables*, it doesn't *force*. You can make independent calls sequentially — it's valid, just slower (takes the sum instead of the max).

**Valid ≠ equally fast.** Sequential gives the correct result, but takes the SUM of the times. Concurrent (with async) takes the MAX (the slowest one).

```
SEQUENTIAL          CONCURRENT
A: [==1s==]         A: [==1s==]
B:        [==1s==]  B: [==1s==]
   total: 2s           total: 1s
```

**Splitting into `suspend` functions does NOT change timing by itself.** A function is just a code wrapper. What decides timing is *how you call them* (direct = sequential, with `async` = concurrent), not where they live.

---

## 6. Asynchronous code vs multithreading

Both achieve concurrency, through different paths:

- **Multithreading:** you use multiple OS threads. Each can run on a different core → can give **real parallelism**. But threads are expensive.
- **Asynchronous code:** it's about **not blocking while waiting**. It can be achieved **even on a single thread** (JavaScript's model).

They're not the same: async is "don't wait blocked"; multithreading is "use multiple threads."

---

## 7. The distinction that defines everything: blocking vs suspending

**Blocking** a thread: the thread stays occupied doing nothing, waiting. It can't execute anything else. Wasteful.

**Suspending** a coroutine: the coroutine pauses and **frees the thread**, which becomes available for another coroutine. When what it was waiting for is ready, it resumes (perhaps on another thread).

```kotlin
// BLOCKING: the thread is frozen for 1 second, useless
fun blockingWork() {
    Thread.sleep(1000)   // the thread can do NOTHING else
}

// SUSPENDING: the coroutine pauses, the thread keeps working on others
suspend fun suspendingWork() {
    delay(1000)          // frees the thread while waiting
}
```

**The practical consequence:**

```kotlin
fun main() = runBlocking {
    // delay suspends: 100,000 run on few threads, no problem
    repeat(100_000) {
        launch { delay(1000) }
    }
    // If it were Thread.sleep (blocking), you'd need 100,000 threads → impossible
}
```

> **Suspending allows thousands of concurrent tasks on few threads, because the ones waiting don't occupy a thread.**

---

## 8. What a coroutine is

A coroutine is a **lightweight, suspendable task** that runs on threads but is managed by the Kotlin runtime, not the operating system.

- It's **lightweight**: you can have hundreds of thousands on few threads.
- It's **suspendable**: it can pause at certain points and resume, without blocking the thread while paused.
- It brings **structured concurrency**: it lives in a scope that handles its cancellation and errors.
- It gives **parallelism** only if the dispatcher spreads it across multiple cores.

**Thread vs coroutine:**

| | Thread | Coroutine |
|---|---|---|
| Managed by | The operating system | The Kotlin runtime |
| Memory cost | ~1-2 MB (stack) | Bytes (an object) |
| How many fit | Thousands | Hundreds of thousands |
| While waiting | Blocks (occupies the thread) | Suspends (frees the thread) |
| Cancellation | Manual, complicated | Cooperative, via scope |

**See also:** [11 — Threads vs Coroutines](11-threads-vs-coroutines.en.md) for the full detail of this contrast.

---

## 9. `suspend`: the heart of asynchronous code

A `suspend` function **can pause and resume**. It can only be called from another `suspend` function or from a coroutine.

```kotlin
suspend fun fetchUser(): User {
    delay(1000)              // suspension point: it can pause here
    return User("Ana")
}
```

**The most confused point:** `suspend` **does not mean "runs on another thread."** It means "this function can suspend at some point." Where it runs is decided by the dispatcher, not the word `suspend`. A `suspend fun` can run on the main thread — what it does is not *block* it when it suspends.

```kotlin
// This does NOT run in the background just because it's suspend
suspend fun wrong() {
    val data = heavyBlockingCall()  // blocks the current thread (maybe Main!)
}
// Needs explicit withContext to change thread
suspend fun right() = withContext(Dispatchers.IO) {
    heavyBlockingCall()
}
```

---

## 10. Synchronous, sequential, and asynchronous code — the three terms separated

These three terms get confused with each other. They're not synonyms:

| Term | What it describes | Example |
|---|---|---|
| **Synchronous** | The thread **blocks** while waiting | `Thread.sleep()`, blocking network call |
| **Sequential** | The **order**: one task after another | `val a = call1(); val b = call2()` (with or without suspend) |
| **Asynchronous** | The thread **doesn't block** while waiting | `delay()`, `suspend fun` with `withContext` |

You can have code that's **sequential but not synchronous** — exactly what coroutines achieve:

```kotlin
suspend fun load() {
    val user = api.getUser()      // sequential (waiting for the result)...
    val posts = api.getPosts()    // ...but NOT synchronous, because it suspends without blocking
}
```

This is sequential (one call after another, in order) but **not** synchronous (it doesn't block the thread). That's why the main thread doesn't freeze even though the code reads "top to bottom, one after another."

**Historical context:** before coroutines, to avoid synchronous code blocking the UI, **callbacks** were used — asynchronous, but they broke sequential readability ("callback hell"). Coroutines combine the best of both worlds: reads sequential, is asynchronous underneath.

---

## 11. Concurrency vs parallelism vs sequential in code

### Concurrency: independent waiting tasks (I/O)

```kotlin
suspend fun concurrent() = coroutineScope {
    val user = async { api.getUser() }    // launch, DON'T wait
    val posts = async { api.getPosts() }  // launch, DON'T wait (user is still running)
    user.await(); posts.await()           // now wait for both
    // total: ~1s (the two waits overlap). No Dispatchers.Default.
}
```

### Parallelism: CPU work across multiple cores

```kotlin
suspend fun processImages(imgs: List<Bitmap>) = coroutineScope {
    val results = imgs.map { img ->
        async(Dispatchers.Default) {   // HERE Default: it's CPU computation
            applyFilter(img)            // burns CPU → real parallelism across cores
        }
    }
    results.awaitAll()
}
```

### Sequential: one after another

```kotlin
// By dependency (mandatory):
val user = api.getUser()
val posts = api.getPosts(user.id)   // needs user.id

// By mistake (await right away): this DESTROYS the concurrency
val user = async { api.getUser() }.await()   // wait here
val posts = async { api.getPosts() }.await() // only now it starts → ~2s
```

### Summary of cases

| Case | Task type | Depend? | Dispatcher | Time (2 tasks of 1s) | What it is |
|---|---|---|---|---|---|
| network `async` + `await` at the end | Waiting (I/O) | No | `IO` / suspend | ~1s | **Concurrency** |
| `async(Default)` computation | CPU | No | `Default` | ~1s (on cores) | **Parallelism** |
| Calls with dependency | Either | Yes | whichever | ~2s | **Sequential** |
| `async{}.await()` right away | Either | No | whichever | ~2s | Sequential (by mistake) |

---

## 12. The role of `await` (often misunderstood)

`await` does **not create** concurrency or parallelism. It only **collects the result** of an `async`. What determines concurrency is **when** you call it:

```kotlin
// await at the end → CONCURRENT (launch everything, collect later)
val a = async { task1() }
val b = async { task2() }
a.await(); b.await()          // ~1s

// await right away → SEQUENTIAL (collect before launching the other)
val a = async { task1() }.await()   // wait here
val b = async { task2() }.await()   // only now it starts → ~2s
```

> Same `await`, opposite result. The **launch-vs-wait order** is what matters, not `await` itself. Concurrency = launch everything first, wait after.

**`async` does not equal parallelism.** `async` is for launching several overlapping tasks and collecting their results. Whether that's concurrency or parallelism is decided by the content and the dispatcher, not `async`.

- `async { api.getX() }` (network) → **I/O concurrency** (works on 1 core).
- `async(Dispatchers.Default) { calc() }` (CPU) → **real parallelism** (multiple cores).

**Combining ≠ depending.**
- **Combining** = gathering results from several tasks into something (a screen object). This is what `async` does.
- **Depending** = one task needs another's result to execute. This forces sequential.

The classic dashboard **combines** three results **without dependency** between them → that's why it can use `async`. If there were dependency, it couldn't.

---

## 13. `launch` vs `async` — which for the real case (screen with independent sections)

Real case (e.g. a bank account screen with balance, profile, and transactions, each with its own state):

**A single `launch` with direct calls → SEQUENTIAL**, even if "separated" into functions:

```kotlin
fun loadAccount() {
    viewModelScope.launch {
        val balance = repo.getBalance()      // wait 1s
        _balanceState.value = balance
        val profile = repo.getProfile()      // only now, 1s
        _profileState.value = profile
        // total: 2s+ — SEQUENTIAL
    }
}
```

**Three separate `launch` calls → CONCURRENT**, and it's the right pattern when each one updates its own state:

```kotlin
fun loadAccount() {
    viewModelScope.launch { _balance.value = repo.getBalance() }
    viewModelScope.launch { _profile.value = repo.getProfile() }
    viewModelScope.launch { _txns.value = repo.getTransactions() }
    // three concurrent coroutines, each updates its state as it arrives
}
```

**`async` + `await` → CONCURRENT**, but suits you when you need to combine the results into a single state:

```kotlin
viewModelScope.launch {
    val balance = async { repo.getBalance() }
    val profile = async { repo.getProfile() }
    _state.value = AccountState(balance.await(), profile.await())
}
```

**Decision rule:**

| Situation | Choice |
|---|---|
| Each section has its own state, appears as it arrives | Several `launch` |
| I need to combine everything into one final state | `async` + `await` |
| `launch` returns no value (Job) — if you need the result, it won't do | use `async` |

**Structured concurrency note:** with several `launch` calls in `viewModelScope` (which uses `SupervisorJob`), if one fails, the others **continue** — each needs its own try/catch. See [06 — Scopes and Structured Concurrency](06-concurrencia-scopes.en.md).

---

## 14. Common mistakes

1. **Thinking `suspend` changes threads.** It doesn't; it only marks that it can suspend. To change threads, `withContext`.
2. **Blocking inside a coroutine.** `Thread.sleep()` or a blocking API cancels out the advantage. Use `delay` and suspend functions.
3. **`runBlocking` in Android production.** Freezes the UI. Use `viewModelScope` / `lifecycleScope`.
4. **Thinking coroutines = automatic parallelism.** Without multiple threads/cores, two CPU tasks run in series.
5. **`async` + immediate `await()` for a single task.** It's extra machinery → use `withContext`.
6. **`async` for dependent tasks.** If the second needs the first's result, there's no parallelism to gain — it's sequential, write it as such.
7. **Confusing "several network calls" with parallelism.** It's I/O concurrency, not parallelism (doesn't use `Default`, would work on one core).
8. **Thinking sequential = synchronous.** They're different concepts — see section 10.
9. **Thinking splitting into suspend functions gives concurrency.** It doesn't — see section 5.
10. **A single `launch` with direct calls thinking it's concurrent.** It isn't — see section 13.

---

## 15. Interview phrases

**Concepts:**
- *"Concurrency is dealing with many things; parallelism is doing them at the same time."*
- *"You can have concurrency without parallelism: one core alternating tasks."*
- *"Overlapping waits is concurrency; overlapping computation on cores is parallelism."*
- *"Sequential = wait-then-launch; concurrent = launch-everything-then-wait."*
- *"Dependency between tasks is what forces sequential — it's the opposite of concurrency."*
- *"Synchronous blocks the thread while waiting; asynchronous doesn't. Sequential is the order, not the blocking."*

**Coroutines / suspend:**
- *"Suspending frees the thread; blocking occupies it without working."*
- *"`suspend` doesn't change threads — it only marks that the function can pause."*
- *"Thousands of coroutines on few threads because suspended ones don't occupy a thread."*

**async and await:**
- *"The value of `async` is parallelizing independent tasks, not launching a single one."*
- *"The exception from `async` explodes at `await`, not when launched."*
- *"`await` doesn't create concurrency — the launch-vs-wait order is what matters."*

---

## 16. Typical interview traps

**"Are coroutines parallelism or concurrency?"**
A **concurrency** model; they give **parallelism only if** the dispatcher spreads them across multiple cores (`Dispatchers.Default`). A single coroutine isn't parallel; many on a single thread are concurrent but not parallel.

**"Is a coroutine faster than a thread?"**
Not in computation: CPU work takes the same time. You gain **concurrency and efficient resource use**: more concurrent waiting with less memory and fewer context switches.

**"Are several network calls with `async` parallelism?"**
No — they're **I/O concurrency**. The calls wait at the same time, they don't execute code at the same time. They'd work on a single core, so it's not parallelism.

**"Is sequential code the same as synchronous code?"**
No. Sequential is the order; synchronous is whether the thread blocks. With `suspend`, the code is sequential (reads in order) but not synchronous (doesn't block, suspends).

**"Does splitting calls into separate `suspend` functions make them concurrent?"**
No. A function is just a code wrapper. What decides the timing is how you call them (direct = sequential; with `async` = concurrent), not where they live.

**"Three independent calls in a ViewModel, one launch or several?"**
Depends on whether you need to combine them into one state or each updates its own. A single `launch` with direct calls = sequential (common mistake). Several `launch` = concurrent, each independent. `async`+`await` = concurrent, combining results.
