# Cancellation / Cancelacion

[English version](08-concurrencia-cancelacion.en.md)

---

## 1. Por que importa la cancelacion

Cuando el usuario cierra una pantalla, navega a otra, o cancela una accion explicitamente, las tareas en curso deberian detenerse. Si no se detienen: se desperdicia red/CPU, se puede intentar actualizar una UI que ya no existe (crash), o se filtran recursos.

Structured concurrency (ver [06 - Scopes](06-concurrencia-scopes.es.md)) ya resuelve una parte: cuando `viewModelScope` se cancela, cancela a todos sus hijos. Pero **que tan rapido y correctamente** se cancela una corrutina depende de que tan bien escrito este el codigo adentro. Eso es lo que cubre este tema.

---

## 2. La cancelacion es cooperativa

Esta es la idea central. Cancelar una corrutina **no la mata de golpe** como un `Thread.interrupt()` forzado. En vez de eso, Kotlin marca el `Job` como cancelado, y es **responsabilidad del codigo dentro de la corrutina** chequear ese estado y detenerse.

```kotlin
val job = launch {
    repeat(1000) { i ->
        println("trabajando $i")
        Thread.sleep(500)   // NO chequea cancelacion, NO suspende
    }
}
delay(1300)
job.cancel()   // pide la cancelacion...
// ...pero el bucle de arriba NUNCA la nota, sigue corriendo para siempre
```

Si el codigo adentro nunca suspende ni chequea el estado, la cancelacion **nunca ocurre de verdad** aunque se haya "pedido". Por eso se dice que la cancelacion es cooperativa: la corrutina tiene que cooperar revisando si fue cancelada.

---

## 3. Donde SI coopera: puntos de suspension

Las funciones de `kotlinx.coroutines` que suspenden (`delay`, `yield`, y las funciones de los builders como `withContext`) **ya chequean la cancelacion automaticamente** en cada punto de suspension. Si el Job fue cancelado, lanzan una `CancellationException` ahi mismo.

```kotlin
val job = launch {
    repeat(1000) { i ->
        println("trabajando $i")
        delay(500)   // SI chequea cancelacion en cada suspension
    }
}
delay(1300)
job.cancel()   // ahora SI detiene el bucle, en el próximo delay()
```

**La regla practica:** si el codigo dentro de una corrutina hace trabajo largo sin ningun punto de suspension (un bucle pesado de CPU, un calculo largo sin `delay`/llamadas suspend), esa corrutina **no es cancelable** hasta que termine ese tramo.

---

## 4. Cancelar codigo que no suspende: `isActive` y `ensureActive`

Para un bucle de CPU que no tiene ningun punto de suspension natural, hay que chequear el estado manualmente.

```kotlin
val job = launch(Dispatchers.Default) {
    var i = 0
    while (isActive) {       // chequeo manual: sale del bucle si se cancelo
        i++
        // trabajo de CPU puro, sin suspend
    }
}
```

`isActive` es una propiedad de extension disponible dentro de una corrutina (viene de `CoroutineScope`/`Job`). Devuelve `false` una vez que se pidio la cancelacion.

La alternativa `ensureActive()` hace lo mismo pero **lanza** `CancellationException` en vez de solo devolver un booleano - util para cortar de inmediato sin tener que envolver todo en un `if`:

```kotlin
val job = launch(Dispatchers.Default) {
    repeat(1_000_000) { i ->
        ensureActive()       // lanza si fue cancelado, corta aqui mismo
        heavyComputation(i)
    }
}
```

---

## 5. Limpieza de recursos al cancelar: `finally` y `NonCancellable`

Cuando una corrutina se cancela, Kotlin lanza una `CancellationException` en el punto de suspension. Eso significa que un bloque `try/finally` **si se ejecuta**, igual que con cualquier excepcion - es el lugar correcto para liberar recursos (cerrar un archivo, un socket, loguear que se cancelo).

```kotlin
val job = launch {
    try {
        repeat(1000) { i ->
            println("trabajando $i")
            delay(500)
        }
    } finally {
        println("limpieza: cerrando recursos")   // SI corre al cancelar
    }
}
```

**El problema:** dentro de ese `finally`, la corrutina ya esta en proceso de cancelacion, asi que cualquier funcion `suspend` que se llame ahi **tambien lanzara inmediatamente** `CancellationException` (porque el Job ya esta cancelado). Si la limpieza necesita hacer algo suspendible (ej. una ultima escritura a disco con una funcion suspend), hay que envolverlo en `withContext(NonCancellable)`:

```kotlin
val job = launch {
    try {
        repeat(1000) { i -> delay(500) }
    } finally {
        withContext(NonCancellable) {
            delay(1000)              // esto SI se ejecuta, aunque el Job este cancelado
            saveStateBeforeExit()    // una suspend fun que necesita correr igual
        }
    }
}
```

`NonCancellable` es un contexto especial que ignora el estado de cancelacion del Job padre - solo deberia usarse para limpieza critica y breve, nunca para seguir hacienda trabajo normal.

