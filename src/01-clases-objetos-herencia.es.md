# Classes, Objects & Inheritance / Clases, objetos y herencia

[English version](01-clases-objetos-herencia.en.md)

---

## 1. `class` — la base

Una `class` es una plantilla para crear objetos. **Encapsula propiedades (estado) y funciones (comportamiento)** en una sola unidad: los datos que un objeto necesita cargar y las acciones que puede realizar viven juntos, en vez de estar sueltos por el código.

Por defecto en Kotlin es **`final`**: no se puede heredar de ella salvo que la marques `open`.

```kotlin
class User(val name: String, val age: Int) {   // propiedades: estado del objeto
    fun isAdult() = age >= 18                   // función: comportamiento del objeto
}

val u = User("Ana", 30)
```

**Por qué `final` por defecto (y no `open`):** Kotlin invierte la filosofía de Java a propósito. Heredar de una clase que no fue diseñada para eso rompe encapsulamiento (el "fragile base class problem" — cambiás la clase base y rompés silenciosamente a las hijas). Por eso heredar es un opt-in explícito con `open`, no el default.

---

## 2. Herencia — definición base

**Herencia** es un **mecanismo de la Programación Orientada a Objetos (POO)** que permite **reutilizar propiedades y comportamientos de otras clases**. Concretamente: una clase (la **subclase** o clase hija) adquiere las propiedades y funciones de otra (la **superclase** o clase base), pudiendo además agregar las suyas propias o modificar el comportamiento heredado.

Resuelve un problema concreto: **reutilizar código** entre clases que comparten características, sin copiar y pegar. Si `Dog` y `Cat` son ambos `Animal`, no repetís `name`, `age` en cada uno — los heredás de una base común.

```kotlin
open class Animal(val name: String, val age: Int)   // superclase / clase base

class Dog(name: String, age: Int) : Animal(name, age)  // subclase / clase hija
// Dog HEREDA name y age de Animal, sin redeclararlos
```

La relación que crea la herencia se describe como **"es un"** (*is-a*): un `Dog` **es un** `Animal`. Esa relación es justo lo que permite el **polimorfismo**: tratar un `Dog` como si fuera un `Animal` en cualquier lugar que espere un `Animal`.

```kotlin
fun describe(animal: Animal) = println(animal.name)
describe(Dog("Rex", 3))   // un Dog se puede pasar donde se espera un Animal
```

---

## 3. Cómo se habilita en Kotlin (`open`, `override`)

Para permitir herencia, la clase y los miembros que se puedan sobreescribir deben ser `open` (recordá: en Kotlin todo es `final` por defecto — ver sección 1).

```kotlin
open class Animal(val name: String) {
    open fun makeSound() = "..."
}

class Dog(name: String) : Animal(name) {
    override fun makeSound() = "Woof"
}
```

- `open class` → se puede heredar.
- `open fun` → se puede sobreescribir en la hija.
- `override fun` → obligatorio para sobreescribir (evita overrides accidentales, a diferencia de Java donde `@Override` es opcional).

**Kotlin no tiene herencia múltiple de clases** (una clase solo puede extender una clase base), pero sí puede implementar **varias interfaces**. Eso es la razón práctica por la que Kotlin empuja hacia composición + interfaces en vez de jerarquías de clases profundas.

```kotlin
class Dog(name: String) : Animal(name), Runnable, Comparable<Dog> {
    // una clase base, N interfaces
}
```

---

## 4. `abstract class`

Una **clase base que sirve para otras clases**: se usa cuando **varias clases comparten la misma estructura y datos en común**, y no se puede instanciar directamente — existe para ser heredada. Puede mezclar miembros **abstractos** (sin implementación, obligatorios de sobreescribir) con miembros **concretos** (con implementación, compartidos por todas las hijas).

```kotlin
abstract class Shape {
    abstract fun area(): Double        // sin cuerpo, obligatorio implementar
    fun describe() = "Área: ${area()}" // con cuerpo, compartido por todas
}

class Circle(val radius: Double) : Shape() {
    override fun area() = Math.PI * radius * radius
}
```

**Por qué existe si ya hay interfaces:** una `abstract class` puede tener **estado** (propiedades con valor, constructor) y lógica **concreta** compartida real. Una interfaz (antes de Kotlin 1.x con default methods) no podía. Hoy las interfaces también permiten cuerpos de función por defecto, así que la línea se angostó — pero la abstract class sigue siendo la opción cuando necesitás **estado compartido** en la base.

---

## 5. `interface`

Un contrato: define qué métodos/propiedades debe tener algo, sin decir (necesariamente) cómo. Las interfaces en Kotlin **sí pueden tener implementación por defecto**:

```kotlin
interface Clickable {
    fun onClick()                          // sin implementación: obligatorio
    fun onLongClick() { println("long") }  // CON implementación por defecto
}

class Button : Clickable {
    override fun onClick() { println("click") }
    // onLongClick no es obligatorio sobreescribirlo, ya tiene default
}
```

