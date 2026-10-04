# Exception Handling / Manejo de excepciones

[English version](09-concurrencia-excepciones.en.md)

---

## 1. Por que el manejo de excepciones en corrutinas es distinto

En codigo normal, una excepcion sube por la pila de llamadas hasta que algo la captura. En corrutinas hay una capa extra: las excepciones tambien se propagan por la **jerarquia de structured concurrency** (ver [06 - Scopes](06-concurrencia-scopes.es.md)), y el comportamiento cambia segun el builder (`launch` vs `async`) y el tipo de Job (`Job` vs `SupervisorJob`).

---

## 2. `try/catch` normal: funciona, con matices

Un `try/catch` alrededor de una llamada suspend funciona exactamente como en codigo sincrono:

```kotlin
viewModelScope.launch {
    try {
        val data = repo.getData()   // suspend fun que puede lanzar
        _state.value = Success(data)
    } catch (e: IOException) {
        _state.value = Error(e.message)
    }
}
```

**La trampa:** `CancellationException` tambien es una `Exception`. Un `catch (e: Exception)` generico la atrapa sin querer y rompe la cancelacion cooperativa (ver [08 - Cancelacion](08-concurrencia-cancelacion.es.md) seccion 6). Siempre conviene capturar tipos especificos, o relanzar la `CancellationException` si se captura `Exception`.

---

## 3. `launch` vs `async`: donde explota la excepcion

Esta es la distincion central del tema.

**`launch`: la excepcion se propaga de inmediato**, en cuanto ocurre, hacia el scope padre - no hace falta ningun `await` ni similar para que aparezca.

```kotlin
viewModelScope.launch {
    throw RuntimeException("boom")   // explota ahora mismo, se propaga al padre
}
```

**`async`: la excepcion queda guardada dentro del `Deferred`** y solo se lanza cuando se llama `.await()`. Si nunca se hace `await`, la excepcion no desaparece - igual se propaga al padre por structured concurrency, pero el punto donde un `try/catch` la atraparia es distinto.

```kotlin
val deferred = async { throw RuntimeException("boom") }
// aqui todavia no exploto...
deferred.await()   // la excepcion sale ACA
```

| | `launch` | `async` |
|---|---|---|
| Cuando se lanza | De inmediato | Al llamar `.await()` |
| Donde capturarla | `try/catch` alrededor del `launch` o del codigo dentro | `try/catch` alrededor del `.await()` |

---

## 4. Propagacion por la jerarquia: `Job` normal

Con un `Job` normal (el que crea `coroutineScope`), una excepcion no capturada en un hijo **cancela al padre y a todos los hermanos**, y luego se re-lanza hacia arriba.

```kotlin
coroutineScope {
    launch { throw Exception("boom") }       // falla
    launch { delay(1000); doWork() }         // se cancela, NUNCA llega a doWork()
}
// la excepcion sigue subiendo mas alla de este coroutineScope
```

Esto es "todo o nada": un fallo tumba al grupo completo. Es el comportamiento correcto cuando las tareas forman una unidad indivisible.

---

## 5. Propagacion con `SupervisorJob`

Con `SupervisorJob` (el que usa `viewModelScope` y `supervisorScope`), el fallo de un hijo **no** cancela a los hermanos - cada uno es independiente.

```kotlin
supervisorScope {
    launch { throw Exception("boom") }       // falla, pero solo este hijo
    launch { delay(1000); doWork() }         // SI llega a ejecutarse
}
```

Esto es lo que permite el patron de "varios `launch` hermanos, cada uno con su propio estado" que se vio en los temas anteriores: el fallo de uno (ej. no cargo el balance) no tumba a los otros (perfil y transacciones siguen cargando bien).

**Importante:** aunque el hermano sobrevive, la excepcion del hijo que fallo **igual debe manejarse** - si no se captura con `try/catch` dentro de ese `launch`, termina llegando al `CoroutineExceptionHandler` (ver seccion siguiente) o, si no hay uno, crashea la app.

---

## 6. `CoroutineExceptionHandler`: la red de seguridad final

Es un elemento del `CoroutineContext` (ver [07 - Context y Dispatchers](07-concurrencia-context.es.md)) que captura excepciones **no manejadas** que terminan de propagarse por toda la jerarquia, antes de que crasheen la app. Es el ultimo punto de captura, no un reemplazo de `try/catch`.

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    Log.e("MyApp", "Excepcion no capturada: ${exception.message}")
}

