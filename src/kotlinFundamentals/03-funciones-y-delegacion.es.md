# Functions & Delegation / Funciones y delegación

[English version](03-funciones-y-delegacion.en.md)

---

## 1. Extension functions (`fun Tipo.nombre()`)

Permiten **agregar funciones a una clase existente sin modificarla ni heredar de ella** — ni siquiera necesitás tener acceso al código fuente de esa clase.

```kotlin
fun String.isValidEmail(): Boolean = this.contains("@") && this.contains(".")

"test@mail.com".isValidEmail()   // true — como si fuera un método nativo de String
```

**Cómo funciona por debajo:** es azúcar sintáctico — el compilador la convierte en una función estática que recibe el objeto como primer parámetro (`isValidEmail(this: String)`). No modifica la clase real, no tiene acceso a sus miembros `private`, y **se resuelve de forma estática** (según el tipo declarado en tiempo de compilación, no el tipo real en runtime) — a diferencia de los métodos de la clase, que son polimórficos.

```kotlin
open class Animal
class Dog : Animal()

fun Animal.speak() = "..."
fun Dog.speak() = "Woof"

val a: Animal = Dog()
a.speak()   // "..." — usa la extensión de Animal (tipo declarado), NO la de Dog
```

**Por qué existen:** evitan clases "utils" estáticas (`StringUtils.isValidEmail(str)`) y dan una sintaxis más legible y encadenable (`str.trim().isValidEmail()`).

---

## 2. Higher-order functions

Una función que **recibe otra función como parámetro** y/o **devuelve una función**. Es la base de `map`, `filter`, los builders de corrutinas (`launch`, `async`), y las scope functions.

```kotlin
fun calculate(a: Int, b: Int, operation: (Int, Int) -> Int): Int = operation(a, b)

calculate(3, 4) { x, y -> x + y }   // 7 — le paso la lógica como argumento
calculate(3, 4) { x, y -> x * y }   // 12
```

Esto es lo que permite patrones como callbacks, estrategias intercambiables, y el estilo declarativo de Kotlin (`list.filter { it > 5 }.map { it * 2 }`).

---

## 3. Lambdas

Es un bloque de código que representa una **función anónima**, puede ser tratada como cualquier otra variable.

```kotlin
val sum: (Int, Int) -> Int = { a, b -> a + b }
sum(2, 3)   // 5

// Sintaxis trailing lambda: si el último parámetro es una función, sale del paréntesis
list.forEach { println(it) }   // 'it' es el nombre implícito del único parámetro
```

### Lambdas estables (el detalle de Compose)

En Compose, para que un composable **se salte la recomposición**, sus parámetros deben ser de tipos **estables**. Una lambda es estable si **no captura nada, o captura solo valores estables**.

```kotlin
// Lambda SIN captura: estable, se puede reutilizar sin recrear el objeto
Button(onClick = { doSomething() })

// Lambda que CAPTURA una variable: su estabilidad depende de si esa variable es estable
Button(onClick = { doSomethingWith(userId) })   // depende de si userId es estable
```
Sirven para evitar que algún composable se recomponga de manera innecesaria solo porque algún callback se volvió a crear en memoria

Si una lambda captura algo inestable (ej. una `List` normal, que Compose considera inestable por ser interfaz), Compose no puede garantizar que comparar la lambda sea confiable, y eso puede forzar recomposiciones de más. Por eso al optimizar Compose también importa qué capturan las lambdas que pasás, no solo los parámetros "normales".

---

## 4. `inline fun`

Le dice al compilador que **copie el cuerpo de la función y de sus lambdas directamente en el call-site**, en vez de crear objetos lambda y hacer llamadas de función.

```kotlin
inline fun processItems(items: List<Int>, action: (Int) -> Unit) {
    for (item in items) action(item)
}
// En el call-site, el compilador pega el código real: sin objeto lambda, sin salto de función
```

**Beneficios:** elimina el overhead de crear un objeto por cada lambda pasada; habilita `return` no-local desde la lambda; y es **requisito** para `reified` (el tipo genérico sobrevive en runtime porque el código se copia con el tipo real sustituido).

