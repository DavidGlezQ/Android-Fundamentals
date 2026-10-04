# Flows

[English version](10-flows.en.md)

---

## 1. Cold vs Hot: la distincion raiz

Un **Flow frio** (`flow { }`) no hace nada hasta que alguien lo colecta. Cada colector dispara una ejecucion nueva e independiente del bloque - si tres pantallas colectan el mismo flow frio, el bloque corre tres veces.

```kotlin
val cold = flow {
    println("empiezo")   // se imprime una vez POR cada colector
    emit(1); emit(2)
}
```

Un **Flow caliente** emite exista o no un colector, y todos los colectores comparten la misma emision. `StateFlow` y `SharedFlow` son calientes.

La analogia: el frio es una cancion en streaming (cada quien la reproduce desde el inicio al darle play); el caliente es una radio en vivo (si se sintoniza tarde, se perdio lo anterior).

---

## 2. Cold Flow: streams bajo demanda

Se usa cuando la fuente produce datos bajo demanda y se quieren operadores encima: una query de Room, leer DataStore, transformar con `map`/`filter`/`combine`. Es frio porque la ejecucion arranca solo cuando la UI lo observa, no antes.

```kotlin
fun observeUser(id: String): Flow<User> = flow {
    emit(api.getUser(id))
}
```

---

## 3. StateFlow: para estado de UI

Un caliente que **siempre tiene un valor actual**, requiere valor inicial, y hace **conflation**: si el valor no cambia por `equals`, no re-emite.

```kotlin
private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

fun load() {
    viewModelScope.launch {
        _uiState.value = UiState.Success(repo.getUser())
    }
}
```

Puntos clave:
- **Siempre tiene valor** - necesita un valor inicial. Perfecto para estado de UI.
- **Conflated por diseno**: emitir el mismo valor dos veces seguidas solo se ve una vez; con emisiones rapidas, un colector lento puede perderse valores intermedios (solo garantiza el mas reciente).
- **`.value` es sincrono** - se puede leer el estado actual sin colectar.
- No sirve para **eventos**: un colector nuevo (tras rotar pantalla, por ejemplo) recibe el ultimo valor otra vez y lo re-procesaria.

---

## 4. SharedFlow: para eventos

Mas configurable que StateFlow. No requiere valor inicial y sirve para **eventos** (navegacion, snackbars) donde la conflation de StateFlow se comeria eventos repetidos.

```kotlin
private val _events = MutableSharedFlow<Event>(
    replay = 0,
    extraBufferCapacity = 1,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
val events: SharedFlow<Event> = _events.asSharedFlow()
```

Tres parametros:
- **`replay`**: cuantas emisiones pasadas recibe un colector tardio. Por defecto es 0 (sin argumentos). `StateFlow` es esencialmente un `SharedFlow` con `replay = 1` + conflation + valor inicial obligatorio.
- **`extraBufferCapacity`**: espacio para que `emit` no suspenda esperando colectores lentos.
- **`onBufferOverflow`**: que hacer cuando el buffer se llena - `SUSPEND` (default), `DROP_OLDEST`, `DROP_LATEST`.

---

## 5. `stateIn` / `shareIn`: convertir frio en caliente

Toman un flow frio (ej. de Room) y lo comparten como caliente, vinculado a un scope.

```kotlin
val user: StateFlow<User> = repo.observeUser()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = User.Empty
    )
```

**Por que existe:** colectar un flow frio a mano hacia un `MutableStateFlow` obliga a reimplementar el manejo de lifecycle (cuando empezar/parar segun haya observadores) y el compartir una sola coleccion entre varios colectores. `stateIn` lo da gratis con `SharingStarted`.

**Las opciones de `SharingStarted`:**
- `Eagerly` - arranca de inmediato y nunca para.
- `Lazily` - arranca con el primer colector y nunca para.
- `WhileSubscribed(timeoutMillis)` - arranca con el primer colector, para cuando se va el ultimo con un timeout de gracia. El clasico `WhileSubscribed(5000)` evita reiniciar el upstream (ej. re-consultar Room) durante una rotacion de pantalla, que dura menos de 5s.

**Cuando usar `stateIn`/`MutableStateFlow` a mano:**

| Fuente | Herramienta |
|---|---|
| Flow frio existente (Room, DataStore, callbackFlow) | `stateIn` |
| Se empuja el estado desde acciones/llamadas puntuales (one-shot) | `MutableStateFlow` |

Ejemplo: una llamada de red one-shot no necesita `stateIn`, un `MutableStateFlow` normal es correcto. Una query reactiva de Room si lo necesita.

---

## 6. Channel: para reparto de un solo consumidor

