# State Management / Manejo de estado

[Versión en español](13-compose-estado.es.md)

---

## 1. State hoisting: the base pattern

**State hoisting** is Compose's central pattern for managing state: the state moves up to the parent component (or whoever calls the composable), and the child composable becomes stateless — it just receives the current value and a function to request a change.

```kotlin
// BEFORE: the composable owns its own state (hard to reuse or control from outside)
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}

// AFTER: state is "hoisted" to the caller, the composable is stateless
@Composable
fun Counter(count: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) {
        Text("Count: $count")
    }
}
```

The practical rule: **state goes down, events go up**. The value flows downward (parameter), the change event flows upward (lambda).

---

## 2. Stateless vs stateful

A **stateless** composable has no `remember`/`mutableStateOf` of its own for what it shows; everything arrives via parameters. A **stateful** composable does keep its own internal state.

The common Android pattern is a thin stateful layer (connected to the ViewModel) wrapping a stateless layer (the actual UI, easy to test and use in Previews):

```kotlin
// Stateful: connected to the ViewModel, lives in navigation
@Composable
fun ProfileScreen(viewModel: ProfileViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    ProfileContent(
        uiState = uiState,
        onSaveClick = viewModel::onSave
    )
}

// Stateless: just UI, easy to test and use in @Preview
@Composable
fun ProfileContent(uiState: ProfileUiState, onSaveClick: () -> Unit) {
    Column {
        Text(uiState.name)
        Button(onClick = onSaveClick) { Text("Save") }
    }
}
```

This separation is exactly what gets asked about when interviewers mention Compose "testability": `ProfileContent` can be tested or previewed by passing any `ProfileUiState` by hand, with no need for a real ViewModel.

---

## 3. Ways to manage state in Compose

| Tool | What it's for |
|---|---|
| `remember { mutableStateOf(...) }` | Local UI state that doesn't need to survive a configuration change (e.g. whether a dropdown is expanded) |
| `rememberSaveable { mutableStateOf(...) }` | Same, but survives rotation/configuration change (saved into the `Bundle`) |
| `ViewModel` + `StateFlow`/`State` | State representing business or screen data, survives configuration changes and (with `SavedStateHandle`) even process death |
| `derivedStateOf` | A value calculated from other state, to avoid unnecessary recalculation on every recomposition |
| `rememberCoroutineScope` | Not state itself, but the tool for launching coroutines from a callback inside a composable |

---

## 4. `remember` vs `rememberSaveable`

`remember` keeps the value while the composable stays in composition, but it's lost on a configuration change (screen rotation, language change) because that recreates the Activity.

`rememberSaveable` saves the value into the `Bundle` (the same mechanism as `onSaveInstanceState`), so it survives those changes. It only works with types that can be saved into a `Bundle` (primitives, `Parcelable`, or a custom `Saver` for complex types).

```kotlin
// Lost on screen rotation
var expanded by remember { mutableStateOf(false) }

// Survives screen rotation
var expanded by rememberSaveable { mutableStateOf(false) }
```

**Practical rule:** if losing the value on rotation would bother the user (text they were typing, a selected filter), use `rememberSaveable`. If it's purely transient and losing it doesn't matter, `remember` is enough and lighter.

---

## 5. Side effects: bridges between Compose and the non-declarative world

Compose is declarative: it describes what to show, not step-by-step instructions. But some things are inherently imperative (a network call on entering a screen, subscribing to a listener, showing a Snackbar). **Side effects** are the controlled way to do that inside a declarative world.

**`LaunchedEffect(key)`**: launches a coroutine tied to the composable's lifecycle. It relaunches if `key` changes, and cancels automatically if the composable leaves composition.

```kotlin
@Composable
fun UserProfile(userId: String, viewModel: ProfileViewModel) {
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)   // reruns if userId changes
    }
}
```

**`DisposableEffect(key)`**: for effects that need explicit cleanup (subscribing/unsubscribing a listener, registering/unregistering a broadcast receiver). Its `onDispose` block runs when the composable leaves composition or when `key` changes.

```kotlin
@Composable
fun LocationTracker(onLocationUpdate: (Location) -> Unit) {
    DisposableEffect(Unit) {
        val listener = LocationListener { onLocationUpdate(it) }
        locationManager.registerListener(listener)
        onDispose {
            locationManager.unregisterListener(listener)   // guaranteed cleanup
        }
    }
}
```

**`SideEffect`**: runs on every successful recomposition (not just on enter/exit). Used to sync Compose state toward something outside Compose (e.g. updating an analytics attribute with the current value).

