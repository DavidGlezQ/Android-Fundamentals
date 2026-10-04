# Flows

[Versión en español](10-flows.es.md)

---

## 1. Cold vs Hot: the root distinction

A **cold Flow** (`flow { }`) does nothing until someone collects it. Each collector triggers a new, independent execution of the block - if three screens collect the same cold flow, the block runs three times.

```kotlin
val cold = flow {
    println("starting")   // printed once PER collector
    emit(1); emit(2)
}
```

A **hot Flow** emits whether or not there's a collector, and every collector shares the same emission. `StateFlow` and `SharedFlow` are hot.

Analogy: cold is a song on streaming (everyone plays it from the start when they hit play); hot is live radio (if you tune in late, you missed what came before).

---

## 2. Cold Flow: on-demand streams

Used when the source produces data on demand and you want operators on top of it: a Room query, reading DataStore, transforming with `map`/`filter`/`combine`. It's cold because execution only starts when the UI observes it, not before.

```kotlin
fun observeUser(id: String): Flow<User> = flow {
    emit(api.getUser(id))
}
```

---

## 3. StateFlow: for UI state

A hot flow that **always has a current value**, requires an initial value, and does **conflation**: if the value doesn't change by `equals`, it doesn't re-emit.

```kotlin
private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

fun load() {
    viewModelScope.launch {
        _uiState.value = UiState.Success(repo.getUser())
    }
}
```

Key points:
- **Always has a value** - needs an initial value. Perfect for UI state.
- **Conflated by design**: emitting the same value twice in a row is only seen once; with fast emissions, a slow collector can miss intermediate values (only guarantees the most recent).
- **`.value` is synchronous** - you can read the current state without collecting.
- Not suited for **events**: a new collector (e.g. after a screen rotation) receives the last value again and would re-process it.

---

## 4. SharedFlow: for events

More configurable than StateFlow. Doesn't require an initial value and is used for **events** (navigation, snackbars) where StateFlow's conflation would eat repeated events.

```kotlin
private val _events = MutableSharedFlow<Event>(
    replay = 0,
    extraBufferCapacity = 1,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
val events: SharedFlow<Event> = _events.asSharedFlow()
```

Three parameters:
- **`replay`**: how many past emissions a late collector receives. Defaults to 0 (no arguments). `StateFlow` is essentially a `SharedFlow` with `replay = 1` + conflation + a mandatory initial value.
- **`extraBufferCapacity`**: space so `emit` doesn't suspend waiting for slow collectors.
- **`onBufferOverflow`**: what to do when the buffer fills - `SUSPEND` (default), `DROP_OLDEST`, `DROP_LATEST`.

---

## 5. `stateIn` / `shareIn`: turning cold into hot

Take a cold flow (e.g. from Room) and share it as hot, tied to a scope.

```kotlin
val user: StateFlow<User> = repo.observeUser()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = User.Empty
    )
```

**Why it exists:** manually collecting a cold flow into a `MutableStateFlow` forces you to reimplement lifecycle handling (when to start/stop based on whether there are observers) and sharing a single collection across several collectors. `stateIn` gives you that for free via `SharingStarted`.

**The `SharingStarted` options:**
- `Eagerly` - starts immediately and never stops.
- `Lazily` - starts with the first collector and never stops.
- `WhileSubscribed(timeoutMillis)` - starts with the first collector, stops when the last one leaves with a grace timeout. The classic `WhileSubscribed(5000)` avoids restarting the upstream (e.g. re-querying Room) during a screen rotation, which takes less than 5s.

**When to use `stateIn` vs a manual `MutableStateFlow`:**

| Source | Tool |
|---|---|
| Existing cold flow (Room, DataStore, callbackFlow) | `stateIn` |
| State pushed from one-off actions/calls | `MutableStateFlow` |

Example: a one-shot network call doesn't need `stateIn`, a regular `MutableStateFlow` is correct. A reactive Room query does need it.

---