Un `Channel` es una cola: cada valor emitido lo recibe **exactamente un** consumidor, y el valor se guarda hasta que alguien lo toma. Contrasta con `SharedFlow`, que hace broadcast a todos los colectores activos.

```kotlin
private val _events = Channel<Event>(Channel.BUFFERED)
val events = _events.receiveAsFlow()

fun onLoginSuccess() {
    viewModelScope.launch {
        _events.send(NavigateToHome)   // se guarda hasta que alguien lo lea
    }
}
```

### Channel vs SharedFlow para eventos

| | SharedFlow | Channel |
|---|---|---|
| Modelo | Broadcast (todos los colectores) | Cola (un consumidor por valor) |
| Si no hay colector al emitir | Se pierde (con replay=0) | Se guarda hasta consumir |
| Multiples colectores | Todos reciben el valor | Se reparten los valores |

Con `SharedFlow(replay=0)`, si se emite un evento y en ese instante no hay colector (ej. pantalla rotando), el evento se pierde. Con `Channel`, el evento queda en la cola hasta que el colector nuevo aparece. Por eso la comunidad se inclina por `Channel` para eventos de UI con garantia de entrega unica.

**Fan-out:** cuando varios consumidores compiten por tomar valores de un mismo `Channel` (reparto de trabajo entre workers), eso es intencional y es exactamente para lo que sirve un `Channel` con multiples colectores - cada tarea la toma uno solo. Para eventos de UI, en cambio, se quiere un solo colector; dos colectores ahi repartirian los eventos entre si, perdiendo la mitad cada uno.

---

## 7. `flowOn`: cambiar el dispatcher del upstream

Cambia el dispatcher donde se ejecuta todo lo que esta **arriba** de `flowOn` en la cadena - el bloque `flow { }` y los operadores previos.

```kotlin
flow {
    emit(readFromDatabase())    // corre en IO
}
.map { transform(it) }          // tambien en IO (esta arriba del flowOn)
.flowOn(Dispatchers.IO)         // afecta todo lo de ARRIBA
.collect { render(it) }         // corre en el contexto del colector (ej. Main)
```

**El punto clave:** `flowOn` solo afecta hacia arriba, no hacia abajo. Todo lo que esta despues, incluido el `collect`, sigue en el contexto de quien colecta - esto respeta "context preservation". Es el equivalente de `withContext` pero para flows, porque dentro de un `flow { }` no se puede usar `withContext` directamente para cambiar el contexto de emision.

---

## 8. Backpressure: cuando el productor es mas rapido que el consumidor

Por defecto, un flow es secuencial: el productor **espera** a que el colector termine de procesar cada valor antes de emitir el siguiente. Hay tres operadores para manejar la diferencia de velocidad:

**`buffer()`**: el productor no espera, encola los valores. Productor y consumidor corren concurrentemente.

```kotlin
flow { /* emite rapido */ }
    .buffer()
    .collect { slowProcess(it) }
```

**`conflate()`**: si el colector esta ocupado, descarta los valores intermedios y se queda con el mas reciente disponible. Para cuando solo importa el ultimo valor (ej. posicion de un slider).

```kotlin
flow { /* emite rapido */ }
    .conflate()
    .collect { slowProcess(it) }
```

**`collectLatest { }`**: cancela el procesamiento en curso si llega un valor nuevo, y reinicia con ese valor. Ideal cuando un valor nuevo invalida el trabajo anterior (ej. busqueda mientras se escribe: el usuario teclea otra letra, se cancela la busqueda anterior).

```kotlin
flow { /* emite */ }
    .collectLatest { value ->
        slowProcess(value)   // si llega un valor nuevo, CANCELA este y empieza de nuevo
    }
```

**La distincion fina `conflate` vs `collectLatest`:** `conflate` deja **terminar** el procesamiento actual y luego salta al ultimo valor disponible, sin interrumpir. `collectLatest` **cancela activamente** el procesamiento en curso en cuanto llega un valor nuevo.

| Operador | Que hace | Cuando usarlo |
|---|---|---|
| (ninguno) | Productor espera al consumidor | Default, procesamiento acoplado |
| `buffer` | Encola, productor no espera | Procesar todo, sin perder valores |
| `conflate` | Descarta intermedios, deja el ultimo | Solo importa el valor mas reciente |
| `collectLatest` | Cancela el anterior al llegar uno nuevo | El valor nuevo invalida el trabajo previo |

**Conexion con `flowOn`:** `flowOn` internamente introduce un buffer al cruzar dispatchers, porque productor y consumidor quedan en hilos distintos.

---

## 9. Operadores de combinacion: `combine`, `zip`, `merge`

