# Types & Modifiers / Tipos y modificadores

[English version](02-tipos-y-modificadores.en.md)

---

## 1. `const val` vs `val`

Ambos crean una propiedad de solo lectura (no reasignable), pero difieren en **cuándo** se fija el valor:

| | `val` | `const val` |
|---|---|---|
| Se evalúa | En runtime | En tiempo de **compilación** |
| Tipos permitidos | Cualquiera | Solo primitivos y `String` |
| Dónde se declara | Donde sea (propiedad, local, constructor) | Top-level o dentro de un `object`/`companion object` |
| Puede depender de una función/cálculo | Sí | No — debe ser un literal conocido en compile-time |

```kotlin
val currentTime = System.currentTimeMillis()   // runtime: cambia cada ejecución
const val MAX_RETRIES = 3                       // compile-time: literal fijo
```

**Por qué existe `const`:** es una optimización — el compilador **inlinea** el valor directamente donde se usa (como en Java `static final`), sin necesidad de acceder a un campo en runtime. Por eso solo acepta tipos que el compilador puede resolver en compilación (no podés poner `const val x = calcularAlgo()`).

**Regla práctica:** `const val` para constantes verdaderas conocidas de antemano (`MAX_RETRIES`, `BASE_URL`); `val` para todo lo demás, incluido cualquier valor que dependa de lógica o de una llamada.

---

## 2. `value class`

Envuelve un tipo (normalmente primitivo) para darle **significado semántico y seguridad de tipos**, sin el costo de crear un objeto real en runtime — el compilador lo "desenvuelve" (inlinea) en la mayoría de los casos.

```kotlin
@JvmInline
value class UserId(val value: String)

fun getUser(id: UserId) { /* ... */ }

getUser(UserId("123"))     // obligado a envolver — no aceptás un String cualquiera
// getUser("123")          // ERROR: String no es UserId
```

**El problema que resuelve:** sin `value class`, un `UserId` sería solo un `String`, y el compilador dejaría pasar cualquier `String` donde se espera un `UserId` — podrías mezclar un `OrderId` con un `UserId` por error y el compilador no se quejaría. `value class` crea un tipo **distinto** en tiempo de compilación, pero en runtime (en la mayoría de los casos) se comporta como el tipo envuelto, sin overhead de un objeto nuevo.

**`value class` vs `typealias`** (ver siguiente sección): esta es la comparación que siempre preguntan.

---

## 3. `typealias`

Un **nombre alternativo** para un tipo que ya existe. No crea un tipo nuevo — es azúcar sintáctico puro, sin costo en runtime.

```kotlin
typealias UserId = String
typealias ClickHandler = (View) -> Unit

fun register(id: UserId, onClick: ClickHandler) { ... }
// equivale EXACTAMENTE a: fun register(id: String, onClick: (View) -> Unit)
```

Sirve para **legibilidad** (nombrar un tipo funcional complejo) y para **acortar** genéricos largos repetidos. Pero **no da seguridad de tipos**: `UserId` sigue siendo intercambiable con cualquier `String`.

```kotlin
val id: UserId = "cualquier string"   // compila perfecto, no hay protección
```

### `typealias` vs `value class` — la diferencia que preguntan

| | `typealias` | `value class` |
|---|---|---|
| Crea tipo nuevo | No, es el mismo tipo | Sí, tipo distinto |
| Type safety | No (intercambiable con el original) | Sí (no mezclable) |
| Costo en runtime | Cero | Cero (se desenvuelve/inlinea) |
| Para qué | Legibilidad, acortar | Distinguir tipos semánticamente |

**Regla:** `typealias` cuando solo querés un nombre más legible; `value class` cuando querés que el compilador **impida mezclar** valores que son técnicamente el mismo tipo primitivo pero semánticamente distintos (`UserId` vs `OrderId`, ambos `String`).

---

## 4. Null safety

