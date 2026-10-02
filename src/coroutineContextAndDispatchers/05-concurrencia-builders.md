# Coroutine Builders / Builders de corrutinas

🌐 [English version](05-concurrencia-builders.en.md)

---

## 1. Qué es un builder

Un **builder** es una función que **crea y lanza** una corrutina. Es el puente entre código normal y código de corrutinas: no se puede llamar una `suspend fun` desde código normal, pero un builder sí abre esa puerta.

---

## 2. `launch` — dispara y olvida

Lanza una corrutina que **no devuelve un resultado**. Sirve para efectos secundarios: guardar algo, actualizar estado, disparar una acción. Devuelve un `Job`, que es un **handle** para controlar la corrutina (cancelarla, esperar a que termine), no su resultado.

```kotlin
fun main() = runBlocking {
    val job: Job = launch {
        delay(1000)
        println("Tarea terminada")
    }
    println("Corrutina lanzada")
    job.join()   // espera a que termine (opcional)
}
// Salida: "Corrutina lanzada" ... (1s) ... "Tarea terminada"
```

Lo que da el `Job`:
- `job.join()` → suspende hasta que la corrutina termina.
- `job.cancel()` → la cancela.
- `job.isActive` / `job.isCompleted` / `job.isCancelled` → su estado.

---

## 3. `async` — cuando se necesita un resultado

`async` lanza una corrutina que **sí devuelve un valor**. Devuelve un `Deferred<T>` (un `Job` que además carga un resultado), y se obtiene el valor con `.await()`.

```kotlin
fun main() = runBlocking {
    val deferred: Deferred<Int> = async {
        delay(1000)
        42
    }
    println("Calculando...")
    val result = deferred.await()   // suspende hasta tener el resultado
    println("Resultado: $result")
}
```

Pero el valor real de `async` **no** es lanzar una sola tarea — para eso `withContext` es mejor. Su punto es **ejecutar varias tareas de forma solapada y obtener sus resultados**.

```kotlin
suspend fun loadDashboard() = coroutineScope {
    val user = async { api.getUser() }      // arranca ya
    val posts = async { api.getPosts() }    // arranca ya, en paralelo
    val notifications = async { api.getNotifications() }

    // las tres ya están corriendo; ahora recoge resultados
    Dashboard(user.await(), posts.await(), notifications.await())
}
```

Si cada llamada tarda 1s, en secuencia serían 3s; así tardan ~1s.

**Importante:** si esas tres tareas son llamadas de red, esto es **concurrencia de I/O**, no paralelismo de CPU — no necesita `Dispatchers.Default`. El dispatcher `IO` vive dentro del repositorio (main-safe), no en el `async`. `Default` solo se usaría si el `async` hiciera cálculo de CPU. Ver [04 — Fundamentos](04-concurrencia-fundamentos.md) sección 4 y 12.

---

## 4. El contraste `launch` vs `async`

| | `launch` | `async` |
|---|---|---|
| Devuelve | `Job` | `Deferred<T>` |
| ¿Da resultado? | No | Sí (con `.await()`) |
| Uso | Efecto secundario | Trabajo concurrente con resultado |
| Excepción | Se propaga de inmediato al scope | Se lanza al hacer `.await()` |

`launch` vs `async` **no** cambia si algo es concurrente o no — eso lo decide cómo se lanzan (ver 04-fundamentos). Cambia **si se necesita recoger el resultado**:

```kotlin
// launch: NO se necesita resultado (efecto secundario)
launch { analytics.track("dashboard_opened") }

// async: SÍ se necesita el resultado
val user = async { api.getUser() }
render(user.await())
```

---

## 5. `runBlocking` — el puente desde código bloqueante

`runBlocking` **bloquea el hilo actual** hasta que la corrutina dentro termina. Es el opuesto filosófico de todo lo demás (bloquea en vez de suspender).

```kotlin
fun main() = runBlocking {   // bloquea el hilo main hasta terminar todo dentro
    launch { delay(1000); println("A") }
    launch { delay(500); println("B") }
}
```

Dónde SÍ se usa:
- En la función `main()` de una app de consola.
- En **tests** (aunque hoy se prefiere `runTest` — ver [12 — Testing](12-testing-corrutinas.md)).
- Puentes puntuales con código legacy bloqueante.

Dónde NO: **nunca en producción Android**, porque bloquearía el hilo principal y congelaría la UI. En Android se usa `viewModelScope`/`lifecycleScope`.

---

## 6. `withContext` — cambiar de contexto, no crear corrutina

`withContext` cambia el contexto (típicamente el dispatcher) para ejecutar un bloque y devuelve su resultado de forma **secuencial** — suspende hasta que termina. No crea una corrutina hija nueva como `launch`/`async`; solo cambia el contexto para ese bloque y suspende.

```kotlin
suspend fun getUser(): User = withContext(Dispatchers.IO) {
    api.fetchUser()
}
```

**`withContext` vs `async` para cambiar de contexto:** si solo se quiere mover un bloque a otro dispatcher y usar su resultado enseguida, `async` + `await()` inmediato es un antipatrón — toda la maquinaria de concurrencia sin ganar concurrencia.

```kotlin
// ANTIPATRÓN: async + await inmediato para cambiar de contexto
val user = async(Dispatchers.IO) { api.getUser() }.await()

// CORRECTO: withContext para lo mismo
val user = withContext(Dispatchers.IO) { api.getUser() }
```

`async` gana su lugar cuando se lanzan **varias** tareas para que corran en paralelo/concurrentes y se recogen los resultados después.

