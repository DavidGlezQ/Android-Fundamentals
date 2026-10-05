# Threads vs Coroutines

[Versión en español](11-threads-vs-coroutines.es.md)

---

## 1. The mental model: who manages whom

A **thread** is a unit of execution managed by the operating system. It's relatively heavy: each one reserves memory for its stack (typically ~1-2 MB on the JVM) and switching between threads involves an OS context switch, which costs.

A **coroutine** is a unit of work that runs *on top of* threads, but it's managed by the Kotlin runtime, not the OS. It's lightweight: you can have hundreds of thousands with no problem.

**Coroutines don't replace threads, they run on top of them.** A dispatcher is what decides which thread or pool each coroutine runs on (see [07 - Context and Dispatchers](07-concurrencia-context.en.md)). It's not "threads vs coroutines" as enemies; coroutines are an abstraction layer on top that lets you squeeze more out of a small number of threads.

---

## 2. The key difference: blocking vs suspending

When a thread waits for something (a network call, a `sleep`), it **blocks**: it stays occupied doing nothing, but keeps holding its memory and its spot.

A coroutine, instead, **suspends**: when it hits a waiting point, it frees the thread so another coroutine can use it, and resumes when its result is ready. Suspending doesn't block the underlying thread.

> Blocking occupies the thread without working; suspending frees it to work on something else.

---

## 3. The practical impact (the numeric example)

If you wanted 100,000 concurrent operations with threads, you'd need 100,000 threads - impossible, you'd run out of memory long before that. With coroutines, those 100,000 can run on a pool of few threads, because most are suspended waiting on I/O and freeing their thread in the meantime.

```kotlin
// With threads: this blows up memory (~100 GB of stacks)
repeat(100_000) { thread { Thread.sleep(1000) } }

// With coroutines: runs fine on a few threads
repeat(100_000) { launch { delay(1000) } }
```

The difference is that `Thread.sleep` blocks a real thread, while `delay` suspends the coroutine and frees the thread.

---

## 4. Comparison table

| | Thread | Coroutine |
|---|---|---|
| Managed by | The operating system | The Kotlin runtime |
| Memory cost | ~1-2 MB (stack) | Bytes (an object) |
| How many fit | Thousands | Hundreds of thousands |
| While waiting | Blocks (occupies the thread) | Suspends (frees the thread) |
| Context switch | OS-level, expensive | Runtime-level, cheap |
| Cancellation | Manual, complicated | Cooperative, via scope |
| Relationship | Runs on the OS | Runs on threads |

---

## 5. What shows real experience

**Structured concurrency**: coroutines live in a scope and follow a hierarchy (see [06 - Scopes](06-concurrencia-scopes.en.md)). If the scope is cancelled, all its children are cancelled. With threads that has to be handled by hand and is prone to leaks.

**Cancellation is cooperative**: a coroutine is cancelled at suspension points, the OS doesn't kill it outright (see [08 - Cancellation](08-concurrencia-cancelacion.en.md)). That's why code has to be written respecting cancellation.

**The danger of mixing them**: if blocking code is called inside a coroutine (an old library, synchronous JDBC), the real thread gets blocked and the advantage is lost. That's why that work goes on `Dispatchers.IO`, which has headroom (up to 64 threads) to absorb blocking without touching the threads `Default` needs for CPU work.

---

## 6. The "faster" trap

A coroutine does **not** execute code faster than a thread; CPU work takes the same time. What you gain is **concurrency and efficient resource use**: you handle much more concurrent waiting with less memory and fewer context switches. It's "more efficient at concurrency," not "faster at computation" - telling these apart is exactly what separates a memorized answer from real understanding.

---

## Summary - quick decision table

| I need... | Option |
|---|---|
| Thousands of tasks waiting on I/O concurrently | Coroutines (cheap concurrency) |
| CPU work spread across cores | Coroutines with `Dispatchers.Default` (which in turn use threads) |
| Interacting with a legacy thread-based API | Direct threads, or wrap with `withContext(Dispatchers.IO)` |
| Structured, hierarchical cancellation | Coroutines (structured concurrency) |

---

## Interview phrases

- "A thread is managed by the OS; a coroutine is managed by the Kotlin runtime and runs on threads."
- "Blocking occupies the thread without working; suspending frees it."
- "It's not threads vs coroutines: coroutines are a layer on top that squeezes few threads."
- "delay suspends, Thread.sleep blocks - that's the difference in one line."
- "Coroutines aren't faster at computation, they're more efficient at concurrency."

## Common interview traps

**"Is a coroutine faster than a thread?"** - Not at executing code: CPU work takes the same time. You gain resource efficiency to handle a lot of waiting concurrency, not computation speed.

**"What happens if I call blocking code inside a coroutine on Dispatchers.Default?"** - It blocks a real thread from the Default pool, which is limited (= number of cores), and that can choke the CPU work of the whole app. That blocking code should go on Dispatchers.IO.

**"Do coroutines replace threads?"** - No, they run on top of them. The dispatcher decides which thread or pool each coroutine runs on.
