# Classes, Objects & Inheritance / Clases, objetos y herencia

[Versión en español](01-clases-objetos-herencia.es.md)

---

## 1. `class` — the basics

A `class` is a blueprint for creating objects. By default in Kotlin it's **`final`**: you can't inherit from it unless you mark it `open`.

```kotlin
class User(val name: String, val age: Int) {
    fun isAdult() = age >= 18
}

val u = User("Ana", 30)
```

**Why `final` by default (and not `open`):** Kotlin intentionally flips Java's philosophy. Inheriting from a class that wasn't designed for it breaks encapsulation (the "fragile base class problem" — you change the base class and silently break its children). That's why inheritance is an explicit opt-in via `open`, not the default.

---

## 2. Inheritance (`open`, `override`)

To allow inheritance, the class and the members that can be overridden must be `open`:

```kotlin
open class Animal(val name: String) {
    open fun makeSound() = "..."
}

class Dog(name: String) : Animal(name) {
    override fun makeSound() = "Woof"
}
```

- `open class` → can be inherited.
- `open fun` → can be overridden in the subclass.
- `override fun` → mandatory to override (prevents accidental overrides, unlike Java where `@Override` is optional).

**Kotlin doesn't have multiple class inheritance** (a class can only extend one base class), but it can implement **several interfaces**. That's the practical reason Kotlin pushes toward composition + interfaces instead of deep class hierarchies.

```kotlin
class Dog(name: String) : Animal(name), Runnable, Comparable<Dog> {
    // one base class, N interfaces
}
```

---

## 3. `abstract class`

A class that **cannot be instantiated directly** — it exists to be inherited. It can mix **abstract** members (no implementation, mandatory to override) with **concrete** members (with implementation, shared by all subclasses).

```kotlin
abstract class Shape {
    abstract fun area(): Double        // no body, mandatory to implement
    fun describe() = "Area: ${area()}" // has a body, shared by all
}

class Circle(val radius: Double) : Shape() {
    override fun area() = Math.PI * radius * radius
}
```

**Why it exists when interfaces already do:** an `abstract class` can hold **state** (properties with a value, a constructor) and real **concrete shared logic**. An interface (before default methods) couldn't. Today interfaces also allow default function bodies, so the line has narrowed — but abstract class remains the right choice when you need **shared state** in the base.

---

## 4. `interface`

A contract: defines what methods/properties something must have, without (necessarily) saying how. Interfaces in Kotlin **can have default implementations**:

```kotlin
interface Clickable {
    fun onClick()                          // no implementation: mandatory
    fun onLongClick() { println("long") }  // default implementation
}

class Button : Clickable {
    override fun onClick() { println("click") }
    // onLongClick isn't mandatory to override, already has a default
}
```

An interface **can't hold real state with a backing field** (you can't store an actual mutable value there), though it can declare abstract properties the implementing class must provide.

### `class` vs `abstract class` vs `interface` — when to use each

| | Instantiable | Own state | Multiple inheritance | Typical use |
|---|---|---|---|---|
| `class` | Yes | Yes | — | Regular concrete object |
| `abstract class` | No | Yes (with constructor) | No (only one) | Base with state + shared logic |
| `interface` | No | No (only abstract) | Yes (several) | Contract / capability (Comparable, Clickable) |

**The deciding question:** do I need the base to hold **state** (a constructor with real properties)? → `abstract class`. Do I only need to define a **contract** that several unrelated classes can fulfill? → `interface`.

---

## 5. `data class`

A class meant to **hold data**. The compiler automatically generates, from the **primary constructor** properties:

- `equals()` / `hashCode()` — structural equality (compares content, not reference).
- `toString()` — readable: `User(id=1, name=Ana)`.
- `copy()` — copy changing only what you specify.
- `componentN()` — for destructuring (`val (id, name) = user`).

```kotlin
data class User(val id: String, val name: String, val age: Int)

val u1 = User("1", "Ana", 30)
val u2 = u1.copy(age = 31)       // copy changing only age
val (id, name, age) = u1         // destructuring
```

**Surprising details:**
- Everything generated comes **only from the primary constructor**. A property declared in the class body is ignored by `equals`/`hashCode`/`copy`/`toString`.
- `copy()` is a **shallow copy**: if a property is a mutable object, the copy shares the same reference.
- Restrictions: constructor needs at least one parameter, can't be `open`/`abstract`/`sealed`/`inner`.
- If you override `equals`/`hashCode`/`toString` yourself, the compiler respects yours.

---

## 6. `object` — native singleton

Declares a class with **a single instance**, created lazily and thread-safe automatically by the compiler. There's no constructor (you can't instantiate it with `()`).

```kotlin
object NetworkManager {
    var isConnected = false
    fun connect() { /* ... */ }
}

NetworkManager.connect()   // accessed directly, no instantiation
```

It's Kotlin's replacement for Java's manual Singleton pattern (with `getInstance()` and double-checked locking) — the language gives it to you for free, with no boilerplate.

There's also an **object expression** (anonymous instance of an interface/class, like Java's anonymous classes):

```kotlin
val listener = object : View.OnClickListener {
    override fun onClick(v: View) { /* ... */ }
}
```

---

## 7. `companion object`

An `object` **nested inside a class**, tied to that class (not to its instances) — it's the replacement for Java's `static`.

```kotlin
class User private constructor(val id: String) {
    companion object {
        fun create(id: String): User = User(id)   // "factory method"
        const val MAX_AGE = 150
    }
}

val u = User.create("123")   // called on the CLASS, not an instance
```

