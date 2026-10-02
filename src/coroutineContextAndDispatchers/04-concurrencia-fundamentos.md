# Concurrency & Coroutines — Fundamentals / Concurrencia y Corrutinas — Fundamentos

🌐 [English version](04-concurrencia-fundamentos.en.md)

---

## 1. Los cuatro conceptos y cómo se relacionan

No son sinónimos. Son capas distintas de la misma pregunta: "¿cómo hago varias cosas?".

| Concepto | Qué es | Nivel |
|---|---|---|
| **Concurrencia** | Varias tareas en progreso, turnándose | Modelo / estructura |
| **Paralelismo** | Varias tareas en el mismo instante (varios cores) | Modelo / ejecución |
| **Código asíncrono** | No bloquear mientras se espera | Técnica para lograr concurrencia |
| **Corrutinas** | Herramienta de Kotlin que usa async para dar concurrencia barata | Implementación |

La frase de Rob Pike que lo resume:
> **Concurrencia es lidiar con muchas cosas a la vez. Paralelismo es hacer muchas cosas a la vez.**

---

## 2. Los tres modos de ejecutar tareas (el mapa mental clave)

Esto es lo que más se confunde. Cuando se quiere "hacer varias cosas", hay **tres** escenarios distintos, no dos. La línea divisoria son dos preguntas: **¿las tareas dependen entre sí?** y **¿el trabajo es de espera o de CPU?**

```
"Hacer varias tareas"
│
├── ¿Una depende del resultado de la otra?
│   │
│   ├── SÍ  → SECUENCIAL (se espera una antes de lanzar la otra, obligado)
│   │
│   └── NO  → pueden solaparse... ¿qué tipo de trabajo?
│            │
│            ├── ESPERA (I/O: red, disco) → CONCURRENCIA
│            │   Ej: async { api.getUser() } + async { api.getPosts() }
│            │   Dispatcher: IO (o suspend main-safe)
│            │   Funciona en 1 core ✓
│            │
│            └── CÁLCULO (CPU) → PARALELISMO
│                Ej: async(Default) { applyFilter(img) }
│                Dispatcher: Default
│                Necesita varios cores ✓
```

**La prueba mental para distinguir concurrencia de paralelismo:**
> *"¿Esto podría funcionar igual en un solo core?"*
> - Sí → es **concurrencia** (solapa esperas).
> - No, necesita varios cores → es **paralelismo** (solapa cómputo real).

---

## 3. Concurrencia

**Definición precisa:** múltiples tareas **en progreso durante el mismo período**, intercalándose, pero no necesariamente ejecutándose en el mismo instante.

La palabra clave no es "al mismo tiempo" sino **"en progreso a la vez"**. Varias tareas están empezadas y sin terminar; el sistema alterna entre ellas. En un solo hilo, en cualquier instante exacto solo una ejecuta código; las demás están suspendidas esperando turno.

**Analogía:** un cocinero con tres ollas. Hay tres platos en progreso, pero en el instante exacto el cocinero solo revuelve una olla; las otras dos se cocinan solas (suspendidas).

**Aclaraciones importantes:**
- Concurrencia **no** es sinónimo de código asíncrono. Uno es el concepto, el otro la técnica que lo logra.
- Concurrencia requiere tareas **independientes** (si dependen, es secuencial).
- Concurrencia **no** exige ejecución simultánea. Eso último es paralelismo.
- Puede ocurrir en **un solo core**.

---

## 4. Paralelismo

**Definición:** múltiples tareas ejecutándose **literalmente en el mismo instante**. Requiere **varios cores**.

El paralelismo es una *forma* de ejecutar trabajo concurrente cuando hay varios cores. Necesita dos cosas juntas:
1. **Trabajo de CPU** (cálculo real, no espera).
2. **Varios hilos repartidos sobre varios cores** (ej. `Dispatchers.Default`).

Si falta cualquiera de los dos, hay concurrencia como mucho, pero no paralelismo.

**Analogía:** tres cocineros, cada uno con su olla. Varias personas, varias tareas al mismo tiempo.

**El error típico:** creer que varias llamadas de red con `async` son "paralelismo". No lo son — son **concurrencia de I/O**. Las llamadas están *esperando* a la vez, no *ejecutando código* a la vez. Esperar no es ejecutar. Podrían correr sobre un solo core y funcionarían igual → por definición no es paralelismo.

