# CoroutineContext & Dispatchers / CoroutineContext y Dispatchers a fondo

🌐 [English version](07-concurrencia-context.en.md)

---

## 1. Qué es el CoroutineContext

Un `CoroutineContext` es el **conjunto de datos que define cómo se comporta una corrutina**: en qué hilo corre, quién es su Job, cómo se llama, cómo maneja errores. Es una **colección de elementos**, como un mapa donde cada elemento tiene una clave única.

Los cuatro elementos principales:
- **`Job`** → controla el ciclo de vida (cancelar, esperar, estado).
- **`CoroutineDispatcher`** → en qué hilo/pool corre (Main, IO, Default).
- **`CoroutineName`** → un nombre, útil para debugging.
- **`CoroutineExceptionHandler`** → cómo maneja excepciones no capturadas.

```kotlin
launch(Dispatchers.IO + CoroutineName("fetch") + SupervisorJob()) {
    // contexto: dispatcher IO, nombre "fetch", un supervisor job
}
```

---

## 2. El operador `+`: cómo se combinan los elementos

Los elementos se combinan con `+`. Cada elemento tiene una **clave única**, así que combinar dos del mismo tipo hace que **el de la derecha gane** (reemplaza al de la izquierda).

```kotlin
val context = Dispatchers.IO + CoroutineName("A")
val newContext = context + CoroutineName("B")  // "B" reemplaza "A"
// Resultado: Dispatchers.IO + CoroutineName("B")
```

Sumar dos dispatchers **no los acumula** — el segundo pisa al primero (misma clave). Sumar un dispatcher y un nombre sí coexisten (claves distintas).

---

## 3. Herencia de contexto: cómo un hijo hereda del padre

Cuando se lanza una corrutina hija, **hereda el contexto del padre**, con dos matices:

1. Los elementos que **no** se especifican en el hijo → se heredan del padre.
2. Los que **sí** se especifican → sobreescriben al del padre.
3. **Excepción: el `Job` NUNCA se hereda** — cada corrutina crea su propio `Job` hijo (para mantener la jerarquía padre-hijo).

```kotlin
viewModelScope.launch(Dispatchers.IO) {   // padre: dispatcher IO
    launch {                               // hijo sin dispatcher especificado
        // HEREDA IO del padre
    }
    launch(Dispatchers.Default) {          // hijo con dispatcher propio
        // usa Default (sobreescribe la herencia)
    }
}
```

Por eso `viewModelScope.launch { }` corre en Main: `viewModelScope` tiene `Dispatchers.Main` en su contexto, y el hijo lo hereda.

---

## 4. Cómo funciona `withContext` por debajo

`withContext(Dispatchers.IO)` **toma el contexto actual y reemplaza el dispatcher** (por la regla del `+`), ejecuta el bloque en ese contexto modificado, y al terminar **vuelve al contexto anterior**.

```kotlin
suspend fun getData() {
    // contexto actual: Main (viewModelScope)
    withContext(Dispatchers.IO) {
        // acá el dispatcher es IO — el resto del contexto se mantiene
        api.fetch()
    }
    // se vuelve a Main automáticamente
}
```

No crea una corrutina nueva (no hay Job hijo nuevo como en `launch`) — solo cambia el contexto para ese bloque y suspende hasta terminar.

---

## 5. El CoroutineDispatcher a fondo

Técnicamente un **`ContinuationInterceptor`** — intercepta la corrutina cada vez que se reanuda y la coloca en el hilo correcto.

- **`Dispatchers.Main`** → hilo principal (UI). Respaldado por el `Handler` del main looper.
- **`Dispatchers.IO`** → pool para bloqueo/I/O, hasta 64 hilos (comparte pool físico con Default).
- **`Dispatchers.Default`** → pool para CPU, tantos hilos como cores.
- **`Dispatchers.Unconfined`** → arranca en el hilo llamador, salta al hilo donde se reanude. Casi nunca en producción.

`Main.immediate`: si ya se está en el hilo principal, ejecuta de inmediato sin re-despachar (evita un salto innecesario).

**Relación IO/Default:** comparten el mismo pool de hilos físico; la diferencia es la **cuota de paralelismo** (Default = cores, IO = hasta 64). No son dos pools separados, son dos cuotas sobre el mismo pool.

---

## 6. Job dentro del contexto

```kotlin
val job = launch {
    val myJob = coroutineContext[Job]
    println("¿activo? ${myJob?.isActive}")
}
job.cancel()
```

`coroutineContext[Job]` usa la clave `Job` para sacar ese elemento. El contexto es consultable por clave.

---

## 7. Ejemplo real de Android

```kotlin
class DataViewModel(private val repo: Repo) : ViewModel() {

    fun processLargeDataset(data: List<Item>) {
        viewModelScope.launch {                    // hereda Main de viewModelScope
            _state.value = Loading

            val processed = withContext(Dispatchers.Default) {
                data.map { heavyTransform(it) }    // corre en Default (CPU)
            }
            // vuelve a Main automáticamente

            _state.value = Success(processed)      // actualiza UI en Main, seguro
        }
    }
}
```

---

## 8. Inyección de dispatchers (detalle senior)

Nunca hardcodear `Dispatchers.IO` directamente — conviene inyectar los dispatchers para testeabilidad:

```kotlin
interface DispatcherProvider {
    val main: CoroutineDispatcher
    val io: CoroutineDispatcher
    val default: CoroutineDispatcher
}

class Repo(private val dispatchers: DispatcherProvider) {
    suspend fun getData() = withContext(dispatchers.io) {  // inyectado, no hardcoded
        api.fetch()
    }
}
```

En tests, se pasa un `StandardTestDispatcher` y se controla el tiempo virtual. Ver [12 — Testing](12-testing-corrutinas.md).

---

## 9. Errores comunes

**a) Creer que sumar dos dispatchers los combina.** `Dispatchers.IO + Dispatchers.Default` no hace ambos — el segundo (Default) gana porque comparten clave.

**b) Esperar que el Job se herede.** No se hereda; cada corrutina tiene su propio Job hijo.

**c) Hardcodear dispatchers.** Rompe la testeabilidad.

**d) Olvidar que `withContext` vuelve al contexto anterior.** Después de un `withContext(IO)`, se vuelve al dispatcher de antes — no se queda en IO.

---

## 10. Frases para entrevista

- *"El contexto es un mapa de elementos indexado por clave: Job, Dispatcher, Name, ExceptionHandler."*
- *"Se combinan con `+`; mismo tipo, gana el de la derecha."*
- *"El hijo hereda todo del padre menos el Job, que siempre es propio."*
- *"withContext reemplaza el dispatcher para un bloque y vuelve al anterior — no crea corrutina nueva."*
- *"Inyectar dispatchers, no hardcodearlos — es lo que hace el código testeable."*
- *"IO y Default son dos cuotas sobre el mismo pool de hilos, no dos pools separados."*

---

## 11. Trampas típicas de entrevista

**"¿Qué pasa si se hace `withContext(Dispatchers.IO + Dispatchers.Default)`?"**
Gana Default, porque ambos son dispatchers (misma clave) y el de la derecha sobreescribe. No corren en ambos.

**"¿Cómo funciona `withContext` internamente?"**
Toma el contexto actual, reemplaza el elemento que se le pasa, ejecuta el bloque, y vuelve al contexto anterior. No crea corrutina nueva, por eso es más barato que `async`/`launch` para cambiar de hilo.
