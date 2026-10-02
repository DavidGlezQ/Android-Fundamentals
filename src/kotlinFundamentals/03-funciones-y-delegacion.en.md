# Functions & Delegation / Funciones y delegación

[Versión en español](03-funciones-y-delegacion.md)

---

## 1. Extension functions (`fun Type.name()`)

Allow **adding functions to an existing class without modifying it or inheriting from it** — you don't even need access to that class's source code.

```kotlin
fun String.isValidEmail(): Boolean = this.contains("@") && this.contains(".")

"test@mail.com".isValidEmail()   // true — as if it were a native String method
```

**How it works under the hood:** it's syntactic sugar — the compiler turns it into a static function that receives the object as its first parameter (`isValidEmail(this: String)`). It doesn't modify the real class, has no access to its `private` members, and **is resolved statically** (based on the declared type at compile time, not the real runtime type) — unlike class methods, which are polymorphic.

```kotlin
open class Animal
class Dog : Animal()

fun Animal.speak() = "..."
fun Dog.speak() = "Woof"

val a: Animal = Dog()
a.speak()   // "..." — uses Animal's extension (declared type), NOT Dog's
```

**Why they exist:** they avoid static "utils" classes (`StringUtils.isValidEmail(str)`) and give a more readable, chainable syntax (`str.trim().isValidEmail()`).

---

## 2. Higher-order functions

A function that **receives another function as a parameter** and/or **returns a function**. It's the foundation of `map`, `filter`, the coroutine builders (`launch`, `async`), and scope functions.

```kotlin
fun calculate(a: Int, b: Int, operation: (Int, Int) -> Int): Int = operation(a, b)

calculate(3, 4) { x, y -> x + y }   // 7 — I pass the logic as an argument
calculate(3, 4) { x, y -> x * y }   // 12
```

This is what enables patterns like callbacks, interchangeable strategies, and Kotlin's declarative style (`list.filter { it > 5 }.map { it * 2 }`).

---

## 3. Lambdas

An **anonymous function** you can treat as a value: assign it to a variable, pass it as an argument, return it.

```kotlin
val sum: (Int, Int) -> Int = { a, b -> a + b }
sum(2, 3)   // 5

// Trailing lambda syntax: if the last parameter is a function, it goes outside the parens
list.forEach { println(it) }   // 'it' is the implicit name of the single parameter
```

### Stable lambdas (the Compose detail)

In Compose, for a composable to **skip recomposition**, its parameters must be of **stable** types. A lambda is stable if it **captures nothing, or captures only stable values**.

```kotlin
// Lambda with NO capture: stable, can be reused without recreating the object
Button(onClick = { doSomething() })

// Lambda that CAPTURES a variable: its stability depends on whether that variable is stable
Button(onClick = { doSomethingWith(userId) })   // depends on whether userId is stable
```