**Paralelismo sin usar tu CPU:** el paralelismo siempre necesita CPU ejecutando, pero no necesariamente el propio. Cuando se hace I/O concurrente, la CPU queda libre (suspende) mientras otro hardware —servidor remoto, disco (DMA), GPU— hace trabajo paralelo real. Desde la app es concurrencia; en el sistema global hay paralelismo, solo que no en el procesador propio.

---

## 5. Secuencial

**Definición:** una cosa después de la otra. Se empieza la tarea 1, se espera a que **termine completamente**, y recién entonces empieza la tarea 2. Nunca hay dos tareas en progreso a la vez.

```kotlin
suspend fun secuencial() {
    val user = api.getUser()      // (1) empieza y ESPERA a que termine — 1s
    val posts = api.getPosts()    // (2) recién ahora empieza — otro 1s
    // Total: 2 segundos
}
```

**Suspender NO es lo mismo que ser concurrente.** Aunque `getUser()` sea `suspend` y libere el hilo, el código sigue siendo secuencial: la línea 2 no empieza hasta que la línea 1 devuelve su valor.

> Suspender es sobre **el hilo** (no se bloquea). Secuencial vs concurrente es sobre **el flujo** (¿se espera una tarea antes de lanzar la otra?). Se puede suspender y aun así ser secuencial.

**Secuencial NO siempre es un error.** Es lo correcto y obligatorio cuando hay **dependencia** entre tareas:

```kotlin
// Secuencial OBLIGATORIO: posts necesita el id del user
val user = api.getUser()              // primero se necesita el user
val posts = api.getPosts(user.id)     // porque se usa su id acá → no se puede paralelizar
```

**La regla:**
- Tareas **independientes** → concurrente (lanzar todas, esperar después) — *opcional*, se puede elegir secuencial igual si se prefiere simplicidad.
- Tareas **dependientes** (una usa el resultado de la otra) → secuencial, sin opción.

**Ojo con la relación concurrencia ↔ dependencia:** son **opuestos**.
- Concurrencia ocurre cuando las tareas **NO dependen** entre sí.
- Dependencia es lo que **fuerza lo secuencial**.
- Si hay dependencia, NO hay concurrencia posible.

**Independiente no obliga a usar `async`.** Habilita, no obliga. Se pueden hacer llamadas independientes de forma secuencial — es válido, solo más lento (tarda la suma en vez del máximo).

**Válido ≠ igual de rápido.** Secuencial da el resultado correcto, pero tarda la SUMA de los tiempos. Concurrente (con async) tarda el MÁXIMO (la más lenta).

```
SECUENCIAL          CONCURRENTE
A: [==1s==]         A: [==1s==]
B:        [==1s==]  B: [==1s==]
   total: 2s           total: 1s
```

**Separar en funciones `suspend` NO cambia el tiempo por sí solo.** Una función es solo un envoltorio de código. Lo que decide el tiempo es *cómo se llaman* (directo = secuencial, con `async` = concurrente), no dónde viven.

---

## 6. Código asíncrono vs multithreading

Ambos logran concurrencia, por caminos distintos:

- **Multithreading:** se usan varios hilos del SO. Cada uno puede correr en un core distinto → puede dar **paralelismo real**. Pero los hilos son caros.
- **Código asíncrono:** se trata de **no bloquear mientras se espera**. Puede lograrse **incluso en un solo hilo** (modelo de JavaScript).

No son lo mismo: async es "no esperar bloqueado"; multithreading es "usar varios hilos".

---

## 7. La distinción que lo define todo: bloquear vs suspender

**Bloquear** un hilo: el hilo queda ocupado sin hacer nada, esperando. No puede ejecutar otra cosa. Desperdicio.

**Suspender** una corrutina: la corrutina se pausa y **libera el hilo**, que queda disponible para otra corrutina. Cuando lo que esperaba está listo, se reanuda (quizá en otro hilo).

```kotlin
// BLOQUEAR: el hilo queda congelado 1 segundo, inútil
fun blockingWork() {
    Thread.sleep(1000)   // el hilo no puede hacer NADA más
}

// SUSPENDER: la corrutina se pausa, el hilo sigue trabajando en otras
suspend fun suspendingWork() {
    delay(1000)          // libera el hilo mientras espera
}
```

**La consecuencia práctica:**

```kotlin
fun main() = runBlocking {
    // delay suspende: 100.000 corren sobre pocos hilos, sin problema
    repeat(100_000) {
        launch { delay(1000) }
    }
    // Si fuera Thread.sleep (bloquea), se necesitarían 100.000 hilos → imposible
}
```