```kotlin
@Composable
fun ScreenWithAnalytics(screenName: String) {
    SideEffect {
        analytics.setCurrentScreen(screenName)   // runs on every successful recomposition
    }
}
```

---

## 6. `rememberCoroutineScope`: launching coroutines outside a "pure" composable

A composable can't directly call a `suspend` function (its body isn't a coroutine). To launch a coroutine from a callback (e.g. `onClick`), you need a `CoroutineScope` tied to the composable's lifecycle — that's what `rememberCoroutineScope` gives you.

```kotlin
@Composable
fun SnackbarDemo(snackbarHostState: SnackbarHostState) {
    val scope = rememberCoroutineScope()

    Button(onClick = {
        scope.launch {
            snackbarHostState.showSnackbar("Saved")   // showSnackbar is suspend
        }
    }) {
        Text("Save")
    }
}
```

**Key difference from `LaunchedEffect`:** `LaunchedEffect` launches automatically on entering composition (or when its key changes); `rememberCoroutineScope` gives a scope to launch manually, inside a user event (click, swipe), not automatically.

---

## 7. `derivedStateOf`: avoiding unnecessary recalculation

When a value is calculated from other state, and that calculation is expensive or triggers extra recompositions, `derivedStateOf` ensures the derived value only recalculates when the base state actually changes in a relevant way (not on every frame).

```kotlin
@Composable
fun ScrollToTopButton(listState: LazyListState) {
    // without derivedStateOf, this would recalculate on every scroll pixel
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }

    if (showButton) {
        Button(onClick = { /* scroll to top */ }) { Text("Scroll up") }
    }
}
```

`listState.firstVisibleItemIndex` changes on every scroll pixel, but `showButton` only flips from `true` to `false` (or back) at one specific point. Without `derivedStateOf`, any reader of `showButton` would recompose on every pixel; with `derivedStateOf`, it only recomposes when the boolean value actually changes.

---

## 8. Complete example: idiomatic pattern

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    var query by rememberSaveable { mutableStateOf("") }   // survives rotation
    val results by viewModel.results.collectAsStateWithLifecycle()
    val scope = rememberCoroutineScope()
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(query) {
        if (query.isNotBlank()) {
            delay(300)   // debounce
            viewModel.search(query)
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) {
        Column {
            TextField(value = query, onValueChange = { query = it })
            LazyColumn {
                items(results) { result ->
                    ResultItem(result, onClick = {
                        scope.launch { snackbarHostState.showSnackbar("Selected: ${result.title}") }
                    })
                }
            }
        }
    }
}
```

This example combines `rememberSaveable` (the search text survives rotation), `LaunchedEffect` (triggers the search with debounce when the query changes), `collectAsStateWithLifecycle` (observes the ViewModel's StateFlow), and `rememberCoroutineScope` (launches the Snackbar from a click).

---

## Summary - quick decision table

| I need... | Tool |
|---|---|
| Local state where losing it on rotation doesn't matter | `remember` |
| Local state that must survive rotation | `rememberSaveable` |
| Business/screen state, survives config change | `ViewModel` + `StateFlow` |
| To run something async on enter / when a key changes | `LaunchedEffect` |
| To run something with guaranteed cleanup on exit | `DisposableEffect` |
| To sync toward something non-Compose on every recomposition | `SideEffect` |
| To launch a coroutine from a callback (onClick) | `rememberCoroutineScope` |
| To avoid recalculating a derived value | `derivedStateOf` |

---

## Interview phrases

- "State hoisting moves state up to the parent: state flows down, events flow up."
- "remember is lost on rotation; rememberSaveable saves it into the Bundle, same mechanism as onSaveInstanceState."
- "LaunchedEffect launches a coroutine automatically; rememberCoroutineScope gives a scope to launch manually from a user event."
- "derivedStateOf prevents a derived value from triggering recompositions every frame when the underlying value changes more often than the derived one does."

## Common interview traps

**"What's the difference between a stateless and a stateful composable?"** - Stateless has no state of its own, everything arrives via parameters (easy to test/preview); stateful keeps its own `remember`/`mutableStateOf`. The common pattern is wrapping a stateless layer with a thin stateful layer connected to the ViewModel.

**"Why can't I call a suspend function directly inside a composable?"** - Because a composable's body isn't a coroutine; it needs an explicit `CoroutineScope` (`rememberCoroutineScope` for callbacks, `LaunchedEffect` to launch automatically).

**"What happens if I use remember instead of rememberSaveable for a search field's text?"** - It's lost on rotation because the Activity gets recreated; rememberSaveable persists it in the state Bundle.
