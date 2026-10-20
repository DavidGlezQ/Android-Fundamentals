# Recomposition & Stability / Recomposición y estabilidad

[English version](14-compose-recomposicion.en.md)

---

## 1. Qué es la recomposición

**Recomposición** es cuando Compose vuelve a ejecutar el código de un composable (o una parte de él) porque alguno de los `State` que lee cambió de valor. No es "redibujar todo desde cero" - Compose intenta ejecutar solo lo necesario, en el alcance más pequeño posible.

```kotlin
@Composable
fun ProfileScreen(name: String, age: Int) {
    Column {
        Text(name)   // solo se recompone si "name" cambia
        Text("$age años")   // solo se recompone si "age" cambia
    }
}
```

Si `age` cambia pero `name` no, Compose solo vuelve a ejecutar el `Text(age)`, no el `Text(name)` ni el `Column` entero. Esto se llama **recomposición granular** o **smart recomposition**.

---

## 2. Scope de recomposición: el alcance más pequeño

Cada lectura de `State` dentro de un composable crea un punto donde Compose puede "cortar" la recomposición. El **scope de recomposición** es la región de código que se vuelve a ejecutar cuando ese estado cambia - normalmente el composable más interno que lee ese valor, no el árbol completo.

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Column {
        Text("Counter app")            // nunca se recompone, no lee "count"
        Text("Count: $count")          // se recompone cada vez que "count" cambia
        Button(onClick = { count++ }) { Text("+1") }
    }
}
```

El primer `Text` nunca se recompone porque no lee `count` - Compose sabe, por inferencia estática, qué composables leen qué estado.

---

## 3. Skipping: por qué Compose puede saltarse una recomposición

Compose puede **saltarse** (skip) la ejecución de un composable si sus parámetros de entrada son iguales a la última vez que se ejecutó. Esto requiere que esos parámetros sean **estables** - si Compose no puede garantizar que un parámetro es estable, no puede confiar en comparar su valor y termina recomponiendo "por si acaso".

```kotlin
@Composable
fun Greeting(name: String) {   // String es estable: Compose puede comparar y saltar si no cambió
    Text("Hola, $name")
}
```

---

## 4. Estabilidad: qué hace que un tipo sea estable o no

Un tipo es **estable** si Compose puede confiar en que, cuando su `equals()` dice que dos instancias son iguales, de verdad lo son y seguirán siéndolo (el tipo es inmutable, o si es mutable notifica sus cambios vía `State`).

**Son estables por defecto:** todos los tipos primitivos (`Int`, `Boolean`, `String`, etc.), las lambdas (si capturan solo valores estables), y las `data class` cuyo contenido es inmutable.

**No son estables por defecto:** las interfaces de colección como `List`, `Map`, `Set`. Aunque en la práctica se les pase una lista inmutable, el *tipo* `List` es una interfaz que *podría* tener una implementación mutable por debajo (como `MutableList`), así que el compilador de Compose no puede garantizar su estabilidad y la marca como inestable.

```kotlin
// "items: List<String>" es inestable para el compilador de Compose,
// aunque en este caso en particular nunca mute - Compose no puede saberlo
@Composable
fun ItemList(items: List<String>) {
    Column {
        items.forEach { Text(it) }
    }
}
```

---

## 5. `@Stable` y `@Immutable`: decirle al compilador la verdad

Cuando se sabe que un tipo sí es seguro, se le puede anotar explícitamente para que Compose confíe en él:

- **`@Immutable`**: promete que una vez creada la instancia, ninguna propiedad pública cambiará jamás.
- **`@Stable`**: promesa más suave - las propiedades pueden cambiar, pero si cambian, Compose se entera (por ejemplo, porque están respaldadas por `State`), y `equals()` sigue siendo confiable.

```kotlin
@Immutable
data class UiState(val items: List<String>, val isLoading: Boolean)
```

**El problema:** anotar `UiState` como `@Immutable` no vuelve estable el `List<String>` de adentro para otros usos - la anotación aplica a esa clase específica, no transforma el tipo `List` en general. Para colecciones de verdad inmutables y estables, se usa `kotlinx.collections.immutable`:

```kotlin
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.persistentListOf