> **Suspender permite miles de tareas concurrentes sobre pocos hilos, porque las que esperan no ocupan hilo.**

---

## 8. Qué es una corrutina

Una corrutina es una **tarea ligera y suspendible** que corre sobre hilos pero la maneja el runtime de Kotlin, no el sistema operativo.

- Es **liviana**: se pueden tener cientos de miles sobre pocos hilos.
- Es **suspendible**: puede pausarse en ciertos puntos y reanudarse, sin bloquear el hilo mientras está pausada.
- Trae **structured concurrency**: vive en un scope que maneja su cancelación y errores.
- Da **paralelismo** solo si el dispatcher la reparte en varios cores.

**Thread vs corrutina:**

| | Thread | Corrutina |
|---|---|---|
| Lo maneja | El sistema operativo | El runtime de Kotlin |
| Costo de memoria | ~1-2 MB (stack) | Bytes (un objeto) |
| Cuántas caben | Miles | Cientos de miles |
| Al esperar | Se bloquea (ocupa el hilo) | Suspende (libera el hilo) |
| Cancelación | Manual, complicada | Cooperativa, vía scope |

**Ver también:** [11 — Threads vs Coroutines](11-threads-vs-coroutines.md) para el detalle completo de este contraste.

---

## 9. `suspend`: el corazón del código asíncrono

Una función `suspend` **puede pausarse y reanudarse**. Solo se llama desde otra `suspend` o desde una corrutina.

```kotlin
suspend fun fetchUser(): User {
    delay(1000)              // punto de suspensión: aquí puede pausarse
    return User("Ana")
}
```

**El punto que más se confunde:** `suspend` **no significa "corre en otro hilo"**. Significa "esta función puede suspenderse en algún punto". Dónde corre lo decide el dispatcher, no la palabra `suspend`. Una `suspend fun` puede correr en el hilo principal — lo que hace es no *bloquearlo* cuando se suspende.

```kotlin
// Esto NO corre en background solo por ser suspend
suspend fun wrong() {
    val data = heavyBlockingCall()  // bloquea el hilo actual (¡quizá Main!)
}
// Necesita withContext explícito para cambiar de hilo
suspend fun right() = withContext(Dispatchers.IO) {
    heavyBlockingCall()
}
```

---

## 10. Código síncrono, secuencial y asíncrono — los tres términos separados

Estos tres términos se confunden entre sí. No son sinónimos:

| Término | Qué describe | Ejemplo |
|---|---|---|
| **Síncrono** | El hilo se **bloquea** mientras espera | `Thread.sleep()`, llamada de red bloqueante |
| **Secuencial** | El **orden**: una tarea después de otra | `val a = call1(); val b = call2()` (con o sin suspend) |
| **Asíncrono** | El hilo **no se bloquea** mientras espera | `delay()`, `suspend fun` con `withContext` |

Se puede tener código **secuencial pero no síncrono** — exactamente lo que logran las corrutinas:

```kotlin
suspend fun cargar() {
    val user = api.getUser()      // secuencial (se espera el resultado)...
    val posts = api.getPosts()    // ...pero NO síncrono, porque suspende sin bloquear
}
```

Esto es secuencial (una llamada tras otra, en orden) pero **no** es síncrono (no bloquea el hilo). Por eso el hilo principal no se congela aunque el código se lea "de arriba a abajo, uno tras otro".

**Contexto histórico:** antes de corrutinas, para evitar código síncrono bloqueando la UI se usaban **callbacks** — asíncronos, pero rompían la legibilidad secuencial ("callback hell"). Las corrutinas juntan lo mejor de ambos mundos: se lee secuencial, es asíncrono por debajo.

---

## 11. Concurrencia vs paralelismo vs secuencial en código

### Concurrencia: tareas independientes de espera (I/O)

```kotlin
suspend fun concurrente() = coroutineScope {
    val user = async { api.getUser() }    // lanza, NO espera
    val posts = async { api.getPosts() }  // lanza, NO espera (user sigue corriendo)
    user.await(); posts.await()           // ahora espera ambas
    // total: ~1s (las dos esperas se solapan). Sin Dispatchers.Default.
}
```

### Paralelismo: trabajo de CPU en varios cores

```kotlin
suspend fun processImages(imgs: List<Bitmap>) = coroutineScope {
    val results = imgs.map { img ->
        async(Dispatchers.Default) {   // AQUÍ sí Default: es cálculo de CPU
            applyFilter(img)            // quema CPU → paralelismo real en cores
        }
    }
    results.awaitAll()
}
```

