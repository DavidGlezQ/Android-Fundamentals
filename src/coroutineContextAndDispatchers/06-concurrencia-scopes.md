# Scopes & Structured Concurrency / Scopes y Structured Concurrency

[English version](06-concurrencia-scopes.en.md)

---

## 1. El problema que resuelve

Corrutinas sin estructura: se lanza una, se olvida, sigue corriendo para siempre aunque ya no se necesite. El usuario cierra la pantalla pero la corrutina sigue haciendo una llamada de red y luego intenta actualizar una UI que ya no existe → crash o fuga de memoria.

**Structured concurrency** es la regla de que **toda corrutina vive dentro de un scope**, y ese scope forma una **jerarquía padre-hijo** con reglas claras: si el padre se cancela, todos los hijos se cancelan; el padre no termina hasta que todos sus hijos terminen. Nada queda huérfano.

---

## 2. CoroutineScope: el contenedor

Un `CoroutineScope` es el contexto donde viven las corrutinas. Todo builder (`launch`, `async`) es una **función de extensión de `CoroutineScope`** — por eso se necesita un scope para llamarlos.

```kotlin
class MyViewModel : ViewModel() {
    fun doWork() {
        viewModelScope.launch {   // viewModelScope ES un CoroutineScope
            // vive dentro del scope del ViewModel
        }
    }
}
```

Scopes que ya se usan en Android:
- **`viewModelScope`** → vive mientras vive el ViewModel; se cancela solo en `onCleared`.
- **`lifecycleScope`** → atado al lifecycle de una Activity/Fragment.

Cuando el ViewModel muere, `viewModelScope` **cancela automáticamente** todas las corrutinas lanzadas en él. Por eso no hay fugas: ninguna corrutina sobrevive a su pantalla.

---

## 3. La jerarquía padre-hijo

Cuando se lanza una corrutina dentro de un scope, se crea una relación padre-hijo. Se forma un **árbol de Jobs**.

Las tres reglas de oro:

1. **El padre espera a todos sus hijos.** No se considera terminado hasta que todas sus hijas terminaron.
2. **Cancelar el padre cancela a todos los hijos.**
3. **Si un hijo falla, cancela al padre y a los hermanos** (con `Job` normal — con `SupervisorJob` cambia, ver sección 6).

```kotlin
coroutineScope {                        // padre
    launch { taskA() }                  // hijo 1
    launch { taskB() }                  // hijo 2
    // este bloque NO termina hasta que taskA Y taskB terminen (regla 1)
}
```

---

## 4. `coroutineScope { }`: agrupar y esperar

Builder suspend que crea un scope hijo y **suspende hasta que todo lo de adentro termine**. Dice "haz todas estas cosas concurrentemente y no sigas hasta que todas terminen".

```kotlin
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val user = async { api.getUser() }
    val posts = async { api.getPosts() }
    Dashboard(user.await(), posts.await())
    // coroutineScope no retorna hasta que ambos async terminen
}
```

¿Por qué usarlo en vez de lanzar sueltos? Porque **agrupa**: si `getUser()` falla, `coroutineScope` cancela `getPosts()` automáticamente y propaga la excepción. Todo el grupo se trata como una unidad.

---

## 5. Caso real: tres `launch` hermanos vs `coroutineScope`

```kotlin
// TRES launch en viewModelScope: son HERMANOS bajo el mismo padre
viewModelScope.launch { repo.getBalance() }
viewModelScope.launch { repo.getProfile() }
viewModelScope.launch { repo.getTransactions() }
```

Con un `viewModelScope` normal (usa `SupervisorJob` por dentro), si uno falla, **los otros siguen**. Pero si se hiciera lo mismo dentro de un `coroutineScope { }` normal, un fallo **sí** cancelaría a los hermanos. Esa diferencia es `Job` vs `SupervisorJob`.

---

## 6. `Job` vs `SupervisorJob`: propagación de fallos

**`Job` normal (`coroutineScope`):** el fallo de un hijo se propaga hacia arriba y cancela a todos. "Si uno cae, caen todos."