If a lambda captures something unstable (e.g. a plain `List`, which Compose considers unstable because it's an interface), Compose can't guarantee comparing the lambda is reliable, which can force extra recompositions. That's why, when optimizing Compose, what lambdas capture matters too — not just "normal" parameters.

---

## 4. `inline fun`

Tells the compiler to **copy the function's body and its lambdas directly at the call site**, instead of creating lambda objects and making function calls.

```kotlin
inline fun processItems(items: List<Int>, action: (Int) -> Unit) {
    for (item in items) action(item)
}
// At the call site, the compiler pastes the real code: no lambda object, no function jump
```

**Benefits:** eliminates the overhead of creating an object per lambda passed; enables non-local `return` from the lambda; and is a **requirement** for `reified` (the generic type survives at runtime because the code is copied with the real type substituted).

**`noinline`** excludes a specific lambda from inlining (so it can be stored as an object or passed to another function). **`crossinline`** keeps the inlining but forbids the non-local `return`, for when the lambda runs in another context (another thread, another object).

```kotlin
inline fun runInBackground(crossinline action: () -> Unit) {
    Thread { action() }.start()   // action runs on ANOTHER thread: non-local return would be unsafe
}
```

**When NOT to use it:** in large functions with no lambdas — inlining copies the whole body at every call site, bloating the bytecode with no real benefit. It's only worth it for **small functions that take lambdas** and get called often.

---

## 5. `by lazy`

Property delegate for **deferred initialization**: the value is computed the first time the property is accessed, not when the object is created, and is cached forever.

```kotlin
class ProfileScreen {
    val formatter: SimpleDateFormat by lazy {
        SimpleDateFormat("dd/MM/yyyy", Locale.getDefault())   // runs ONCE, on first use
    }
}
```

Typical Android use: dependencies that need the containing object to already exist.

```kotlin
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by lazy {
        ViewModelProvider(this)[UserViewModel::class.java]
    }
}
```

By default it's **thread-safe** (`LazyThreadSafetyMode.SYNCHRONIZED`). If you guarantee single-thread access, `LazyThreadSafetyMode.NONE` is faster (no locks). Only applies to `val` — a value computed once that doesn't change.

---

## 6. `lateinit var`

Declares a non-null `var` property without initializing it right away, promising it'll be initialized before use. Avoids forcing a nullable type or a "fake" value just to declare the property.

```kotlin
class UserActivity : AppCompatActivity() {
    private lateinit var binding: ActivityUserBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        binding = ActivityUserBinding.inflate(layoutInflater)   // initialized here
    }
}
```

**The risk:** accessing it before initialization throws `UninitializedPropertyAccessException` at runtime — the compiler doesn't protect you from this (unlike regular null safety). That's why `lateinit` is a **contract you guarantee**, not a compiler-verified one.

**Restrictions:** only `var` (not `val`), only non-null types, and doesn't work with primitive types (`Int`, `Boolean`) due to how they're implemented internally — for those, `Delegates.notNull()` is the alternative.

### `lateinit var` vs `by lazy` — when to use each

| | `lateinit var` | `by lazy` |
|---|---|---|
| Mutability | `var` (reassignable) | `val` (fixed once computed) |
| Who initializes | You, whenever you want | Automatic, on first access |
| Primitive types | Not supported | Yes |
| Typical use | ViewBinding, manual injection | Expensive computation, deferred dependency |

---

## 7. Scope functions: `let`, `run`, `with`, `apply`, `also`

The five are distinguished by **two axes**: how you reference the object (`this` or `it`) and what they return (the object itself or the block's result).

| Function | Reference | Returns | Is extension |
|---|---|---|---|
| `let` | `it` | block's result | Yes |
| `run` | `this` | block's result | Yes |
| `with` | `this` | block's result | No (takes the object as argument) |
| `apply` | `this` | the object itself | Yes |
| `also` | `it` | the object itself | Yes |

```kotlin
// let: null-safety + transform, returns the result
user?.let { println(it.name) }

// apply: configure and return the object itself
val intent = Intent(this, DetailActivity::class.java).apply {
    putExtra("ID", userId)
}

// also: side effect (logging) without breaking the chain, returns the object
val result = fetchUser().also { println("Fetched: $it") }

// run: operate on members (this) and return a result
val message = user.run { if (isActive) "Active: $name" else "Inactive" }

// with: same as run but non-extension, for "do several things with this object"
with(binding) {
    title.text = "Hello"
    subtitle.text = "World"
}
```

**Decision criteria:** do I need the object back, or a result? → object: `apply`/`also`; result: `let`/`run`/`with`. Reference with `this` or `it`? → `this` to configure members directly; `it` to make a side effect or null-check explicit.

---

## 8. Delegation (`by`)

`by` covers **two distinct mechanisms** in Kotlin that share a keyword but have nothing to do with each other.

### Class delegation (forwarding an interface)

You implement an interface **by forwarding the work to another object**, without writing the boilerplate for each method — the decorator pattern with no ceremony.

```kotlin
interface Repository { fun getData(): String }
class RepositoryImpl : Repository { override fun getData() = "data" }

// 'by delegate' forwards ALL of Repository's methods to 'delegate'
class CachedRepository(private val delegate: Repository) : Repository by delegate {
    // I only override what I want to change, the rest is handled by delegate
}
```

### Property delegation (delegating get/set)

`by lazy` (seen above) is the famous case, but there are more:

```kotlin
// observable: notifies every change
var name: String by Delegates.observable("initial") { _, old, new ->
    println("changed from $old to $new")
}

// vetoable: like observable, but can REJECT the change
var age: Int by Delegates.vetoable(0) { _, _, new -> new >= 0 }

// delegating to a Map: the property reads from the map by its name
class User(map: Map<String, Any?>) {
    val name: String by map   // reads map["name"]
}
```

**Under the hood:** any object implementing the `getValue`/`setValue` operators works as a delegate — you can write your own (e.g. a custom delegate for reading/writing SharedPreferences transparently).

---

## Summary — quick decision table

| I need... | Use |
|---|---|
| Add a method to a class I don't own | Extension function |
| Pass logic as a parameter | Higher-order function + lambda |
| Remove lambda overhead in a small, frequent function | `inline fun` |
| Use `T` at runtime inside a generic function | `inline fun <reified T>` |
| Compute something expensive only if used, once | `by lazy` |
| Initialize a non-null `var` after declaring it | `lateinit var` |
| Configure an object and return it | `apply` |
| Side effect without breaking a chain | `also` |
| Null-check + transform, return the result | `let` |
| Forward an interface to another object | Class delegation (`by`) |
| React to or veto property changes | `Delegates.observable` / `vetoable` |

---

## Interview phrases

- *"Extension functions resolve statically by the declared type, they're not polymorphic like real methods."*
- *"inline copies the body at the call site — removes lambda objects, but bloats bytecode if overused."*
- *"lateinit is a contract you guarantee; if you're wrong, UninitializedPropertyAccessException at runtime, no compiler safety net."*
- *"apply and also return the object; let, run and with return the block's result."*
- *"by covers two different things: class delegation (forwarding an interface) and property delegation (delegating get/set)."*
- *"A lambda with no capture is stable for Compose; if it captures something unstable, it can force extra recomposition."*

## Common interview traps

**"Why isn't an extension function polymorphic?"** → Because it's resolved at compile time based on the variable's declared type, not the object's real runtime type — the opposite of regular instance methods.

**"Difference between `let` and `also`?"** → Both use `it`, but `let` returns the block's result (transforming); `also` returns the object (side effect).

**"When `lateinit` instead of nullable (`var x: X? = null`)?"** → When you're sure it'll be initialized before use and don't want to clutter the code with `?.`/`!!` on every access — but you take on the risk of a crash if you're wrong.