# State Management / Manejo de estado

[English version](13-compose-estado.en.md)

---

## 1. State hoisting: el patrón base

**State hoisting** es el patrón central de Compose para manejar estado: el estado se mueve hacia arriba, al componente padre (o a quien llama el composable), y el composable hijo se queda sin estado propio, solo recibe el valor actual y una función para pedir que cambie.

```kotlin
// ANTES: el composable es dueño de su propio estado (dificil de reusar o controlar desde afuera)
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}

// DESPUES: el estado se "eleva" al llamador, el composable es stateless
@Composable
fun Counter(count: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) {
        Text("Count: $count")
    }
}
```

La regla practica: **state goes down, events go up**. El valor fluye hacia abajo (parámetro), el evento de cambio fluye hacia arriba (lambda).

---

## 2. Stateless vs stateful

Un composable **stateless** no tiene ningún `remember`/`mutableStateOf` propio para lo que muestra; todo le llega por parámetros. Un composable **stateful** sí mantiene su propio estado interno.

El patrón común en Android es tener una capa stateful "delgada" (conectada a ViewModel) que envuelve una capa stateless (la UI real, fácil de testear y de usar en Previews):

```kotlin
// Stateful: conectado al ViewModel, vive en la navegación
@Composable
fun ProfileScreen(viewModel: ProfileViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    ProfileContent(
        uiState = uiState,
        onSaveClick = viewModel::onSave
    )
}

// Stateless: solo UI, fácil de probar y de usar en @Preview
@Composable
fun ProfileContent(uiState: ProfileUiState, onSaveClick: () -> Unit) {
    Column {
        Text(uiState.name)
        Button(onClick = onSaveClick) { Text("Guardar") }
    }
}
```

Esta separación es justo lo que se suele preguntar en entrevista cuando mencionan "testability" de Compose: `ProfileContent` se puede testear o previsualizar pasando cualquier `ProfileUiState` a mano, sin necesitar un ViewModel real.

---

## 3. Las formas de manejar estado en Compose

| Herramienta | Para qué sirve |
|---|---|
| `remember { mutableStateOf(...) }` | Estado local de UI que no necesita sobrevivir cambios de configuración (ej. si un dropdown está expandido) |
| `rememberSaveable { mutableStateOf(...) }` | Igual, pero sobrevive rotación/cambio de configuración (se guarda en el `Bundle`) |
| `ViewModel` + `StateFlow`/`State` | Estado que representa datos de negocio o de pantalla, sobrevive cambios de configuración y (con `SavedStateHandle`) hasta process death |
| `derivedStateOf` | Un valor calculado a partir de otro estado, para evitar recálculos innecesarios en cada recomposición |
| `rememberCoroutineScope` | No es estado en sí, pero es la herramienta para lanzar corrutinas desde un callback dentro de un composable |

---

## 4. `remember` vs `rememberSaveable`

`remember` guarda el valor mientras el composable permanece en composición, pero se pierde si hay un cambio de configuración (rotación de pantalla, cambio de idioma) porque eso recrea la Activity.

`rememberSaveable` guarda el valor en el `Bundle` (el mismo mecanismo de `onSaveInstanceState`), así que sobrevive esos cambios. Solo funciona con tipos que se puedan guardar en un `Bundle` (primitivos, `Parcelable`, o con un `Saver` custom para tipos complejos).

```kotlin
// Se pierde al rotar la pantalla
var expanded by remember { mutableStateOf(false) }

// Sobrevive al rotar la pantalla
var expanded by rememberSaveable { mutableStateOf(false) }
```

**Regla práctica:** si perder el valor al rotar molestaría al usuario (un texto que estaba escribiendo, un filtro seleccionado), usa `rememberSaveable`. Si es puramente transitorio y no importa perderlo, `remember` es suficiente y más liviano.

---

## 5. Side effects: puentes entre Compose y el mundo no declarativo

Compose es declarativo: describe qué mostrar, no instrucciones paso a paso. Pero hay cosas que sí son inherentemente imperativas (una llamada de red al entrar a la pantalla, suscribirse a un listener, mostrar un Snackbar). Los **side effects** son la forma controlada de hacer eso dentro de un mundo declarativo.

**`LaunchedEffect(key)`**: lanza una corrutina atada al ciclo de vida del composable. Se relanza si `key` cambia, y se cancela automáticamente si el composable sale de composición.

```kotlin
@Composable
fun UserProfile(userId: String, viewModel: ProfileViewModel) {
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)   // se vuelve a ejecutar si userId cambia
    }
}
```

**`DisposableEffect(key)`**: para efectos que necesitan limpieza explícita (suscribir/desuscribir un listener, registrar/desregistrar un broadcast receiver). Su bloque `onDispose` corre cuando el composable sale de composición o cuando `key` cambia.

```kotlin
@Composable
fun LocationTracker(onLocationUpdate: (Location) -> Unit) {
    DisposableEffect(Unit) {
        val listener = LocationListener { onLocationUpdate(it) }
        locationManager.registerListener(listener)
        onDispose {
            locationManager.unregisterListener(listener)   // limpieza garantizada
        }
    }
}
```

