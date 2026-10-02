# Scopes & Structured Concurrency / Scopes y Structured Concurrency

[Versión en español](06-concurrencia-scopes.md)

---

## 1. The problem it solves

Coroutines with no structure: you launch one, forget about it, it keeps running forever even if you no longer need it. The user closes the screen but the coroutine keeps making a network call and then tries to update a UI that no longer exists → crash or memory leak.

**Structured concurrency** is the rule that **every coroutine lives inside a scope**, and that scope forms a **parent-child hierarchy** with clear rules: if the parent is cancelled, all children are cancelled; the parent doesn't finish until all its children finish. Nothing is left orphaned.

---

## 2. CoroutineScope: the container

A `CoroutineScope` is the context where coroutines live. Every builder (`launch`, `async`) is an **extension function of `CoroutineScope`** — that's why you need a scope to call them.

```kotlin
class MyViewModel : ViewModel() {
    fun doWork() {
        viewModelScope.launch {   // viewModelScope IS a CoroutineScope
            // I live inside the ViewModel's scope
        }
    }
}
```

Scopes you already use in Android:
- **`viewModelScope`** → lives as long as the ViewModel; cancels only in `onCleared`.
- **`lifecycleScope`** → tied to an Activity/Fragment's lifecycle.

When the ViewModel dies, `viewModelScope` **automatically cancels** every coroutine launched in it. That's why there are no leaks: no coroutine outlives its screen.

---

## 3. The parent-child hierarchy

When you launch a coroutine inside a scope, a parent-child relationship is created. A **tree of Jobs** forms.

The three golden rules:

1. **The parent waits for all its children.** It's not considered finished until all its children are finished.
2. **Cancelling the parent cancels all children.**
3. **If a child fails, it cancels the parent and siblings** (with a regular `Job` — with `SupervisorJob` this changes, see section 6).

```kotlin
coroutineScope {                        // parent
    launch { taskA() }                  // child 1
    launch { taskB() }                  // child 2
    // this block does NOT finish until taskA AND taskB finish (rule 1)
}
```

---

## 4. `coroutineScope { }`: grouping and waiting

A suspend builder that creates a child scope and **suspends until everything inside finishes**. It says "do all these things concurrently and don't continue until all of them finish."

```kotlin
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val user = async { api.getUser() }
    val posts = async { api.getPosts() }
    Dashboard(user.await(), posts.await())
    // coroutineScope doesn't return until both async finish
}
```

Why use it instead of launching loose coroutines? Because it **groups them**: if `getUser()` fails, `coroutineScope` automatically cancels `getPosts()` and propagates the exception. The whole group is treated as a unit.

---

## 5. Real case: three sibling `launch` calls vs `coroutineScope`

```kotlin
// THREE launch calls in viewModelScope: they're SIBLINGS under the same parent
viewModelScope.launch { repo.getBalance() }
viewModelScope.launch { repo.getProfile() }
viewModelScope.launch { repo.getTransactions() }
```

With a regular `viewModelScope` (uses `SupervisorJob` inside), if one fails, **the others continue**. But if you did the same inside a regular `coroutineScope { }`, a failure **would** cancel the siblings. That difference is `Job` vs `SupervisorJob`.

---

## 6. `Job` vs `SupervisorJob`: failure propagation

**Regular `Job` (`coroutineScope`):** a child's failure propagates upward and cancels everyone. "If one falls, they all fall."

```kotlin
coroutineScope {
    launch { throw Exception("boom") }  // fails
    launch { delay(1000); print("never gets here") } // ← cancelled by the sibling
}
```

**`SupervisorJob` (`supervisorScope`):** a child's failure does **not** affect its siblings. "If one falls, the rest continue."

```kotlin
supervisorScope {
    launch { throw Exception("boom") }  // fails
    launch { delay(1000); print("does get here") } // ← survives
}
```

When to use each:
- **`coroutineScope`** → tasks that are **a whole**: if one fails, the others don't make sense (e.g. you need all three to build a single screen).
- **`supervisorScope`** → **independent** tasks: one failing doesn't invalidate the others (e.g. a screen with separate sections).

`viewModelScope` uses `SupervisorJob` inside, that's why several sibling `launch` calls don't bring each other down.

---

## 7. Real Android example

**"All or nothing" case — coroutineScope:**

```kotlin
fun loadProfile() {
    viewModelScope.launch {
        _state.value = Loading
        try {
            val screen = coroutineScope {   // group: if one fails, cancel the other
                val user = async { repo.getUser() }
                val settings = async { repo.getSettings() }
                ProfileScreen(user.await(), settings.await())
            }
            _state.value = Success(screen)   // only if BOTH succeeded
        } catch (e: Exception) {
            _state.value = Error(e)          // if either failed, a single error
        }
    }
}
```

**"Independent" case — each section its own state (e.g. bank screen):**

```kotlin
fun loadAccount() {
    // Each sibling launch in viewModelScope (SupervisorJob): independent
    viewModelScope.launch {
        _balance.value = runCatching { repo.getBalance() }
            .fold({ Balance.Success(it) }, { Balance.Error(it) })
    }
    viewModelScope.launch {
        _profile.value = runCatching { repo.getProfile() }
            .fold({ Profile.Success(it) }, { Profile.Error(it) })
    }
    viewModelScope.launch {
        _txns.value = runCatching { repo.getTransactions() }
            .fold({ Txns.Success(it) }, { Txns.Error(it) })
    }
}
```

---

## 8. Common mistakes

**a) Using `GlobalScope`.** Creates a coroutine that **doesn't belong to any structured scope** — lives forever, doesn't cancel with the screen, leaks memory. Antipattern #1.

```kotlin
// BAD: outlives the screen, guaranteed leak
GlobalScope.launch { repo.getData() }
// GOOD: tied to the lifecycle
viewModelScope.launch { repo.getData() }
```

**b) Creating a custom scope and not cancelling it.** If you create your own `CoroutineScope(...)`, you're responsible for cancelling it. In Android, prefer the provided scopes.

**c) Expecting a `coroutineScope` to survive a failure.** If a task can fail without invalidating the others, `coroutineScope` will cancel everything — you need `supervisorScope`.

**d) Launching in the wrong scope.** Long-lived work in `lifecycleScope` (dies on rotation) when it should be in `viewModelScope` (survives).

---

## 9. Interview phrases

- *"Every coroutine lives in a scope — there are no orphans."*
- *"The parent waits for its children; cancelling the parent cancels the whole tree."*
- *"coroutineScope: if one fails, they all fall. supervisorScope: each lives its own life."*
- *"GlobalScope is the antipattern: doesn't cancel with the screen, leaks memory."*
- *"viewModelScope only cancels in onCleared — that's why there are no leaks."*

---

## 10. Typical interview traps

**"What is structured concurrency?"**
Every coroutine lives in a scope and forms a parent-child hierarchy: the parent doesn't finish until its children finish, cancelling the parent cancels the children, and a failure propagates according to the type of Job. The benefit: no orphan coroutines.

**"Difference between coroutineScope and supervisorScope?"**
Both create a child scope, but differ in failure propagation. `coroutineScope`: if a child fails, everyone is cancelled — "all or nothing." `supervisorScope`: a child's failure doesn't affect siblings.

**"Three independent calls in a ViewModel, one fails, do the others continue?"**
Depends on the scope. With three `launch` calls in `viewModelScope` (SupervisorJob), yes they continue. Inside a `coroutineScope { }`, no — the failure cancels the siblings.
