# Coroutine Testing / Testing de corrutinas

[English version](12-testing-corrutinas.en.md)

---

## 1. Por que testear corrutinas es distinto

Un test normal corre de forma sincrona. Las corrutinas suspenden, usan `delay`, y dependen de un dispatcher - si se testea con el dispatcher real (`Dispatchers.Main`, que no existe en una JVM de test), el test crashea o queda esperando tiempos reales (un `delay(5000)` haria que el test tarde 5 segundos de verdad).

`kotlinx-coroutines-test` resuelve esto con **dispatchers de prueba** que controlan el tiempo de forma virtual.

---

## 2. `runTest`: el reemplazo de `runBlocking` para tests

`runTest` crea un scope de corrutina especial donde el tiempo es **virtual**: los `delay` no esperan de verdad, se saltan instantaneamente, pero el orden logico de ejecucion se respeta.

```kotlin
@Test
fun `cuando cargo el usuario, el estado pasa a Success`() = runTest {
    val viewModel = UserViewModel(fakeRepo)
    viewModel.loadUser()
    advanceUntilIdle()   // avanza el tiempo virtual hasta que no quede trabajo pendiente
    assertEquals(UiState.Success(fakeUser), viewModel.state.value)
}
```

`runTest` tambien detecta corrutinas que quedaron colgadas (nunca terminaron) y falla el test explicitamente, en vez de dejarlo colgado para siempre como pasaria con `runBlocking`.

**Control del tiempo virtual:**
- `advanceUntilIdle()` - avanza hasta que no haya mas trabajo pendiente en la cola.
- `advanceTimeBy(ms)` - avanza una cantidad especifica de tiempo virtual.
- `runCurrent()` - ejecuta las tareas ya programadas para el tiempo actual, sin avanzar el reloj.

---

## 3. `MainDispatcherRule`: reemplazar `Dispatchers.Main` en tests

El codigo de produccion suele usar `viewModelScope`, que internamente usa `Dispatchers.Main`. En una JVM de test (sin Android real) `Dispatchers.Main` no esta inicializado y lanza una excepcion. La solucion es una JUnit Rule que lo reemplaza por un dispatcher de test antes de cada test:

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class MainDispatcherRule(
    private val testDispatcher: TestDispatcher = StandardTestDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}