## 6. Channel: for single-consumer delivery

A `Channel` is a queue: each emitted value is received by **exactly one** consumer, and the value is kept until someone picks it up. It contrasts with `SharedFlow`, which broadcasts to all active collectors.

```kotlin
private val _events = Channel<Event>(Channel.BUFFERED)
val events = _events.receiveAsFlow()

fun onLoginSuccess() {
    viewModelScope.launch {
        _events.send(NavigateToHome)   // kept until someone reads it
    }
}
```

### Channel vs SharedFlow for events

| | SharedFlow | Channel |
|---|---|---|
| Model | Broadcast (all collectors) | Queue (one consumer per value) |
| If no collector when emitting | Lost (with replay=0) | Kept until consumed |
| Multiple collectors | All receive the value | Values are split among them |

With `SharedFlow(replay=0)`, if you emit an event and at that instant there's no collector (e.g. screen rotating), the event is lost. With `Channel`, the event stays in the queue until the new collector shows up. That's why the community leans toward `Channel` for UI events with single-delivery guarantees.

**Fan-out:** when several consumers compete to pick up values from the same `Channel` (splitting work among workers), that's intentional and exactly what a `Channel` with multiple collectors is for - each task is picked up by just one. For UI events, on the other hand, you want a single collector; two collectors there would split the events between them, each losing half.

---

## 7. `flowOn`: changing the upstream's dispatcher

Changes the dispatcher where everything **above** `flowOn` in the chain runs - the `flow { }` block and the previous operators.

```kotlin
flow {
    emit(readFromDatabase())    // runs on IO
}
.map { transform(it) }          // also on IO (it's above flowOn)
.flowOn(Dispatchers.IO)         // affects everything ABOVE
.collect { render(it) }         // runs in the collector's context (e.g. Main)
```

**The key point:** `flowOn` only affects upward, not downward. Everything after it, including the `collect`, stays in the collector's own context - this respects "context preservation." It's the equivalent of `withContext` but for flows, because inside a `flow { }` you can't use `withContext` directly to change the emission context.

---

## 8. Backpressure: when the producer is faster than the consumer

By default, a flow is sequential: the producer **waits** for the collector to finish processing each value before emitting the next. There are three operators to handle a speed mismatch:

**`buffer()`**: the producer doesn't wait, it queues values. Producer and consumer run concurrently.

```kotlin
flow { /* emits fast */ }
    .buffer()
    .collect { slowProcess(it) }
```