---

## 6. `CancellationException` no es un error normal

Cuando una corrutina se cancela, internamente se lanza una `CancellationException`. Esto es **intencional y se propaga de forma especial**: las corrutinas padre e hijas la reconocen como "cancelacion normal", no como un fallo real, asi que **no** dispara el `CoroutineExceptionHandler` ni se trata como un crash.

**La trampa:** si se captura `Exception` de forma generica sin volver a lanzar la `CancellationException`, se rompe la cancelacion:

```kotlin
// MAL: traga la CancellationException, la corrutina nunca se cancela de verdad
try {
    delay(1000)
} catch (e: Exception) {
    log(e)   // esto tambien atrapa CancellationException y la esconde
}

// BIEN: capturar especificamente lo que se espera, dejar pasar CancellationException
try {
    delay(1000)
} catch (e: IOException) {
    log(e)
}
// o volver a lanzarla explicitamente si se captura Exception por alguna razon:
catch (e: Exception) {
    if (e is CancellationException) throw e
    log(e)
}
```

Esta es una de las trampas mas comunes en entrevista: un `catch (e: Exception)` generico alrededor de una llamada suspend silenciosamente rompe la cancelacion cooperativa.

---

## 7. Cancelacion explicita por el usuario (ejemplo real de Android)

Caso tipico: una busqueda que se cancela si el usuario escribe algo nuevo antes de que termine la anterior.

```kotlin
class SearchViewModel(private val repo: SearchRepository) : ViewModel() {

    private var searchJob: Job? = null

    fun onQueryChanged(query: String) {
        searchJob?.cancel()              // cancela la busqueda anterior si seguia corriendo
        searchJob = viewModelScope.launch {
            delay(300)                    // debounce
            val results = repo.search(query)
            _state.value = SearchState.Success(results)
        }
    }
}
```

Guardar la referencia al `Job` y cancelarlo manualmente antes de lanzar uno nuevo es el patron estandar para "la ultima accion del usuario invalida la anterior". (Este mismo caso se resuelve tambien con el operador `collectLatest` en Flows, que cancela automaticamente - ver [10 - Flows](10-flows.es.md).)

---

## 8. Cancelacion en cascada: cancelar el padre

Por structured concurrency, cancelar un scope padre cancela automaticamente a todos sus hijos activos - no hace falta cancelar cada uno manualmente.

```kotlin
class MyViewModel : ViewModel() {
    fun loadEverything() {
        viewModelScope.launch { taskA() }   // hijo 1
        viewModelScope.launch { taskB() }   // hijo 2
    }
}
// Cuando el ViewModel se destruye, viewModelScope.cancel() se llama solo (onCleared),
// y eso cancela taskA y taskB automaticamente - sin codigo extra
```

---

## Resumen - tabla de decision rapida

| Situacion | Herramienta |
|---|---|
| Codigo con `delay`/llamadas suspend normales | Se cancela solo, nada que hacer |
| Bucle de CPU puro sin puntos de suspension | `isActive` (chequeo manual) o `ensureActive()` (lanza) |
| Liberar recursos al cancelar | `try/finally` |
| Limpieza que necesita codigo suspend | `withContext(NonCancellable)` dentro del `finally` |
| Capturar excepciones sin romper cancelacion | No capturar `Exception` generico sin relanzar `CancellationException` |
| Invalidar una tarea anterior con una nueva | Guardar el `Job` y llamar `.cancel()` antes de relanzar |
| Cancelar varias tareas relacionadas a la vez | Cancelar el scope padre (structured concurrency) |

---

## Frases para entrevista

- "La cancelacion es cooperativa: no mata la corrutina de golpe, marca el Job y el codigo debe chequearlo."
- "Los puntos de suspension como delay chequean cancelacion solos; un bucle de CPU puro necesita isActive o ensureActive."
- "finally se ejecuta al cancelar, pero una suspend fun ahi dentro lanza CancellationException igual - para eso existe NonCancellable."
- "Un catch generico de Exception sin relanzar CancellationException rompe la cancelacion cooperativa."

## Trampas tipicas de entrevista

**"Tengo un bucle pesado de CPU dentro de una corrutina y job.cancel() no lo detiene, ?por que?"** - Porque el bucle no tiene ningun punto de suspension donde se chequee la cancelacion. Hay que agregar `isActive` o `ensureActive()` dentro del bucle.

**"?Por que un try/catch(Exception) puede romper la cancelacion?"** - Porque `CancellationException` hereda de `Exception`, y capturarla sin relanzarla le hace creer a la jerarquia de corrutinas que la tarea sigue viva, rompiendo structured concurrency.

**"Necesito guardar algo en disco cuando se cancela una corrutina, ?como lo hago si disco es una suspend fun?"** - Envolviendo esa llamada en `withContext(NonCancellable)` dentro de un bloque `finally`, para que se ejecute a pesar de que el Job ya este cancelado.