```

Uso en el test:

```kotlin
class UserViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @Test
    fun `test con viewModelScope funciona correctamente`() = runTest {
        // viewModelScope.launch ahora usa el dispatcher de test, no el real
    }
}
```

**Por que es una Rule y no codigo suelto:** garantiza que `Dispatchers.Main` se resetea despues de cada test (en `finished`), evitando que un test deje el dispatcher de prueba instalado y rompa los tests siguientes.

---

## 4. `StandardTestDispatcher` vs `UnconfinedTestDispatcher`

Dos formas de controlar como se ejecutan las corrutinas en el test:

**`StandardTestDispatcher`**: las corrutinas lanzadas **no corren inmediatamente** - quedan en cola hasta que se llama explicitamente `advanceUntilIdle()` o similar. Da control preciso sobre el orden, util para verificar estados intermedios (ej. que el estado pase por `Loading` antes de `Success`).

**`UnconfinedTestDispatcher`**: las corrutinas corren **inmediatamente**, de forma eager, sin necesidad de avanzar el tiempo manualmente. Mas simple pero se pierde la posibilidad de verificar estados intermedios facilmente.

```kotlin
@Test
fun `con StandardTestDispatcher puedo verificar el estado Loading`() = runTest {
    val viewModel = UserViewModel(fakeRepo)
    viewModel.loadUser()
    assertEquals(UiState.Loading, viewModel.state.value)   // todavia no avanzo el tiempo
    advanceUntilIdle()
    assertEquals(UiState.Success(fakeUser), viewModel.state.value)
}
```

`runTest` usa `StandardTestDispatcher` por defecto.

---

## 5. Testing de Flows con Turbine

Coleccionar un `Flow` en un test a mano (con `toList()` o similar) es incomodo porque hay que saber cuando parar de coleccionar. **Turbine** da una API pensada para esto:

```kotlin
@Test
fun `el flow emite Loading y luego Success`() = runTest {
    viewModel.state.test {
        assertEquals(UiState.Loading, awaitItem())
        viewModel.loadUser()
        assertEquals(UiState.Success(fakeUser), awaitItem())
        cancelAndIgnoreRemainingEvents()   // el flow es caliente, no termina solo
    }
}
```

`awaitItem()` suspende hasta la proxima emision, `awaitComplete()` espera a que el flow termine, `awaitError()` espera una excepcion. Como `StateFlow`/`SharedFlow` son calientes y nunca terminan por si solos, siempre hay que cerrar la coleccion explicitamente con `cancelAndIgnoreRemainingEvents()` o similar al final del bloque `test { }`.

---

## 6. El patron AAA aplicado a corrutinas

**Arrange, Act, Assert** sigue siendo la estructura, solo que el "Act" suele necesitar `advanceUntilIdle()` antes del "Assert":

```kotlin
@Test
fun `al fallar la carga, el estado pasa a Error`() = runTest {
    // Arrange
    val fakeRepo = FakeUserRepository(shouldFail = true)
    val viewModel = UserViewModel(fakeRepo)

    // Act
    viewModel.loadUser()
    advanceUntilIdle()

    // Assert
    assertTrue(viewModel.state.value is UiState.Error)
}
```

---

## 7. JUnit 4 vs JUnit 5 (lo que cambia para corrutinas)

La mecanica de `runTest`/`MainDispatcherRule` es la misma en ambos, pero cambia como se declaran las reglas:

- **JUnit 4**: `@get:Rule val mainDispatcherRule = MainDispatcherRule()` (usa `TestWatcher`, como se mostro arriba).
- **JUnit 5**: no tiene `@Rule` - se usa una `Extension` (`@ExtendWith(MainDispatcherExtension::class)`) que implementa `BeforeEachCallback`/`AfterEachCallback` en vez de `TestWatcher`.

En Android, JUnit 4 sigue siendo mas comun por la integracion con el tooling de Android (Robolectric, AndroidX Test), aunque JUnit 5 es soportado.

---

## 8. Mockear dependencias: dobles de prueba

Para no depender de Retrofit/Room reales en un unit test:

```kotlin
class FakeUserRepository(private val shouldFail: Boolean = false) : UserRepository {
    override suspend fun getUser(): User {
        if (shouldFail) throw IOException("network error")
        return User("1", "Ana")
    }
}
```

Un **fake** (implementacion real simplificada, como arriba) suele preferirse sobre un **mock** (`MockK`/`Mockito`) cuando la logica es simple, porque es mas legible y no depende de configurar expectativas. Para casos mas complejos (verificar que se llamo un metodo N veces, con ciertos argumentos), `MockK` es el estandar en Kotlin.

```kotlin
val repo = mockk<UserRepository>()
coEvery { repo.getUser() } returns fakeUser   // coEvery: version de MockK para suspend fun
```

---

## Resumen - tabla de decision rapida

| Necesito... | Herramienta |
|---|---|
| Correr una corrutina en un test sin esperar tiempo real | `runTest` |
| Que viewModelScope funcione en un test de JVM | `MainDispatcherRule` + `Dispatchers.setMain` |
| Verificar un estado intermedio (ej. Loading) | `StandardTestDispatcher` (control manual del tiempo) |
| Que las corrutinas corran inmediatamente sin avanzar tiempo | `UnconfinedTestDispatcher` |
| Testear emisiones de un Flow/StateFlow | Turbine (`.test { awaitItem() }`) |
| Mockear una suspend fun | `MockK` con `coEvery` |
| Un doble de prueba simple sin libreria de mocking | Fake (implementacion real simplificada) |

---

## Frases para entrevista

- "runTest usa tiempo virtual: los delay se saltan, pero el orden logico se respeta."
- "MainDispatcherRule reemplaza Dispatchers.Main por un TestDispatcher, porque Main no existe en una JVM de test."
- "StandardTestDispatcher da control manual del tiempo para verificar estados intermedios; UnconfinedTestDispatcher corre todo inmediato."
- "Turbine evita coleccionar un Flow a mano - awaitItem suspende hasta la proxima emision."
- "Un StateFlow es caliente y nunca termina solo, por eso hay que cerrar la coleccion explicitamente en el test."

## Trampas tipicas de entrevista

**"Mi test usa viewModelScope.launch y falla con una excepcion sobre Dispatchers.Main, ?por que?"** - Porque Dispatchers.Main no esta inicializado en una JVM de test sin Android real. Se soluciona con MainDispatcherRule, que lo reemplaza por un TestDispatcher.

**"?Por que usar runTest en vez de runBlocking en un test?"** - Porque runTest usa tiempo virtual (los delay no esperan de verdad) y detecta corrutinas colgadas, fallando el test explicitamente en vez de dejarlo esperando para siempre.

**"?Como testeo que un StateFlow emite Loading y luego Success?"** - Con Turbine: `viewModel.state.test { assertEquals(Loading, awaitItem()); ...; assertEquals(Success(x), awaitItem()) }`, cerrando con cancelAndIgnoreRemainingEvents porque el flow es caliente.