**`noinline`** excluye una lambda específica del inlining (para poder guardarla como objeto o pasarla a otra función). **`crossinline`** mantiene el inlining pero prohíbe el `return` no-local, para cuando la lambda se ejecuta en otro contexto (otro hilo, otro objeto).

```kotlin
inline fun runInBackground(crossinline action: () -> Unit) {
    Thread { action() }.start()   // action corre en OTRO hilo: return no-local sería peligroso
}
```

**Cuándo NO usarlo:** en funciones grandes sin lambdas — el inlining copia todo el cuerpo en cada call-site, inflando el bytecode (code bloat) sin beneficio real. Vale la pena solo en funciones **chicas que reciben lambdas** y se llaman seguido.

---

## 5. `by lazy`

Delegado de propiedad para **inicialización diferida**: el valor se calcula la primera vez que se accede a la propiedad, no al crear el objeto, y se cachea para siempre.

```kotlin
class ProfileScreen {
    val formatter: SimpleDateFormat by lazy {
        SimpleDateFormat("dd/MM/yyyy", Locale.getDefault())   // corre UNA vez, al primer uso
    }
}
```

Uso típico en Android: dependencias que necesitan que el objeto contenedor ya esté creado.

```kotlin
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by lazy {
        ViewModelProvider(this)[UserViewModel::class.java]
    }
}
```

Por defecto es **thread-safe** (`LazyThreadSafetyMode.SYNCHRONIZED`). Si garantizás acceso desde un solo hilo, `LazyThreadSafetyMode.NONE` es más rápido (sin locks). Solo aplica a `val` — un valor que se calcula una vez y no cambia.

---

## 6. `lateinit var`

Declara una propiedad `var` **no-nula** sin inicializarla de inmediato, prometiendo que se inicializará antes de usarla. Evita forzar un tipo nullable o un valor "falso" solo para poder declarar la propiedad.

```kotlin
class UserActivity : AppCompatActivity() {
    private lateinit var binding: ActivityUserBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        binding = ActivityUserBinding.inflate(layoutInflater)   // se inicializa acá
    }
}
```

**El riesgo:** si accedés antes de inicializar, lanza `UninitializedPropertyAccessException` en runtime — el compilador no te protege de esto (a diferencia de null safety normal). Por eso `lateinit` es un **contrato que vos garantizás**, no una verificación del compilador.

**Restricciones:** solo `var` (no `val`), solo tipos no-nulos, y no funciona con tipos primitivos (`Int`, `Boolean`) por cómo están implementados internamente — para esos casos, `Delegates.notNull()` es la alternativa.

### `lateinit var` vs `by lazy` — cuándo cada uno

| | `lateinit var` | `by lazy` |
|---|---|---|
| Mutabilidad | `var` (podés reasignar) | `val` (una vez calculado, fijo) |
| Quién inicializa | Vos, en el momento que quieras | Automático, en el primer acceso |
| Tipos primitivos | No soportado | Sí |
| Uso típico | ViewBinding, inyección manual | Cálculo costoso, dependencia diferida |

---

## 7. Scope functions: `let`, `run`, `with`, `apply`, `also`

Las cinco se distinguen por **dos ejes**: cómo referís al objeto (`this` o `it`) y qué devuelven (el objeto mismo o el resultado del bloque).

| Función | Referencia | Devuelve | Es extensión |
|---|---|---|---|
| `let` | `it` | resultado del bloque | Sí |
| `run` | `this` | resultado del bloque | Sí |
| `with` | `this` | resultado del bloque | No (toma el objeto como argumento) |
| `apply` | `this` | el objeto mismo | Sí |
| `also` | `it` | el objeto mismo | Sí |

```kotlin
// let: null-safety + transformar, devuelve el resultado
user?.let { println(it.name) }

// apply: configurar y devolver el objeto mismo
val intent = Intent(this, DetailActivity::class.java).apply {
    putExtra("ID", userId)
}

// also: efecto secundario (logging) sin romper la cadena, devuelve el objeto
val result = fetchUser().also { println("Fetched: $it") }

// run: operar sobre miembros (this) y devolver un resultado
val message = user.run { if (isActive) "Activo: $name" else "Inactivo" }

// with: igual que run pero no-extensión, para "hacer varias cosas con este objeto"
with(binding) {
    title.text = "Hola"
    subtitle.text = "Mundo"
}
```

