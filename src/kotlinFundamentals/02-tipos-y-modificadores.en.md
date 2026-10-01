# Types & Modifiers / Tipos y modificadores

[Versión en español](02-tipos-y-modificadores.es.md)

---

## 1. `const val` vs `val`

Both create a read-only (non-reassignable) property, but they differ in **when** the value is fixed:

| | `val` | `const val` |
|---|---|---|
| Evaluated | At runtime | At **compile time** |
| Allowed types | Any | Only primitives and `String` |
| Where declared | Anywhere (property, local, constructor) | Top-level or inside an `object`/`companion object` |
| Can depend on a function/computation | Yes | No — must be a literal known at compile-time |

```kotlin
val currentTime = System.currentTimeMillis()   // runtime: changes every execution
const val MAX_RETRIES = 3                       // compile-time: fixed literal
```

**Why `const` exists:** it's an optimization — the compiler **inlines** the value directly where it's used (like Java's `static final`), without needing to access a field at runtime. That's why it only accepts types the compiler can resolve at compile time (you can't write `const val x = computeSomething()`).

**Practical rule:** `const val` for true constants known ahead of time (`MAX_RETRIES`, `BASE_URL`); `val` for everything else, including any value that depends on logic or a call.

---

## 2. `value class`

Wraps a type (usually primitive) to give it **semantic meaning and type safety**, without the cost of creating a real object at runtime — the compiler "unwraps" it (inlines it) in most cases.

```kotlin
@JvmInline
value class UserId(val value: String)

fun getUser(id: UserId) { /* ... */ }

getUser(UserId("123"))     // forced to wrap — won't accept a plain String
// getUser("123")          // ERROR: String is not UserId
```

**The problem it solves:** without `value class`, a `UserId` would just be a `String`, and the compiler would let any `String` through where a `UserId` is expected — you could mix up an `OrderId` with a `UserId` by mistake and the compiler wouldn't complain. `value class` creates a **distinct** type at compile time, but at runtime (in most cases) behaves like the wrapped type, with no new-object overhead.

**`value class` vs `typealias`** (next section): this is the comparison that always comes up.

---

## 3. `typealias`

An **alternative name** for an existing type. It doesn't create a new type — it's pure syntactic sugar, no runtime cost.

```kotlin
typealias UserId = String
typealias ClickHandler = (View) -> Unit

fun register(id: UserId, onClick: ClickHandler) { ... }
// EXACTLY equivalent to: fun register(id: String, onClick: (View) -> Unit)
```

It's useful for **readability** (naming a complex functional type) and for **shortening** long repeated generics. But it gives **no type safety**: `UserId` is still interchangeable with any `String`.

```kotlin
val id: UserId = "any string"   // compiles fine, no protection
```

### `typealias` vs `value class` — the difference interviewers ask about

| | `typealias` | `value class` |
|---|---|---|
| Creates new type | No, same type | Yes, distinct type |
| Type safety | No (interchangeable with the original) | Yes (not mixable) |
| Runtime cost | Zero | Zero (unwraps/inlines) |
| Purpose | Readability, shortening | Distinguishing types semantically |

**Rule:** `typealias` when you just want a more readable name; `value class` when you want the compiler to **prevent mixing** values that are technically the same primitive type but semantically different (`UserId` vs `OrderId`, both `String`).

---

## 4. Null safety

Kotlin's type system distinguishes at **compile time** between types that can be `null` and those that can't, eliminating `NullPointerException` as a class of bug.

```kotlin
var name: String = "Ana"       // CANNOT be null — the compiler guarantees it
var nickname: String? = null   // CAN be null — the `?` marks it explicitly

name = null        // compile ERROR, doesn't even run
nickname = null     // OK, the type allows it
```

To work with a nullable type, the compiler **forces** you to handle the null case before using it:

```kotlin
// Safe call: runs only if not null, otherwise returns null
val length = nickname?.length

// Elvis operator: default value if null
val safeName = nickname ?: "No nickname"

// Null check + smart cast: inside the if, the compiler KNOWS it's not null
if (nickname != null) {
    println(nickname.length)   // smart cast to String (not String?)
}

// !! (not-null assertion): "I trust it's not null" — throws NPE if you're wrong
val forced = nickname!!.length   // use with great care, almost always avoidable
```

**Why it matters:** it moves an error that in Java is discovered at **runtime** (crash with NPE) to one caught at **compile time** (it won't even compile if you don't handle the null). It's one of the central reasons Android moved to Kotlin.

**The `!!` is the trap:** using it "just to make it compile" without thinking reintroduces exactly the problem null safety was meant to solve. It's acceptable only when you have a real guarantee (e.g. right after a check the compiler couldn't smart-cast).

---

## 5. Generics

Allow writing code that works with **any type** while keeping compile-time type checking, instead of using `Any` and casting by hand.

```kotlin
class Box<T>(val value: T)

val stringBox = Box("hello")   // Box<String>, checked
val intBox = Box(42)           // Box<Int>, checked
```

**Type erasure:** on the JVM, generic types get **erased** at runtime — `List<String>` and `List<Int>` are, at runtime, just `List`. That's why you can't do `if (list is List<String>)`, nor use `T` directly inside a generic function for a runtime type check.

### Variance: `out` / `in`

By default generics are **invariant**: `List<Dog>` is not a subtype of `List<Animal>` even though `Dog` is a subtype of `Animal`. Variance relaxes this safely:

- **`out` (covariant):** the generic only **produces** `T` (never receives it as a parameter). So `Producer<Dog>` can be treated as `Producer<Animal>` — it moves up the hierarchy. Real example: `List<out T>` in Kotlin is covariant.
- **`in` (contravariant):** the generic only **consumes** `T`. So `Consumer<Animal>` can be treated as `Consumer<Dog>` — it moves down the hierarchy. Real example: `Comparator<in T>`.

```kotlin
val dogs: List<Dog> = listOf(Dog())
val animals: List<Animal> = dogs   // OK: List is covariant (out), only reads

val animalComparator: Comparator<Animal> = compareBy { it.name }
dogs.sortedWith(animalComparator)  // OK: Comparator is contravariant (in)
```

`MutableList` is **invariant** (not covariant) because it also **consumes** (with `add`) — if it were covariant, you could put a `Cat` into a `MutableList<Dog>` viewed as `MutableList<Animal>`.

### `reified` — recovering the type at runtime

Because of type erasure, a regular generic function can't use `T` at runtime (`value is T` doesn't compile). `reified` lifts that restriction, but **requires `inline`**: the compiler copies the function's body at each call site, substituting `T` with the real type, so the type never gets erased.