**`SideEffect`**: corre en cada recomposición exitosa (no solo al entrar/salir). Se usa para sincronizar estado de Compose hacia algo que no es Compose (ej. actualizar un atributo de analytics con el valor actual).

```kotlin
@Composable
fun ScreenWithAnalytics(screenName: String) {
    SideEffect {
        analytics.setCurrentScreen(screenName)   // corre en cada recomposición exitosa
    }
}
```

---

## 6. `rememberCoroutineScope`: lanzar corrutinas fuera de un composable "puro"

Un composable no puede llamar directamente a una función `suspend` (su cuerpo no es una corrutina). Para lanzar una corrutina desde un callback (ej. `onClick`), se necesita un `CoroutineScope` atado al ciclo de vida del composable: eso es lo que da `rememberCoroutineScope`.

```kotlin
@Composable
fun SnackbarDemo(snackbarHostState: SnackbarHostState) {
    val scope = rememberCoroutineScope()

    Button(onClick = {
        scope.launch {
            snackbarHostState.showSnackbar("Guardado")   // showSnackbar es suspend
        }
    }) {
        Text("Guardar")
    }
}
```

**Diferencia clave con `LaunchedEffect`:** `LaunchedEffect` lanza automáticamente al entrar en composición (o cuando cambia su key); `rememberCoroutineScope` da un scope para lanzar manualmente, dentro de un evento del usuario (click, swipe), no automáticamente.

---

## 7. `derivedStateOf`: evitar recálculo innecesario

Cuando un valor se calcula a partir de otro estado, y ese cálculo es costoso o dispara recomposiciones de más, `derivedStateOf` asegura que el valor derivado solo se recalcule cuando el estado base realmente cambia de forma relevante (no en cada frame).

```kotlin
@Composable
fun ScrollToTopButton(listState: LazyListState) {
    // sin derivedStateOf, esto se recalcularía en cada scroll pixel a pixel
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }

    if (showButton) {
        Button(onClick = { /* scroll to top */ }) { Text("Subir") }
    }
}
```

`listState.firstVisibleItemIndex` cambia en cada pixel de scroll, pero `showButton` solo cambia de `true` a `false` (o viceversa) en un punto puntual. Sin `derivedStateOf`, cualquier lector de `showButton` se recompondría en cada pixel; con `derivedStateOf`, solo se recompone cuando el valor booleano realmente cambia.

---

## 8. Ejemplo completo: patrón idiomático

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    var query by rememberSaveable { mutableStateOf("") }   // sobrevive rotación
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
                        scope.launch { snackbarHostState.showSnackbar("Seleccionado: ${result.title}") }
                    })
                }
            }
        }
    }
}
```

Este ejemplo combina `rememberSaveable` (el texto de búsqueda sobrevive rotación), `LaunchedEffect` (dispara la búsqueda con debounce cuando cambia el query), `collectAsStateWithLifecycle` (observa el StateFlow del ViewModel) y `rememberCoroutineScope` (lanza el Snackbar desde un click).

---

## Resumen - tabla de decisión rápida

| Necesito... | Herramienta |
|---|---|
| Estado local que no importa perder al rotar | `remember` |
| Estado local que sí debe sobrevivir rotación | `rememberSaveable` |
| Estado de negocio/pantalla, sobrevive config change | `ViewModel` + `StateFlow` |
| Ejecutar algo async al entrar/cuando cambia una key | `LaunchedEffect` |
| Ejecutar algo con limpieza garantizada al salir | `DisposableEffect` |
| Sincronizar hacia algo no-Compose en cada recomposición | `SideEffect` |
| Lanzar una corrutina desde un callback (onClick) | `rememberCoroutineScope` |
| Evitar recálculos de un valor derivado | `derivedStateOf` |

---

## Frases para entrevista

- "State hoisting mueve el estado hacia el padre: el estado baja, los eventos suben."
- "remember se pierde al rotar; rememberSaveable lo guarda en el Bundle, igual que onSaveInstanceState."
- "LaunchedEffect lanza una corrutina automáticamente; rememberCoroutineScope da un scope para lanzar manualmente desde un evento del usuario."
- "derivedStateOf evita que un valor derivado dispare recomposiciones en cada frame cuando el valor real cambia con menos frecuencia."

## Trampas típicas de entrevista

**"¿Cuál es la diferencia entre un composable stateless y uno stateful?"** - Stateless no tiene estado propio, todo le llega por parámetros (fácil de testear/previsualizar); stateful mantiene su propio `remember`/`mutableStateOf`. El patrón común es envolver un stateless con una capa stateful delgada conectada al ViewModel.

**"¿Por qué no puedo llamar una función suspend directamente dentro de un composable?"** - Porque el cuerpo de un composable no es una corrutina; necesita un `CoroutineScope` explícito (`rememberCoroutineScope` para callbacks, `LaunchedEffect` para lanzar automáticamente).

**"¿Qué pasa si uso remember en vez de rememberSaveable para el texto de un campo de búsqueda?"** - Se pierde al rotar la pantalla porque la Activity se recrea; rememberSaveable lo persiste en el Bundle de estado.
