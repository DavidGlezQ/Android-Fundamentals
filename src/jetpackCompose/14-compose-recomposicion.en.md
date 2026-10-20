# Recomposition & Stability / Recomposición y estabilidad

[Versión en español](14-compose-recomposicion.es.md)

---

## 1. What recomposition is

**Recomposition** is when Compose re-runs a composable's code (or part of it) because some `State` it reads changed value. It's not "redraw everything from scratch" - Compose tries to run only what's needed, in the smallest possible scope.

```kotlin
@Composable
fun ProfileScreen(name: String, age: Int) {
    Column {
        Text(name)   // only recomposes if "name" changes
        Text("$age years old")   // only recomposes if "age" changes
    }
}
```

If `age` changes but `name` doesn't, Compose only re-runs `Text(age)`, not `Text(name)` or the whole `Column`. This is called **granular recomposition** or **smart recomposition**.

---

## 2. Recomposition scope: the smallest possible region

Every read of a `State` inside a composable creates a point where Compose can "cut" the recomposition. The **recomposition scope** is the region of code that reruns when that state changes - usually the innermost composable that reads that value, not the whole tree.

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Column {
        Text("Counter app")            // never recomposes, doesn't read "count"
        Text("Count: $count")          // recomposes every time "count" changes
        Button(onClick = { count++ }) { Text("+1") }
    }
}
```

The first `Text` never recomposes because it doesn't read `count` - Compose knows, through static analysis, which composables read which state.

---

## 3. Skipping: why Compose can skip a recomposition

Compose can **skip** running a composable if its input parameters are equal to the last time it ran. This requires those parameters to be **stable** - if Compose can't guarantee a parameter is stable, it can't trust comparing its value and ends up recomposing "just in case."

```kotlin
@Composable
fun Greeting(name: String) {   // String is stable: Compose can compare and skip if unchanged
    Text("Hello, $name")
}
```

---

## 4. Stability: what makes a type stable or not

A type is **stable** if Compose can trust that, when its `equals()` says two instances are equal, they truly are and will remain so (the type is immutable, or if it's mutable it notifies its changes via `State`).

**Stable by default:** all primitive types (`Int`, `Boolean`, `String`, etc.), lambdas (if they only capture stable values), and `data class`es whose content is immutable.

**Not stable by default:** collection interfaces like `List`, `Map`, `Set`. Even though in practice you might pass an immutable list, the *type* `List` is an interface that *could* have a mutable implementation underneath (like `MutableList`), so the Compose compiler can't guarantee its stability and marks it as unstable.

```kotlin
// "items: List<String>" is unstable to the Compose compiler,
// even if in this particular case it never mutates - Compose can't know that
@Composable
fun ItemList(items: List<String>) {
    Column {
        items.forEach { Text(it) }
    }
}
```

---

## 5. `@Stable` and `@Immutable`: telling the compiler the truth

When you know a type really is safe, you can annotate it explicitly so Compose trusts it:

- **`@Immutable`**: promises that once the instance is created, no public property will ever change.
- **`@Stable`**: a softer promise - properties can change, but if they do, Compose finds out (for example because they're backed by `State`), and `equals()` remains trustworthy.

```kotlin
@Immutable
data class UiState(val items: List<String>, val isLoading: Boolean)
```

**The catch:** annotating `UiState` as `@Immutable` doesn't make the `List<String>` inside it stable for other uses - the annotation applies to that specific class, it doesn't transform the `List` type in general. For truly immutable, stable collections, use `kotlinx.collections.immutable`:

```kotlin
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.persistentListOf

data class UiState(val items: ImmutableList<String> = persistentListOf())
```

`ImmutableList` is stable because it's an interface that, by design, has no possible mutable implementation - unlike `List`, which can have one.

---

## 6. Strong skipping mode (Compose Compiler 2.0+)

Before **strong skipping mode**, a composable with unstable parameters completely lost the ability to skip - any single unstable parameter "contaminated" the whole decision.

With the Compose Compiler from Kotlin 2.0 onward, **strong skipping mode** is enabled by default and changes this rule: even a composable with unstable parameters can skip recomposition, comparing by **identity** instead of structural equality when stability can't be guaranteed. It also automatically "remembers" (memoizes) lambdas without needing to manually wrap them in `remember`.

In practice, this reduces - but doesn't eliminate - the importance of marking types as stable; using immutable collections and `data class`es for composable parameters remains good practice.

---

## 7. The three Compose phases

Compose processes each frame in three sequential phases:

1. **Composition**: runs the `@Composable` code and builds a description tree of the UI (what to show).
2. **Layout**: measures and positions each element (size and position on screen).
3. **Drawing**: paints pixels on screen according to what layout decided.

This matters because **reading a `State` in a different phase changes how much work gets repeated**. If state is read in composition, changing its value forces all three phases to run again. If it's read only in layout or drawing, you can skip the (more expensive) composition phase and repeat only the phase that's actually needed.

```kotlin
// Read in COMPOSITION: changing "offsetX" recomposes, relayouts, and redraws
Box(modifier = Modifier.offset(x = offsetX.dp))

// Read in LAYOUT: changing "offsetX" only relayouts and redraws, NO recomposition
Box(modifier = Modifier.offset { IntOffset(offsetX.roundToInt(), 0) })

// Read in DRAWING: changing "alpha" only redraws, NO recomposition or relayout
Box(modifier = Modifier.graphicsLayer { alpha = currentAlpha })
```

The lambda version of `offset {}` and using `graphicsLayer {}` are exactly the idiomatic ways to animate properties (position, scale, alpha, rotation) without paying the cost of a full recomposition on every animation frame - heavily used in high-performance animations.

---

## Summary - quick decision table

| Situation | What to do |
|---|---|
| A composable receives a `List` parameter and complains about stability | Use `ImmutableList`/`persistentListOf` from `kotlinx.collections.immutable` |
| A UI state `data class` that's fully immutable inside | Annotate it with `@Immutable` to help the compiler |
| Animating offset/scale/alpha on every frame | Read the value in layout (`offset {}`) or drawing (`graphicsLayer {}`), not in composition |
| A composable keeps recomposing for no apparent reason | Check whether a parameter is an unstable type (e.g. `List` instead of `ImmutableList`) |

---

## Interview phrases

- "Recomposition is granular: Compose only reruns the composables whose read state changed, not the whole tree."
- "Skipping requires stability: if Compose can't trust equals(), it can't safely skip recomposition."
- "List is unstable by definition because it's an interface that could have a mutable implementation; ImmutableList is stable."
- "Reading state in composition forces all three phases; reading it in layout or drawing lets you skip composition."

## Common interview traps

**"Why doesn't a composable with a List<T> parameter skip even though the list never changes?"** - Because List is an interface that could have a mutable implementation; the Compose compiler can't guarantee its stability and conservatively marks it unstable, even though in that particular case it never mutates.

**"What's the difference between @Stable and @Immutable?"** - @Immutable promises no public property will ever change. @Stable is more flexible: properties can change, but Compose finds out about those changes (e.g. via State) and equals() remains trustworthy.

**"Why use Modifier.offset { } instead of Modifier.offset(x.dp) to animate?"** - Because the lambda version reads the value in the layout phase, not composition, so animating that property doesn't force a full recomposition every frame - it only relayouts and redraws.