```kotlin
inline fun <reified T> isType(value: Any): Boolean = value is T

isType<String>("hello")   // true — T exists at runtime thanks to inline + reified
```

---

## 6. Immutability vs mutability

**`val`/`var` controls the reference; `List`/`MutableList` controls the content** — two independent axes.

```kotlin
val list = mutableListOf(1, 2, 3)
list.add(4)              // OK: val protects the REFERENCE, not the content
// list = mutableListOf() // ERROR: val DOES prevent this
```

| Declaration | Reassign? | Modify content? |
|---|---|---|
| `val x: List` | No | No |
| `var x: List` | Yes | No |
| `val x: MutableList` | No | Yes |
| `var x: MutableList` | Yes | Yes |

**`List` is just read-only, not immutable.** If another mutable reference points to the same object, it can change it "underneath":

```kotlin
val mutable = mutableListOf(1, 2, 3)
val readOnly: List<Int> = mutable   // same list, read-only view
mutable.add(4)
println(readOnly)   // [1, 2, 3, 4] — the "read-only" one changed too!
```

### Why prefer immutability

1. **Thread-safety without locks:** an object no one can modify is safe to share across coroutines/threads with no synchronization — it eliminates race conditions at the root.
2. **Predictable state:** you don't have to track "who modified this and when." A classic bug: you pass a list to a function that modifies it without you expecting it.
3. **Stable identity in collections:** an immutable object has a stable `hashCode`, safe in `HashSet`/`HashMap`. A `data class` with `var` can "get lost" in a set if its hashCode changes while it's inside.
4. **Enables reliable structural comparison:** this is what `StateFlow`'s conflation and MVI's immutable state pattern rely on — every change is a new object (`copy()`), never a mutation.

**The cost:** every change creates a new object — memory pressure if the state is large and changes very often. In practice, for Android, it's almost always worth it anyway.

---

## Summary — quick decision table

| I need... | Use |
|---|---|
| A compile-time-known literal constant | `const val` |
| A read-only runtime value (any type) | `val` |
| A distinct type with no overhead, avoid mixing values | `value class` |
| Just a more readable name, no new type safety | `typealias` |
| Expressing that something may or may not have a value | `String?` (nullable) + safe calls / Elvis |
| Reusable, type-safe code for any type | Generics |
| A generic that only produces T | `out` (covariant) |
| A generic that only consumes T | `in` (contravariant) |
| Using `T` at runtime inside a generic function | `inline fun <reified T>` |
| Preventing accidental mutation / safe sharing across threads | Immutability (`val` + `List`, not `MutableList`) |

---

## Interview phrases

- *"const val is resolved at compile time and inlined; val is evaluated at runtime."*
- *"value class creates a distinct type with no overhead; typealias is just a name, no new type safety."*
- *"Null safety moves the NPE from a runtime crash to a compile-time error."*
- *"List in Kotlin is read-only, not immutable — the underlying object can still mutate through another reference."*
- *"An immutable object is thread-safe by definition: no mutation, no race condition."*
- *"reified needs inline because the code gets copied with the real type substituted at each call site."*

## Common interview traps

**"Does typealias give you type safety?"** → No, it's interchangeable with the original type. That's what `value class` is for.

**"`val list = mutableListOf(1,2,3); list.add(4)` — does it compile?"** → Yes, because `val` protects the reference, not the content.

**"Is `List` immutable?"** → No, it's read-only — there could be another `MutableList` reference to the same object that does change it.