El sistema de tipos de Kotlin distingue en **tiempo de compilación** entre tipos que pueden ser `null` y los que no, para eliminar el `NullPointerException` como clase de bug.

```kotlin
var name: String = "Ana"       // NO puede ser null — el compilador lo garantiza
var nickname: String? = null   // SÍ puede ser null — el `?` lo marca explícito

name = null        // ERROR de compilación, ni siquiera corre
nickname = null     // OK, el tipo lo permite
```

Para trabajar con un tipo nullable, el compilador **obliga** a manejar el caso null antes de usarlo:

```kotlin
// Safe call: ejecuta solo si no es null, si no, devuelve null
val length = nickname?.length

// Elvis operator: valor por defecto si es null
val safeName = nickname ?: "Sin apodo"

// Null check + smart cast: dentro del if, el compilador SABE que no es null
if (nickname != null) {
    println(nickname.length)   // smart cast a String (no String?)
}

// !! (not-null assertion): "confío en que no es null" — lanza NPE si me equivoco
val forced = nickname!!.length   // usar con MUCHO cuidado, casi siempre evitable
```

**Por qué importa:** mueve un error que en Java se descubre en **runtime** (crash con NPE) a un error que se detecta en **tiempo de compilación** (ni siquiera compila si no manejás el null). Es una de las razones centrales por las que Android migró a Kotlin.

**El `!!` es la trampa:** usarlo "para que compile" sin pensar reintroduce exactamente el problema que null safety vino a resolver. Es aceptable solo cuando tenés una garantía real (ej. justo después de un chequeo que el compilador no pudo smart-castear).

---

## 5. Generics

Permiten escribir código que funciona con **cualquier tipo** manteniendo la verificación de tipos en compilación, en vez de usar `Any` y castear a mano.

```kotlin
class Box<T>(val value: T)

val stringBox = Box("hola")   // Box<String>, verificado
val intBox = Box(42)          // Box<Int>, verificado
```

**Type erasure:** en la JVM, los tipos genéricos se **borran** en runtime — `List<String>` y `List<Int>` son, en ejecución, solo `List`. Por eso no podés hacer `if (list is List<String>)`, ni usar `T` directamente dentro de una función genérica para un chequeo de tipo en runtime.

### Variance: `out` / `in`

Por defecto los genéricos son **invariantes**: `List<Dog>` no es subtipo de `List<Animal>` aunque `Dog` sí lo sea de `Animal`. La varianza relaja esto de forma segura:

- **`out` (covariante):** el genérico solo **produce** `T` (nunca lo recibe como parámetro). Entonces `Producer<Dog>` puede tratarse como `Producer<Animal>` — sube en la jerarquía. Ejemplo real: `List<out T>` en Kotlin es covariante.
- **`in` (contravariante):** el genérico solo **consume** `T`. Entonces `Consumer<Animal>` puede tratarse como `Consumer<Dog>` — baja en la jerarquía. Ejemplo real: `Comparator<in T>`.

```kotlin
val dogs: List<Dog> = listOf(Dog())
val animals: List<Animal> = dogs   // OK: List es covariante (out), solo lee

val animalComparator: Comparator<Animal> = compareBy { it.name }
dogs.sortedWith(animalComparator)  // OK: Comparator es contravariante (in)
```

`MutableList` es **invariante** (no covariante) porque también **consume** (con `add`) — si fuera covariante, podrías meter un `Cat` en una `MutableList<Dog>` vista como `MutableList<Animal>`.

### `reified` — recuperar el tipo en runtime

Por el type erasure, una función genérica normal no puede usar `T` en runtime (`value is T` no compila). `reified` levanta esa restricción, pero **requiere `inline`**: el compilador copia el cuerpo de la función en cada call-site, sustituyendo `T` por el tipo real, así que el tipo nunca se borra.

```kotlin
inline fun <reified T> isType(value: Any): Boolean = value is T

isType<String>("hola")   // true — T existe en runtime gracias a inline + reified
```

---

## 6. Inmutabilidad vs mutabilidad