Una interfaz **no puede tener estado propio con backing field** (no podés guardar un valor mutable real ahí), aunque sí puede declarar propiedades abstractas que la clase implementadora debe proveer.

### `class` vs `abstract class` vs `interface` — cuándo cada una

| | Instanciable | Estado propio | Herencia múltiple | Uso típico |
|---|---|---|---|---|
| `class` | Sí | Sí | — | Objeto concreto normal |
| `abstract class` | No | Sí (con constructor) | No (una sola) | Base con estado + lógica compartida |
| `interface` | No | No (solo abstractas) | Sí (varias) | Contrato / capacidad (Comparable, Clickable) |

**La pregunta que decide:** ¿necesito que la base cargue **estado** (un constructor con propiedades reales)? → `abstract class`. ¿Solo necesito definir un **contrato** que varias clases no relacionadas puedan cumplir? → `interface`.

---

## 6. `data class`

Clase pensada para **contener datos**. El compilador genera automáticamente, a partir de las propiedades del **constructor primario**:

- `equals()` / `hashCode()` — igualdad estructural (compara contenido, no referencia).
- `toString()` — legible: `User(id=1, name=Ana)`.
- `copy()` — copia cambiando solo lo que indiques.
- `componentN()` — para destructuring (`val (id, name) = user`).

```kotlin
data class User(val id: String, val name: String, val age: Int)

val u1 = User("1", "Ana", 30)
val u2 = u1.copy(age = 31)       // copia cambiando solo age
val (id, name, age) = u1         // destructuring
```

**Detalles que sorprenden:**
- Todo lo generado sale **solo del constructor primario**. Una propiedad declarada en el cuerpo de la clase se ignora en `equals`/`hashCode`/`copy`/`toString`.
- `copy()` es **copia superficial** (shallow): si una propiedad es un objeto mutable, la copia comparte la misma referencia.
- Restricciones: constructor con al menos un parámetro, no puede ser `open`/`abstract`/`sealed`/`inner`.
- Si sobreescribís vos `equals`/`hashCode`/`toString`, el compilador respeta el tuyo.

---

## 7. `object` — singleton nativo

Declara una clase con **una sola instancia**, creada de forma lazy y thread-safe automáticamente por el compilador. No hay constructor (no podés instanciarlo con `()`).

```kotlin
object NetworkManager {
    var isConnected = false
    fun connect() { /* ... */ }
}

NetworkManager.connect()   // se accede directo, sin instanciar
```

Es el reemplazo de Kotlin para el patrón Singleton manual de Java (con `getInstance()` y doble-check locking) — el lenguaje te lo da gratis y sin boilerplate.

También existe **object expression** (instancia anónima de una interfaz/clase, como una clase anónima de Java):

```kotlin
val listener = object : View.OnClickListener {
    override fun onClick(v: View) { /* ... */ }
}
```

---

## 8. `companion object`

Un `object` **anidado dentro de una clase**, ligado a esa clase (no a sus instancias) — es el reemplazo de `static` de Java.

```kotlin
class User private constructor(val id: String) {
    companion object {
        fun create(id: String): User = User(id)   // "factory method"
        const val MAX_AGE = 150
    }
}

val u = User.create("123")   // se llama sobre la CLASE, no una instancia
```

Usos típicos: factory methods, constantes ligadas a la clase, implementar una interfaz "estática" (un companion puede implementar una interfaz).

---

## 9. `data object`

Combinación de `object` + los beneficios de `data class`, pero sin datos (porque un `object` no tiene constructor con propiedades). Da automáticamente un `toString()` legible y `equals`/`hashCode` consistentes — útil sobre todo para estados sin datos en una `sealed class/interface`.

```kotlin
sealed interface UiState {
    data object Loading : UiState      // toString → "Loading", no "Loading@3f2a"
    data class Success(val data: String) : UiState
}
```

Sin `data`, un `object` plano igual funciona en un `when`, pero su `toString()` por defecto es feo (`Loading@hashcode`) — malo para logs/debugging.

### `data object` vs `companion object` — la diferencia que se confunde

Son cosas completamente distintas que solo comparten la palabra `object`:

| | `companion object` | `data object` |
|---|---|---|
| Qué es | Un `object` **ligado a una clase** (reemplaza `static`) | Un **modificador** sobre `object` que agrega `toString`/`equals` legibles |
| Dónde vive | Anidado dentro de una clase | Standalone, o como caso de una sealed |
| Para qué | Factory methods, constantes de clase | Representar un caso sin datos (ideal en sealed) |
| Se puede combinar | `companion data object { ... }` también es válido | — |

No son alternativas una de la otra — de hecho podés tener un `companion data object` si necesitás ambas cosas a la vez.

---

## 10. `enum class`

Conjunto **fijo y cerrado de instancias únicas**, todas con la misma forma (estructura).

```kotlin
enum class Status(val label: String) {
    LOADING("Cargando"),
    SUCCESS("Listo"),
    ERROR("Error");

    fun isTerminal() = this != LOADING
}

Status.values()       // array con todos los casos
Status.valueOf("LOADING")
```