### Secuencial: una tras otra

```kotlin
// Por dependencia (obligatorio):
val user = api.getUser()
val posts = api.getPosts(user.id)   // necesita user.id

// Por error (await enseguida): esto DESTRUYE la concurrencia
val user = async { api.getUser() }.await()   // espera acá
val posts = async { api.getPosts() }.await() // recién ahora arranca → ~2s
```

### Resumen de los casos

| Caso | Tipo de tarea | ¿Dependen? | Dispatcher | Tiempo (2 tareas de 1s) | Qué es |
|---|---|---|---|---|---|
| `async` red + `await` al final | Espera (I/O) | No | `IO` / suspend | ~1s | **Concurrencia** |
| `async(Default)` cálculo | CPU | No | `Default` | ~1s (en cores) | **Paralelismo** |
| Llamadas con dependencia | Cualquiera | Sí | el que sea | ~2s | **Secuencial** |
| `async{}.await()` enseguida | Cualquiera | No | el que sea | ~2s | Secuencial (por error) |

---

## 12. El rol de `await` (se malinterpreta)

`await` **no crea** concurrencia ni paralelismo. Solo **recoge el resultado** de un `async`. Lo que define si hay concurrencia es **cuándo** se llama:

```kotlin
// await al final → CONCURRENTE (lanza todo, luego recoge)
val a = async { tarea1() }
val b = async { tarea2() }
a.await(); b.await()          // ~1s

// await enseguida → SECUENCIAL (recoge antes de lanzar la otra)
val a = async { tarea1() }.await()   // espera acá
val b = async { tarea2() }.await()   // recién ahora arranca → ~2s
```

> Mismo `await`, resultado opuesto. El **orden lanzar-vs-esperar** es lo que manda, no el `await` en sí. Concurrencia = lanzar todo primero, esperar después.

**`async` no equivale a paralelismo.** `async` sirve para lanzar varias tareas solapadas y recoger sus resultados. Que eso sea concurrencia o paralelismo lo decide el contenido y el dispatcher, no `async`.

- `async { api.getX() }` (red) → **concurrencia de I/O** (funciona en 1 core).
- `async(Dispatchers.Default) { calc() }` (CPU) → **paralelismo real** (varios cores).

**Combinar ≠ depender.**
- **Combinar** = juntar resultados de varias tareas en algo (un objeto de pantalla). Esto es lo que hace `async`.
- **Depender** = una tarea necesita el resultado de otra para ejecutarse. Esto fuerza lo secuencial.

El dashboard clásico **combina** tres resultados **sin dependencia** entre ellos → por eso puede usar `async`. Si hubiera dependencia, no podría.

---

## 13. `launch` vs `async` — cuál para el caso real (pantalla con secciones independientes)

Caso real (ej. pantalla de cuenta bancaria con balance, perfil y transacciones, cada uno con su propio estado):

**Un solo `launch` con llamadas directas → SECUENCIAL**, aunque "estén separadas" en funciones:

```kotlin
fun loadAccount() {
    viewModelScope.launch {
        val balance = repo.getBalance()      // espera 1s
        _balanceState.value = balance
        val profile = repo.getProfile()      // recién ahora, 1s
        _profileState.value = profile
        // total: 2s+ — SECUENCIAL
    }
}
```

**Tres `launch` separados → CONCURRENTE**, y es el patrón correcto cuando cada uno actualiza su propio estado:

```kotlin
fun loadAccount() {
    viewModelScope.launch { _balance.value = repo.getBalance() }
    viewModelScope.launch { _profile.value = repo.getProfile() }
    viewModelScope.launch { _txns.value = repo.getTransactions() }
    // tres corrutinas concurrentes, cada una actualiza su estado al llegar
}
```

**`async` + `await` → CONCURRENTE**, pero conviene cuando se necesita combinar los resultados en un solo estado:

```kotlin
viewModelScope.launch {
    val balance = async { repo.getBalance() }
    val profile = async { repo.getProfile() }
    _state.value = AccountState(balance.await(), profile.await())
}
```

**Regla de decisión:**

| Situación | Opción |
|---|---|
| Cada sección tiene su propio estado, aparece cuando llega | Varios `launch` |
| Se necesita combinar todo en un solo estado final | `async` + `await` |
| `launch` no devuelve valor (Job) — si se necesita el resultado, no sirve | usar `async` |