**`val`/`var` controla la referencia; `List`/`MutableList` controla el contenido** — son dos ejes independientes.

```kotlin
val list = mutableListOf(1, 2, 3)
list.add(4)              // OK: val protege la REFERENCIA, no el contenido
// list = mutableListOf() // ERROR: esto sí lo impide val
```

| Declaración | ¿Reasignar? | ¿Modificar contenido? |
|---|---|---|
| `val x: List` | No | No |
| `var x: List` | Sí | No |
| `val x: MutableList` | No | Sí |
| `var x: MutableList` | Sí | Sí |

**`List` es solo read-only, no inmutable.** Si otra referencia mutable apunta al mismo objeto, puede cambiarlo "por debajo":

```kotlin
val mutable = mutableListOf(1, 2, 3)
val readOnly: List<Int> = mutable   // misma lista, vista de solo lectura
mutable.add(4)
println(readOnly)   // [1, 2, 3, 4] — ¡"read-only" cambió igual!
```

### Por qué preferir inmutabilidad

1. **Thread-safety sin locks:** un objeto que nadie puede modificar es seguro de compartir entre corrutinas/hilos sin sincronización — elimina de raíz las condiciones de carrera.
2. **Estado predecible:** no tenés que rastrear "quién modificó esto y cuándo". Un bug clásico: pasás una lista a una función que la modifica sin que lo esperes.
3. **Identidad estable en colecciones:** un objeto inmutable tiene `hashCode` estable, seguro en `HashSet`/`HashMap`. Una `data class` con `var` puede "perderse" en un set si su hashCode cambia mientras está dentro.
4. **Habilita comparación estructural confiable:** es lo que usa el conflation de `StateFlow` y el patrón de estado inmutable de MVI — cada cambio es un objeto nuevo (`copy()`), nunca una mutación.

**El costo:** cada cambio crea un objeto nuevo — presión de memoria si el estado es grande y cambia muy seguido. En la práctica, para Android, casi siempre vale la pena igual.

---

## Resumen — tabla de decisión rápida

| Necesito... | Uso |
|---|---|
| Constante literal conocida en compilación | `const val` |
| Valor de solo lectura en runtime (cualquier tipo) | `val` |
| Un tipo distinto sin overhead, evitar mezclar valores | `value class` |
| Solo un nombre más legible, sin nueva seguridad de tipos | `typealias` |
| Expresar que algo puede o no tener valor | `String?` (nullable) + safe calls / Elvis |
| Código reusable con cualquier tipo, type-safe | Generics |
| Un genérico que solo produce T | `out` (covariante) |
| Un genérico que solo consume T | `in` (contravariante) |
| Usar `T` en runtime dentro de una función genérica | `inline fun <reified T>` |
| Prevenir mutación accidental / compartir entre hilos seguro | Inmutabilidad (`val` + `List`, no `MutableList`) |

---

## Frases para entrevista

- *"const val se resuelve en compilación e inlinea el valor; val se evalúa en runtime."*
- *"value class crea un tipo distinto sin overhead; typealias es solo un nombre, sin seguridad de tipos nueva."*
- *"Null safety mueve el NPE de un crash en runtime a un error de compilación."*
- *"List en Kotlin es read-only, no inmutable — el objeto subyacente puede mutar por otra referencia."*
- *"Un objeto inmutable es thread-safe por definición: sin mutación no hay condición de carrera."*
- *"reified necesita inline porque el código se copia con el tipo real sustituido en cada call-site."*

## Trampas típicas de entrevista

**"¿typealias te da seguridad de tipos?"** → No, es intercambiable con el tipo original. Para eso existe `value class`.

**"`val list = mutableListOf(1,2,3); list.add(4)` — ¿compila?"** → Sí, porque `val` protege la referencia, no el contenido.

**"¿`List` es inmutable?"** → No, es read-only — puede haber otra referencia `MutableList` al mismo objeto que sí lo cambie.