data class UiState(val items: ImmutableList<String> = persistentListOf())
```

`ImmutableList` sí es estable porque es una interfaz que, por diseño, no tiene ninguna implementación mutable posible - a diferencia de `List`, que sí puede serlo.

---

## 6. Strong skipping mode (Compose Compiler 2.0+)

Antes del **strong skipping mode**, un composable con parámetros inestables perdía por completo la capacidad de hacer skip - cualquier parámetro inestable "contaminaba" la decisión entera.

Con el Compose Compiler de Kotlin 2.0 en adelante, el **strong skipping mode** está activado por defecto y cambia esta regla: incluso un composable con parámetros inestables puede saltarse la recomposición, comparando por **identidad** en vez de por igualdad estructural cuando la estabilidad no se puede garantizar. También hace que las lambdas sean automáticamente "recordadas" (memoizadas) sin necesidad de envolverlas manualmente en `remember`.

En la práctica, esto reduce - pero no elimina - la importancia de marcar tipos como estables; sigue siendo buena práctica usar colecciones inmutables y `data class` para los parámetros de los composables.

---

## 7. Las tres fases de Compose

Compose procesa cada frame en tres fases secuenciales:

1. **Composición**: ejecuta el código `@Composable` y construye un árbol de descripción de la UI (qué mostrar).
2. **Layout**: mide y posiciona cada elemento (tamaño y posición en pantalla).
3. **Dibujo (drawing)**: pinta los píxeles en pantalla según lo que decidió el layout.

Esto importa porque **leer un `State` en una fase distinta cambia cuánto trabajo se repite**. Si el estado se lee en composición, cambiar su valor fuerza a recorrer las tres fases otra vez. Si se lee solo en layout o en dibujo, se puede saltar la fase de composición (más cara) y repetir solo la fase necesaria.

```kotlin
// Leído en COMPOSICIÓN: cambiar "offsetX" recompone, relayoutea y redibuja
Box(modifier = Modifier.offset(x = offsetX.dp))

// Leído en LAYOUT: cambiar "offsetX" solo relayoutea y redibuja, SIN recomponer
Box(modifier = Modifier.offset { IntOffset(offsetX.roundToInt(), 0) })

// Leído en DIBUJO: cambiar "alpha" solo redibuja, SIN recomponer ni relayoutear
Box(modifier = Modifier.graphicsLayer { alpha = currentAlpha })
```

La versión lambda de `offset {}` y el uso de `graphicsLayer {}` son justo las formas idiomáticas de animar propiedades (posición, escala, alpha, rotación) sin pagar el costo de una recomposición completa en cada frame de la animación - algo muy usado en animaciones de alto rendimiento.

---

## Resumen - tabla de decisión rápida

| Situación | Qué hacer |
|---|---|
| Un composable recibe una `List` como parámetro y se queja de estabilidad | Usar `ImmutableList`/`persistentListOf` de `kotlinx.collections.immutable` |
| Una `data class` de UI state con todo inmutable adentro | Anotarla con `@Immutable` para ayudar al compilador |
| Animar offset/escala/alpha en cada frame | Leer el valor en layout (`offset {}`) o dibujo (`graphicsLayer {}`), no en composición |
| Un composable no deja de recomponerse sin razón aparente | Revisar si algún parámetro es un tipo inestable (ej. `List` en vez de `ImmutableList`) |

---

## Frases para entrevista

- "La recomposición es granular: Compose solo vuelve a ejecutar los composables cuyo estado leído cambió, no el árbol completo."
- "Skipping requiere estabilidad: si Compose no puede confiar en equals(), no puede saltarse la recomposición de forma segura."
- "List es inestable por definición porque es una interfaz que podría tener una implementación mutable; ImmutableList sí es estable."
- "Leer estado en composición fuerza las tres fases; leerlo en layout o dibujo permite saltarse la composición."

## Trampas típicas de entrevista

**"¿Por qué un composable con un parámetro List<T> no hace skip aunque la lista nunca cambie?"** - Porque List es una interfaz que podría tener una implementación mutable; el compilador de Compose no puede garantizar su estabilidad y por seguridad lo marca como inestable, aunque en ese caso particular nunca mute.

**"¿Qué diferencia hay entre @Stable y @Immutable?"** - @Immutable promete que ninguna propiedad pública cambiará nunca. @Stable es más flexible: las propiedades pueden cambiar, pero Compose se entera de esos cambios (por ejemplo vía State) y equals() sigue siendo confiable.

**"¿Por qué usar Modifier.offset { } en vez de Modifier.offset(x.dp) para animar?"** - Porque la versión lambda lee el valor en la fase de layout, no en composición, así que animar esa propiedad no fuerza una recomposición completa en cada frame - solo relayoutea y redibuja.