Da gratis: `values()`, `valueOf()`, `.ordinal`, `.name`, y es iterable. Pero **todas las instancias comparten la misma estructura** — no podés tener un caso que cargue datos distintos a otro.

---

## 11. `sealed class` / `sealed interface`

Jerarquía **cerrada**: todas las implementaciones deben vivir en el mismo módulo/paquete, así que el compilador conoce **todo el conjunto** y puede verificar que un `when` sea **exhaustivo sin `else`**.

```kotlin
sealed interface Result
data class Success(val data: String) : Result
data class Error(val message: String) : Result
data object Loading : Result

fun render(result: Result) = when (result) {
    is Success -> showData(result.data)
    is Error -> showError(result.message)
    Loading -> showSpinner()
    // sin else: si agrego un caso nuevo y olvido manejarlo, ERROR de compilación
}
```

Con una `interface` normal, el `when` te obliga a `else` (el compilador no puede garantizar que conoce todos los casos) — y ese `else` es donde se esconden los bugs cuando agregás un caso nuevo y olvidás manejarlo.

### `sealed class` vs `sealed interface`

`sealed` es un **modificador** que cierra la jerarquía; funciona igual sobre `class` o `interface`. La diferencia es la de siempre entre class e interface:

- **sealed class**: puede tener **estado/constructor** en la raíz, compartido por todos los casos. Herencia única.
- **sealed interface**: sin estado propio, pero permite que un caso **implemente varias interfaces** (más flexible).

```kotlin
// sealed class: cuando TODOS los casos comparten una propiedad
sealed class ScreenState(val isRefreshing: Boolean) {
    data object Loading : ScreenState(isRefreshing = false)
    data class Success(val data: Data) : ScreenState(isRefreshing = false)
}

// sealed interface: casos independientes, sin nada en común
sealed interface UiState {
    data object Loading : UiState
    data class Success(val data: Data) : UiState
}
```

**Regla práctica:** `sealed interface` por defecto (más flexible, no arrastra estado innecesario); `sealed class` solo cuando **todos** los casos comparten una propiedad o lógica común que no querés repetir en cada uno.

### `sealed` vs `enum` — cuándo cada uno

| | `enum class` | `sealed class/interface` |
|---|---|---|
| Instancias | Únicas, fijas | Puede haber **múltiples** instancias de un mismo caso |
| Estructura | Igual en todos los casos | Cada caso puede tener **datos propios distintos** |
| `values()`/`ordinal` | Sí, gratis | No |
| Uso típico | Días de la semana, opciones de un filtro | Estado de UI (`Success(data)`, `Error(msg)`) |

**La pregunta que decide:** ¿cada caso necesita cargar **datos distintos**? → `sealed`. ¿Son solo constantes intercambiables sin datos propios? → `enum`.

---

## Resumen — tabla de decisión rápida

| Necesito... | Uso |
|---|---|
| Contenedor de datos simple | `data class` |
| Una sola instancia global | `object` |
| Algo "estático" ligado a una clase | `companion object` |
| Caso sin datos dentro de una sealed | `data object` |
| Conjunto cerrado de constantes homogéneas | `enum class` |
| Conjunto cerrado donde cada caso lleva datos distintos | `sealed class` / `sealed interface` |
| Contrato que implementan clases no relacionadas | `interface` |
| Base con estado + lógica compartida, sin instanciar directo | `abstract class` |
| Reutilizar propiedades/comportamiento de otra clase | Herencia (`open`/`override`) |

---

## Frases para entrevista

- *"Una clase encapsula propiedades y comportamiento — estado y funciones — en una sola unidad."*
- *"Herencia es un mecanismo de POO para reutilizar propiedades y comportamientos de otras clases; crea una relación 'es un' que habilita polimorfismo."*
- *"Kotlin no permite herencia múltiple de clases, pero sí de interfaces — por eso favorece composición sobre jerarquías profundas."*
- *"abstract class cuando necesito estado compartido en la base; interface cuando solo defino un contrato."*
- *"data class genera equals/hashCode/copy solo del constructor primario — una propiedad en el cuerpo se ignora."*
- *"companion object reemplaza static; data object es un modificador para toString/equals legibles en objetos sin datos — no son lo mismo."*
- *"sealed da exhaustividad sin else: agregar un caso nuevo rompe la compilación en vez de esconderse en un else."*
- *"sealed interface por defecto; sealed class solo si todos los casos comparten estado."*

## Trampas típicas de entrevista

**"¿Por qué class es `final` por defecto en Kotlin?"** → Para evitar el fragile base class problem: heredar de algo no diseñado para eso rompe encapsulamiento. Herencia es opt-in con `open`.

**"Te muestro una data class con una `var` declarada en el cuerpo, ¿son iguales dos instancias que difieren solo ahí?"** → Sí, `equals` solo mira el constructor primario.

**"¿Cuándo `sealed class` en vez de `sealed interface`?"** → Solo cuando todos los casos comparten una propiedad o lógica común en la raíz; si no, interface es más flexible y evita boilerplate repetido.