**Criterio de decisión:** ¿necesito el objeto de vuelta, o un resultado? → objeto: `apply`/`also`; resultado: `let`/`run`/`with`. ¿Referir con `this` o `it`? → `this` para configurar miembros directo; `it` para dejar explícito un side-effect o un null-check.

---

## 8. Delegation (`by`)

`by` cubre **dos mecanismos distintos** en Kotlin, que comparten palabra clave pero no tienen nada que ver entre sí.

### Delegación de clase (reenviar una interfaz)

Implementás una interfaz **reenviando el trabajo a otro objeto**, sin escribir el boilerplate de cada método — el patrón decorador sin ceremonia.

```kotlin
interface Repository { fun getData(): String }
class RepositoryImpl : Repository { override fun getData() = "data" }

// 'by delegate' reenvía TODOS los métodos de Repository a 'delegate'
class CachedRepository(private val delegate: Repository) : Repository by delegate {
    // solo sobreescribo lo que quiero cambiar, el resto lo hace delegate
}
```

### Delegación de propiedades (delegar get/set)

`by lazy` (ya visto) es el caso famoso, pero hay más:

```kotlin
// observable: notifica cada cambio
var name: String by Delegates.observable("inicial") { _, old, new ->
    println("cambió de $old a $new")
}

// vetoable: como observable, pero puede RECHAZAR el cambio
var age: Int by Delegates.vetoable(0) { _, _, new -> new >= 0 }

// delegar a un Map: la propiedad lee del mapa por su nombre
class User(map: Map<String, Any?>) {
    val name: String by map   // lee map["name"]
}
```

**Por dentro:** cualquier objeto que implemente los operadores `getValue`/`setValue` sirve como delegado — podés escribir los tuyos (ej. un delegado custom para leer/escribir en SharedPreferences de forma transparente).

---

## Resumen — tabla de decisión rápida

| Necesito... | Uso |
|---|---|
| Agregar un método a una clase que no controlo | Extension function |
| Pasar lógica como parámetro | Higher-order function + lambda |
| Eliminar overhead de lambdas en una función chica y frecuente | `inline fun` |
| Usar `T` en runtime dentro de una función genérica | `inline fun <reified T>` |
| Calcular algo caro solo si se usa, una vez | `by lazy` |
| Inicializar un `var` no-nulo después de declararlo | `lateinit var` |
| Configurar un objeto y devolverlo | `apply` |
| Efecto secundario sin romper una cadena | `also` |
| Null-check + transformar, devolver resultado | `let` |
| Reenviar una interfaz a otro objeto | Delegación de clase (`by`) |
| Reaccionar o vetar cambios de una propiedad | `Delegates.observable` / `vetoable` |

---

## Frases para entrevista

- *"Las extension functions se resuelven estáticamente por el tipo declarado, no son polimórficas como los métodos reales."*
- *"inline copia el cuerpo en el call-site — elimina objetos lambda, pero infla el bytecode si se abusa."*
- *"lateinit es un contrato que vos garantizás; si fallás, UninitializedPropertyAccessException en runtime, sin red del compilador."*
- *"apply y also devuelven el objeto; let, run y with devuelven el resultado del bloque."*
- *"by cubre dos cosas distintas: delegación de clase (reenviar una interfaz) y delegación de propiedad (delegar el get/set)."*
- *"Una lambda sin captura es estable para Compose; si captura algo inestable, puede forzar recomposición de más."*

## Trampas típicas de entrevista

**"¿Por qué una extension function no es polimórfica?"** → Porque se resuelve en tiempo de compilación según el tipo declarado de la variable, no el tipo real del objeto en runtime — al revés que los métodos de instancia normales.

**"¿Diferencia entre `let` y `also`?"** → Ambas usan `it`, pero `let` devuelve el resultado del bloque (transformar); `also` devuelve el objeto (side-effect).

**"¿Cuándo `lateinit` en vez de nullable (`var x: X? = null`)?"** → Cuando estás seguro de que se inicializa antes de usarse y no querés ensuciar el código con `?.`/`!!` en cada acceso — pero asumís el riesgo de un crash si te equivocás.