**`combine`**: combina las **ultimas** emisiones de varios flows cada vez que cualquiera de ellos emite.

```kotlin
combine(userFlow, settingsFlow) { user, settings ->
    UiState(user, settings)
}
```

**`zip`**: empareja emisiones por **posicion** - espera a que ambos flows tengan un valor nuevo correspondiente antes de combinar.

```kotlin
flowA.zip(flowB) { a, b -> a + b }
```

**`merge`**: junta las emisiones de varios flows en uno solo, sin combinarlas - cada emision pasa tal cual, de cualquiera de los flows de origen.

```kotlin
merge(flowA, flowB).collect { println(it) }
```

La diferencia practica: `combine` reacciona a cualquier cambio usando los valores mas recientes de todos; `zip` empareja uno a uno en orden; `merge` solo intercala sin combinar nada.

---

## 10. Ejemplo real de Android: lifecycle en Compose/Vistas

```kotlin
// ViewModel: Room como fuente reactiva
val uiState: StateFlow<UiState> = repo.observeUser()
    .map { UiState.Success(it) }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), UiState.Loading)
```

```kotlin
// Compose: respeta el lifecycle automaticamente
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
}
```

```kotlin
// Sistema de Vistas: el equivalente manual
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state -> render(state) }
    }
}
```

`collectAsStateWithLifecycle()` usa `repeatOnLifecycle(STARTED)` por debajo - corta el collect cuando la app va a background. Junto con `WhileSubscribed(5000)` en el ViewModel, el par completo logra que el upstream (ej. Room) se apague cuando no hay UI observando, y se reactive al volver, sin reiniciar en cada rotacion de pantalla.

**La trampa comun:** `collectAsState()` (sin `WithLifecycle`) viene de Compose puro y no respeta el lifecycle de Android - sigue colectando en background. Para Android siempre conviene `collectAsStateWithLifecycle()`.

---

## Resumen - tabla de decision rapida

| Necesito... | Herramienta |
|---|---|
| Stream bajo demanda con operadores | Flow frio (`flow { }`) |
| Estado de UI que yo empujo (one-shot) | `MutableStateFlow` |
| Estado de UI desde un flow frio (Room, DataStore) | `stateIn` |
| Eventos one-shot (navegar, snackbar) con garantia de entrega | `Channel` + `receiveAsFlow()` |
| Eventos que varios observadores deben recibir a la vez | `SharedFlow` |
| Compartir un flow frio entre colectores sin re-ejecutarlo | `stateIn` / `shareIn` |
| Cambiar el dispatcher de un flow | `flowOn` (solo afecta upstream) |
| Productor mas rapido que el consumidor, sin perder valores | `buffer` |
| Solo importa el valor mas reciente | `conflate` |
| El valor nuevo invalida el procesamiento anterior | `collectLatest` |
| Combinar ultimos valores de varios flows | `combine` |
| Emparejar por posicion | `zip` |
| Intercalar sin combinar | `merge` |
| Colectar en Compose respetando el lifecycle | `collectAsStateWithLifecycle()` |

---

## Frases para entrevista

- "Flow frio arranca con cada colector; caliente emite exista o no colector y todos comparten la emision."
- "StateFlow es un SharedFlow con replay 1, conflation y valor inicial obligatorio."
- "SharedFlow es broadcast a todos los colectores; Channel reparte cada valor a uno solo."
- "flowOn solo afecta el upstream; el collect sigue en el contexto del colector, por context preservation."
- "conflate deja terminar y salta al ultimo; collectLatest cancela el actual y reinicia."
- "WhileSubscribed(5000) evita reiniciar el upstream en cada rotacion de pantalla."

## Trampas tipicas de entrevista

**"?Por que no usar StateFlow para un evento de navegacion?"** - Porque al re-suscribirse (ej. tras rotar) un colector nuevo recibe el ultimo valor otra vez y navegaria de nuevo. Los eventos van en Channel o SharedFlow con replay=0.

**"?conflate y collectLatest son lo mismo?"** - No: conflate deja terminar el procesamiento actual y salta al ultimo valor disponible; collectLatest cancela activamente el procesamiento en curso al llegar uno nuevo.

**"Si pongo flowOn(IO) al final de la cadena, ?el collect corre en IO?"** - No, flowOn solo afecta hacia arriba; el collect corre en el contexto de quien colecta (ej. Main si se lanza desde viewModelScope).

**"?Por que coleccionar con collectAsState() en vez de collectAsStateWithLifecycle() es un problema en Android?"** - Porque collectAsState() no respeta el lifecycle de Android y sigue colectando aunque la app este en background, desperdiciando recursos.