```kotlin
coroutineScope {
    launch { throw Exception("boom") }  // falla
    launch { delay(1000); print("no llego") } // ← se cancela por el hermano
}
```

**`SupervisorJob` (`supervisorScope`):** el fallo de un hijo **no** afecta a los hermanos. "Si uno cae, los demás siguen."

```kotlin
supervisorScope {
    launch { throw Exception("boom") }  // falla
    launch { delay(1000); print("sí llego") } // ← sobrevive
}
```

Cuándo cada uno:
- **`coroutineScope`** → tareas que son **un todo**: si una falla, las demás no tienen sentido (ej. se necesitan las tres para armar una pantalla única).
- **`supervisorScope`** → tareas **independientes**: el fallo de una no invalida las otras (ej. pantalla con secciones separadas).

`viewModelScope` usa `SupervisorJob` por dentro, por eso varios `launch` hermanos no se tumban entre sí.

---

## 7. Ejemplo real de Android

**Caso "todo o nada" — coroutineScope:**

```kotlin
fun loadProfile() {
    viewModelScope.launch {
        _state.value = Loading
        try {
            val screen = coroutineScope {   // agrupa: si una falla, cancela la otra
                val user = async { repo.getUser() }
                val settings = async { repo.getSettings() }
                ProfileScreen(user.await(), settings.await())
            }
            _state.value = Success(screen)   // solo si AMBAS salieron bien
        } catch (e: Exception) {
            _state.value = Error(e)          // si cualquiera falló, un solo error
        }
    }
}
```

**Caso "independiente" — cada sección su estado (ej. pantalla de banco):**

```kotlin
fun loadAccount() {
    // Cada launch hermano en viewModelScope (SupervisorJob): independientes
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

## 8. Errores comunes

**a) Usar `GlobalScope`.** Crea una corrutina que **no pertenece a ningún scope estructurado** — vive para siempre, no se cancela con la pantalla, filtra memoria. Antipatrón #1.

```kotlin
// MAL: sobrevive a la pantalla, fuga asegurada
GlobalScope.launch { repo.getData() }
// BIEN: atado al ciclo de vida
viewModelScope.launch { repo.getData() }
```

**b) Crear un scope propio y no cancelarlo.** Si se crea un `CoroutineScope(...)` propio, hay que ser responsable de cancelarlo. En Android conviene preferir los scopes provistos.

**c) Esperar que un `coroutineScope` sobreviva a un fallo.** Si una tarea puede fallar sin invalidar a las otras, `coroutineScope` cancelará todo — se necesita `supervisorScope`.

**d) Lanzar en un scope equivocado.** Trabajo de larga duración en `lifecycleScope` (muere al rotar) cuando debería estar en `viewModelScope` (sobrevive).

---

## 9. Frases para entrevista

- *"Toda corrutina vive en un scope — no hay huérfanas."*
- *"El padre espera a los hijos; cancelar el padre cancela el árbol."*
- *"coroutineScope: si uno falla, caen todos. supervisorScope: cada uno vive su vida."*
- *"GlobalScope es el antipatrón: no se cancela con la pantalla, filtra memoria."*
- *"viewModelScope se cancela solo en onCleared — por eso no hay fugas."*

---

## 10. Trampas típicas de entrevista

**"¿Qué es structured concurrency?"**
Toda corrutina vive en un scope y forma jerarquía padre-hijo: el padre no termina hasta que sus hijos terminan, cancelar el padre cancela a los hijos, y un fallo se propaga según el tipo de Job. El beneficio: no hay corrutinas huérfanas.

**"¿Diferencia entre coroutineScope y supervisorScope?"**
Ambos crean un scope hijo, pero difieren en propagación de fallos. `coroutineScope`: si un hijo falla, se cancelan todos — "todo o nada". `supervisorScope`: el fallo de un hijo no afecta a los hermanos.

**"Tres llamadas independientes en un ViewModel, una falla, ¿las otras siguen?"**
Depende del scope. Con tres `launch` en `viewModelScope` (SupervisorJob), sí siguen. Dentro de un `coroutineScope { }`, no — el fallo cancela a las hermanas.