Typical uses: factory methods, constants tied to the class, implementing a "static" interface (a companion can implement an interface).

---

## 8. `data object`

A combination of `object` + the benefits of `data class`, but without data (since an `object` has no constructor with properties). It automatically gives you a readable `toString()` and consistent `equals`/`hashCode` — mainly useful for dataless states in a `sealed class/interface`.

```kotlin
sealed interface UiState {
    data object Loading : UiState      // toString → "Loading", not "Loading@3f2a"
    data class Success(val data: String) : UiState
}
```

Without `data`, a plain `object` still works fine in a `when`, but its default `toString()` is ugly (`Loading@hashcode`) — bad for logs/debugging.

### `data object` vs `companion object` — the confusion to clear up

These are completely different things that only share the word `object`:

| | `companion object` | `data object` |
|---|---|---|
| What it is | An `object` **tied to a class** (replaces `static`) | A **modifier** on `object` that adds readable `toString`/`equals` |
| Where it lives | Nested inside a class | Standalone, or as a case of a sealed |
| What for | Factory methods, class constants | Representing a dataless case (ideal in sealed) |
| Can combine | `companion data object { ... }` is also valid | — |

They aren't alternatives to each other — in fact you can have a `companion data object` if you need both at once.

---

## 9. `enum class`

A **fixed, closed set of unique instances**, all with the same shape (structure).

```kotlin
enum class Status(val label: String) {
    LOADING("Loading"),
    SUCCESS("Done"),
    ERROR("Error");

    fun isTerminal() = this != LOADING
}

Status.values()       // array with all cases
Status.valueOf("LOADING")
```

Gives you for free: `values()`, `valueOf()`, `.ordinal`, `.name`, and it's iterable. But **all instances share the same structure** — you can't have one case carry different data from another.

---

## 10. `sealed class` / `sealed interface`

A **closed** hierarchy: all implementations must live in the same module/package, so the compiler knows **the whole set** and can verify a `when` is **exhaustive without `else`**.

```kotlin
sealed interface Result
data class Success(val data: String) : Result
data class Error(val message: String) : Result
data object Loading : Result

fun render(result: Result) = when (result) {
    is Success -> showData(result.data)
    is Error -> showError(result.message)
    Loading -> showSpinner()
    // no else: if I add a new case and forget to handle it, COMPILE ERROR
}
```

With a regular `interface`, the `when` forces you to add `else` (the compiler can't guarantee it knows every case) — and that `else` is exactly where bugs hide when you add a new case and forget to handle it.

### `sealed class` vs `sealed interface`

`sealed` is a **modifier** that closes the hierarchy; it works the same on `class` or `interface`. The difference is the usual one between class and interface:

- **sealed class**: can have **state/constructor** at the root, shared by all cases. Single inheritance.
- **sealed interface**: no state of its own, but allows a case to **implement several interfaces** (more flexible).

```kotlin
// sealed class: when ALL cases share a property
sealed class ScreenState(val isRefreshing: Boolean) {
    data object Loading : ScreenState(isRefreshing = false)
    data class Success(val data: Data) : ScreenState(isRefreshing = false)
}

// sealed interface: independent cases, nothing in common
sealed interface UiState {
    data object Loading : UiState
    data class Success(val data: Data) : UiState
}
```

**Practical rule:** `sealed interface` by default (more flexible, doesn't drag unnecessary state); `sealed class` only when **all** cases share a property or common logic you don't want to repeat in each one.

### `sealed` vs `enum` — when to use each

| | `enum class` | `sealed class/interface` |
|---|---|---|
| Instances | Unique, fixed | Can have **multiple** instances of the same case |
| Structure | Same across all cases | Each case can have **different own data** |
| `values()`/`ordinal` | Yes, free | No |
| Typical use | Days of the week, filter options | UI state (`Success(data)`, `Error(msg)`) |

**The deciding question:** does each case need to carry **different data**? → `sealed`. Are they just interchangeable constants with no data of their own? → `enum`.

---

## Summary — quick decision table

| I need... | Use |
|---|---|
| A simple data container | `data class` |
| A single global instance | `object` |
| Something "static" tied to a class | `companion object` |
| A dataless case inside a sealed | `data object` |
| A closed set of homogeneous constants | `enum class` |
| A closed set where each case carries different data | `sealed class` / `sealed interface` |
| A contract implemented by unrelated classes | `interface` |
| A base with state + shared logic, not instantiated directly | `abstract class` |

---

## Interview phrases

- *"Kotlin doesn't allow multiple class inheritance, but does for interfaces — that's why it favors composition over deep hierarchies."*
- *"abstract class when I need shared state in the base; interface when I'm only defining a contract."*
- *"data class generates equals/hashCode/copy only from the primary constructor — a property in the body gets ignored."*
- *"companion object replaces static; data object is a modifier for readable toString/equals on dataless objects — they're not the same thing."*
- *"sealed gives exhaustiveness without else: adding a new case breaks compilation instead of hiding in an else."*
- *"sealed interface by default; sealed class only if all cases share state."*

## Common interview traps

**"Why is class `final` by default in Kotlin?"** → To avoid the fragile base class problem: inheriting from something not designed for it breaks encapsulation. Inheritance is opt-in via `open`.

**"I show you a data class with a `var` declared in the body — are two instances that differ only there equal?"** → Yes, `equals` only looks at the primary constructor.

**"When would you use `sealed class` instead of `sealed interface`?"** → Only when all cases share a property or common logic at the root; otherwise interface is more flexible and avoids repeated boilerplate.