**Nota de structured concurrency:** con varios `launch` en `viewModelScope` (que usa `SupervisorJob`), si uno falla, los otros **siguen** — cada uno necesita su propio try/catch. Ver [06 — Scopes y Structured Concurrency](06-concurrencia-scopes.md).

---

## 14. Errores comunes

1. **Creer que `suspend` cambia de hilo.** No lo hace; solo marca que puede suspenderse. Para cambiar de hilo, `withContext`.
2. **Bloquear dentro de una corrutina.** `Thread.sleep()` o API bloqueante anula la ventaja. Usar `delay` y suspend functions.
3. **`runBlocking` en producción Android.** Congela la UI. Usar `viewModelScope` / `lifecycleScope`.
4. **Creer que corrutinas = paralelismo automático.** Sin varios hilos/cores, dos tareas de CPU corren en serie.
5. **`async` + `await()` inmediato para una sola tarea.** Es maquinaria de más → usar `withContext`.
6. **`async` para tareas dependientes.** Si la segunda necesita el resultado de la primera, no hay paralelismo posible → es secuencial, escribirlo secuencial y claro.
7. **Confundir "varias llamadas de red" con paralelismo.** Es concurrencia de I/O, no paralelismo (no usa `Default`, funcionaría en un core).
8. **Creer que secuencial = síncrono.** Son conceptos distintos — ver sección 10.
9. **Creer que separar en funciones suspend da concurrencia.** No la da — ver sección 5.
10. **Un solo `launch` con llamadas directas pensando que es concurrente.** No lo es — ver sección 13.

---

## 15. Frases para entrevista

**Conceptos:**
- *"Concurrencia es lidiar con muchas cosas; paralelismo es hacerlas al mismo tiempo."*
- *"Se puede tener concurrencia sin paralelismo: un core alternando tareas."*
- *"Solapar esperas es concurrencia; solapar cómputo en cores es paralelismo."*
- *"Secuencial = espero-luego-lanzo; concurrente = lanzo-todo-luego-espero."*
- *"La dependencia entre tareas es lo que fuerza lo secuencial — es lo opuesto a la concurrencia."*
- *"Síncrono bloquea el hilo mientras espera; asíncrono no. Secuencial es el orden, no el bloqueo."*

**Corrutinas / suspend:**
- *"Suspender libera el hilo; bloquear lo ocupa sin trabajar."*
- *"`suspend` no cambia de hilo — solo marca que la función puede pausarse."*
- *"Miles de corrutinas sobre pocos hilos porque las suspendidas no ocupan hilo."*

**async y await:**
- *"El valor de `async` es paralelizar tareas independientes, no lanzar una sola."*
- *"La excepción de `async` estalla en el `await`, no al lanzarlo."*
- *"`await` no crea concurrencia — el orden lanzar-vs-esperar es lo que manda."*

---

## 16. Trampas típicas de entrevista

**"¿Las corrutinas son paralelismo o concurrencia?"**
Modelo de **concurrencia**; dan **paralelismo solo si** el dispatcher las reparte en varios cores (`Dispatchers.Default`). Una corrutina sola no es paralela; muchas sobre un solo hilo son concurrentes pero no paralelas.

**"¿Una corrutina es más rápida que un hilo?"**
No en cómputo: el trabajo de CPU tarda lo mismo. Se gana **concurrencia y uso eficiente de recursos**: más espera concurrente con menos memoria y menos context switches.

**"¿Varias llamadas de red con `async` son paralelismo?"**
No — son **concurrencia de I/O**. Las llamadas esperan a la vez, no ejecutan código a la vez. Funcionarían en un solo core, así que no es paralelismo.

**"¿Código secuencial es lo mismo que código síncrono?"**
No. Secuencial es el orden; síncrono es si el hilo se bloquea. Con `suspend`, el código es secuencial (se lee en orden) pero no síncrono (no bloquea, suspende).

**"¿Separar llamadas en funciones `suspend` distintas las hace concurrentes?"**
No. Una función es solo un envoltorio de código. Lo que decide el tiempo es cómo se llaman (directo = secuencial; con `async` = concurrente), no dónde viven.

**"Tres llamadas independientes en un ViewModel, ¿un launch o varios?"**
Depende de si se necesitan combinar en un estado o si cada una actualiza el suyo. Un solo `launch` con llamadas directas = secuencial (error común). Varios `launch` = concurrente, cada uno independiente. `async`+`await` = concurrente, combinando resultados.