**`conflate()`**: if the collector is busy, it drops intermediate values and keeps the most recent one available. For when only the latest value matters (e.g. a slider's position).

```kotlin
flow { /* emits fast */ }
    .conflate()
    .collect { slowProcess(it) }
```

**`collectLatest { }`**: cancels the in-progress processing if a new value arrives, and restarts with that value. Ideal when a new value makes the previous work irrelevant (e.g. search while typing: the user types another letter, the previous search is cancelled).

```kotlin
flow { /* emits */ }
    .collectLatest { value ->
        slowProcess(value)   // if a new value arrives, CANCELS this one and starts over
    }
```

**The fine distinction `conflate` vs `collectLatest`:** `conflate` lets the current processing **finish** and then jumps to the latest available value, without interrupting. `collectLatest` **actively cancels** the in-progress processing as soon as a new value arrives.

| Operator | What it does | When to use it |
|---|---|---|
| (none) | Producer waits for the consumer | Default, coupled processing |
| `buffer` | Queues, producer doesn't wait | Process everything, don't lose values |
| `conflate` | Drops intermediates, keeps the latest | Only the most recent value matters |
| `collectLatest` | Cancels the previous one when a new one arrives | The new value invalidates previous work |

**Connection with `flowOn`:** `flowOn` internally introduces a buffer when crossing dispatchers, because producer and consumer end up on different threads.

---

## 9. Combination operators: `combine`, `zip`, `merge`

**`combine`**: combines the **latest** emissions from several flows every time any of them emits.

```kotlin
combine(userFlow, settingsFlow) { user, settings ->
    UiState(user, settings)
}
```

**`zip`**: pairs emissions by **position** - waits until both flows have a corresponding new value before combining.

```kotlin
flowA.zip(flowB) { a, b -> a + b }
```

**`merge`**: joins the emissions from several flows into one, without combining them - each emission passes through as-is, from whichever source flow.

```kotlin
merge(flowA, flowB).collect { println(it) }
```

The practical difference: `combine` reacts to any change using the most recent values from all of them; `zip` pairs them one by one in order; `merge` just interleaves without combining anything.

---

## 10. Real Android example: lifecycle in Compose/Views

```kotlin
// ViewModel: Room as a reactive source
val uiState: StateFlow<UiState> = repo.observeUser()
    .map { UiState.Success(it) }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), UiState.Loading)
```

```kotlin
// Compose: respects the lifecycle automatically
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
}
```

```kotlin
// View system: the manual equivalent
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state -> render(state) }
    }
}
```

`collectAsStateWithLifecycle()` uses `repeatOnLifecycle(STARTED)` underneath - it stops the collect when the app goes to background. Together with `WhileSubscribed(5000)` in the ViewModel, the pair achieves that the upstream (e.g. Room) shuts off when there's no UI observing, and reactivates on return, without restarting on every screen rotation.

**The common trap:** `collectAsState()` (without `WithLifecycle`) comes from plain Compose and doesn't respect Android's lifecycle - it keeps collecting even in background. For Android, `collectAsStateWithLifecycle()` is always the right call.

---

## Summary - quick decision table

| I need... | Tool |
|---|---|
| On-demand stream with operators | Cold Flow (`flow { }`) |
| UI state I push myself (one-shot) | `MutableStateFlow` |
| UI state from a cold flow (Room, DataStore) | `stateIn` |
| One-shot events (navigate, snackbar) with delivery guarantee | `Channel` + `receiveAsFlow()` |
| Events several observers should receive at once | `SharedFlow` |
| Sharing a cold flow across collectors without re-running it | `stateIn` / `shareIn` |
| Changing a flow's dispatcher | `flowOn` (only affects upstream) |
| Producer faster than consumer, no values lost | `buffer` |
| Only the most recent value matters | `conflate` |
| A new value invalidates previous processing | `collectLatest` |
| Combining the latest values of several flows | `combine` |
| Pairing by position | `zip` |
| Interleaving without combining | `merge` |
| Collecting in Compose respecting the lifecycle | `collectAsStateWithLifecycle()` |

---

## Interview phrases

- "Cold flow starts with each collector; hot emits whether or not there's a collector and everyone shares the emission."
- "StateFlow is a SharedFlow with replay 1, conflation, and a mandatory initial value."
- "SharedFlow is broadcast to all collectors; Channel splits each value to just one."
- "flowOn only affects the upstream; the collect stays in the collector's context, due to context preservation."
- "conflate lets it finish and jumps to the latest; collectLatest cancels the current one and restarts."
- "WhileSubscribed(5000) avoids restarting the upstream on every screen rotation."

## Common interview traps

**"Why not use StateFlow for a navigation event?"** - Because on re-subscription (e.g. after rotating) a new collector receives the last value again and would navigate again. Events go in a Channel or a SharedFlow with replay=0.

**"Are conflate and collectLatest the same thing?"** - No: conflate lets the current processing finish and jumps to the latest available value; collectLatest actively cancels the in-progress processing when a new one arrives.

**"If I put flowOn(IO) at the end of the chain, does the collect run on IO?"** - No, flowOn only affects upward; the collect runs in the collector's own context (e.g. Main if launched from viewModelScope).

**"Why is collecting with collectAsState() instead of collectAsStateWithLifecycle() a problem in Android?"** - Because collectAsState() doesn't respect Android's lifecycle and keeps collecting even while the app is in background, wasting resources.