viewModelScope.launch(handler) {
    throw RuntimeException("boom")   // no hay try/catch, cae en el handler
}
```

**Reglas clave:**
- Solo funciona con `launch`, no con `async` - en `async` la excepcion vive en el `Deferred` y se espera en el `.await()`, el handler no la intercepta.
- Solo captura excepciones que **llegan hasta la raiz** del arbol de corrutinas (normalmente se instala en el scope de nivel mas alto, no en cada `launch` hijo).
- No evita que el `Job` se cancele - solo da un lugar centralizado para loguear/reportar antes de que la excepcion se pierda.

---

## 7. `runCatching` como alternativa funcional

Para evitar try/catch anidados, es comun envolver una llamada suspend en `runCatching`, que devuelve un `Result<T>`:

```kotlin
viewModelScope.launch {
    val result = runCatching { repo.getData() }
    _state.value = result.fold(
        onSuccess = { Success(it) },
        onFailure = { Error(it.message) }
    )
}
```

**Ojo:** `runCatching` tambien atrapa `CancellationException` por defecto (hereda de `Throwable`, no se trata distinto). Si se usa dentro de una corrutina que puede cancelarse, conviene relanzarla:

```kotlin
val result = runCatching { repo.getData() }
    .onFailure { if (it is CancellationException) throw it }
```

---

## 8. Ejemplo real de Android: combinando todo

```kotlin
class AccountViewModel(private val repo: AccountRepository) : ViewModel() {

    private val handler = CoroutineExceptionHandler { _, e ->
        Log.e("AccountVM", "Error no capturado", e)
    }

    private val _balance = MutableStateFlow<BalanceState>(BalanceState.Loading)
    val balance = _balance.asStateFlow()

    fun loadBalance() {
        // SupervisorJob implicito de viewModelScope: este launch no tumba otros
        viewModelScope.launch(handler) {
            try {
                _balance.value = BalanceState.Success(repo.getBalance())
            } catch (e: IOException) {
                _balance.value = BalanceState.Error(e.message)
            }
            // CancellationException NO se captura aqui porque no coincide con IOException,
            // asi que la cancelacion cooperativa sigue funcionando bien
        }
    }
}
```

---

## Resumen - tabla de decision rapida

| Necesito... | Herramienta |
|---|---|
| Capturar un error esperado (red, IO) en un launch | `try/catch` especifico alrededor del codigo |
| Capturar el error de una tarea con resultado | `try/catch` alrededor del `.await()` |
| Que un hijo fallido no tumbe a sus hermanos | `SupervisorJob` / `supervisorScope` / `viewModelScope` |
| Una red de seguridad final para errores no capturados | `CoroutineExceptionHandler` (solo con `launch`) |
| Evitar try/catch anidados | `runCatching` + `fold` (cuidado con CancellationException) |

---

## Frases para entrevista

- "launch propaga la excepcion de inmediato; async la guarda en el Deferred hasta el await."
- "Con Job normal, un fallo cancela a todo el grupo; con SupervisorJob, solo afecta al hijo que fallo."
- "CoroutineExceptionHandler es la red de seguridad final, solo funciona con launch, no reemplaza try/catch."
- "CancellationException hereda de Exception - un catch generico sin relanzarla rompe la cancelacion cooperativa."

## Trampas tipicas de entrevista

**"Tengo un try/catch(Exception) generico alrededor de una llamada suspend, ?que problema tiene?"** - Que tambien atrapa `CancellationException`, rompiendo la cancelacion cooperativa de la corrutina. Hay que capturar tipos especificos o relanzarla.

**"?Por que un CoroutineExceptionHandler puesto en un async no funciona?"** - Porque en async la excepcion queda guardada en el Deferred y se lanza recien en el await - el handler nunca la intercepta ahi.

**"Tres launch hermanos en viewModelScope, uno lanza una excepcion sin capturar, ?que pasa?"** - Como viewModelScope usa SupervisorJob, los otros dos siguen corriendo normalmente; la excepcion del que fallo sube hasta el CoroutineExceptionHandler si hay uno instalado, o crashea la app si no hay ninguno.