```kotlin
// async brilla aquí: dos llamadas EN PARALELO/CONCURRENTES
suspend fun loadScreen() = coroutineScope {
    val user = async(Dispatchers.IO) { api.getUser() }
    val posts = async(Dispatchers.IO) { api.getPosts() }
    Screen(user.await(), posts.await()) // ambas corrieron a la vez
}
// con withContext serían secuenciales: getUser TERMINA antes de empezar getPosts
```

| | `withContext` | `async` |
|---|---|---|
| Devuelve | El resultado directo | `Deferred<T>` (se espera con `await`) |
| Ejecución | Secuencial (suspende hasta terminar) | Concurrente (sigue sin esperar) |
| Crea corrutina nueva | No, solo cambia contexto | Sí, una corrutina hija |
| Uso ideal | Cambiar de dispatcher para un bloque | Paralelizar/concurrencia de varias tareas |
| Overhead | Menor | Mayor (coordina una hija) |

**Regla mental:** una tarea, cambiar de contexto, usar el resultado enseguida → `withContext`. Varias tareas que se quieren solapadas → `async` + `await`. Si se escribe `async { ... }.await()` en la misma línea, casi siempre debería ser `withContext`.

---

## 7. Ejemplo real de Android

`launch` es el builder que se usa el 90% del tiempo en Android:

```kotlin
class OrderViewModel(private val repo: OrderRepository) : ViewModel() {

    private val _state = MutableStateFlow<OrderState>(OrderState.Idle)
    val state = _state.asStateFlow()

    // launch: efecto secundario (actualizar estado), no devuelve nada
    fun placeOrder(order: Order) {
        viewModelScope.launch {
            _state.value = OrderState.Loading
            try {
                repo.submit(order)
                _state.value = OrderState.Success
            } catch (e: Exception) {
                _state.value = OrderState.Error(e.message)
            }
        }
    }
}
```

`async` cuando se necesita **combinar resultados** de varias llamadas para armar una pantalla:

```kotlin
fun loadProfileScreen() {
    viewModelScope.launch {                    // launch: el contenedor
        _state.value = ProfileState.Loading
        try {
            // async: las dos llamadas se solapan dentro del launch
            val profile = async { repo.getProfile() }
            val activity = async { repo.getRecentActivity() }
            _state.value = ProfileState.Success(
                profile.await(),
                activity.await()
            )
        } catch (e: Exception) {
            _state.value = ProfileState.Error(e.message)
        }
    }
}
```

Patrón idiomático: **`launch` como contenedor** (el efecto secundario de actualizar el estado), y **`async` adentro** para solapar las llamadas de red.

Y el repositorio, main-safe por dentro:

```kotlin
class Repo(private val api: Api) {
    suspend fun getProfile(): Profile = withContext(Dispatchers.IO) {
        api.getProfile()   // el dispatcher vive acá, no en el async del ViewModel
    }
}
```

---

## 8. Errores comunes

**a) `async` + `await()` inmediato en vez de `withContext`.**

```kotlin
// Antipatrón
val user = async { repo.getUser() }.await()
// Correcto
val user = withContext(Dispatchers.IO) { repo.getUser() }
```

**b) Usar `async` para tareas que en realidad son secuenciales (dependientes).**

```kotlin
// No paraleliza nada: posts NECESITA el id del user
val user = async { repo.getUser() }.await()
val posts = async { repo.getPosts(user.id) }.await()
// Mejor secuencial y claro:
val user = repo.getUser()
val posts = repo.getPosts(user.id)
```

**c) Olvidar que la excepción de `async` espera al `await()`.** Con `launch`, una excepción se propaga de inmediato. Con `async`, la excepción queda "guardada" y **estalla cuando se llama `.await()`**.

```kotlin
val deferred = async { throw RuntimeException("boom") }
// aquí todavía no explota...
deferred.await()  // ← la excepción salta ACÁ
```

**d) `runBlocking` en el hilo principal de Android.** Congela la UI. Nunca.

**e) Usar `Dispatchers.Default` para llamadas de red.** Es concurrencia de I/O, no paralelismo de CPU — el dispatcher va en el repo con `IO`.

---

## 9. Frases para entrevista

- *"`launch` devuelve Job (control); `async` devuelve Deferred (resultado)."*
- *"El valor de `async` es ejecutar tareas solapadas y obtener resultados, no lanzar una sola."*
- *"La excepción de `async` estalla en el `await`, no al lanzarlo."*
- *"`runBlocking` bloquea el hilo — para main() y tests, nunca en producción Android."*
- *"withContext cambia contexto y suspende secuencialmente; async lanza concurrencia y devuelve un Deferred."*
- *"async + await inmediato es un antipatrón: toda la maquinaria de concurrencia sin ganar concurrencia — eso es withContext."*

---

## 10. Trampas típicas de entrevista

**"¿Cuándo `async` NO aporta nada?"**
Con una sola tarea + `await()` inmediato (usar `withContext`), y con tareas secuencialmente dependientes (no hay nada que solapar). Solo brilla con tareas independientes que corren a la vez.

**"Te dan `async(IO) { fetch() }.await()`, ¿qué mejorarías?"**
Cambiarlo a `withContext(IO) { fetch() }`, porque el `async`+`await` inmediato no aporta concurrencia y sí añade el costo de crear y coordinar una corrutina hija.

**"¿launch vs async cambia si es concurrente?"**
No. La concurrencia viene de lanzarlas sin esperar entre una y otra — dos `launch` también se solapan. La diferencia es si se necesita recoger el resultado (async) o no (launch).
