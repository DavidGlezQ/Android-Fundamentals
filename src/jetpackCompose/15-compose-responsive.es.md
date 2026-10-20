# Responsive Design / Diseño responsivo

[English version](../../../../Downloads/15-compose-responsive.en.md)

---

## 1. La filosofía: adaptarse al espacio disponible, no al dispositivo

El error clásico es pensar en términos de "si es teléfono, hago X; si es tablet, hago Y". El enfoque correcto en Compose es pensar en términos del **espacio disponible** en ese momento, no del tipo de dispositivo - porque el mismo teléfono puede estar en modo multi-ventana, con split-screen, o ser un foldable abierto o cerrado, y de repente "teléfono" no te dice nada confiable sobre cuánto espacio real tienes.

```kotlin
// MAL: asume que el tipo de dispositivo determina el layout
if (isTablet()) {
    TwoPaneLayout()
} else {
    SinglePaneLayout()
}

// BIEN: decide según el espacio disponible en ese momento
BoxWithConstraints {
    if (maxWidth >= 600.dp) {
        TwoPaneLayout()
    } else {
        SinglePaneLayout()
    }
}
```

---

## 2. `BoxWithConstraints`: adaptación a nivel de componente

`BoxWithConstraints` da acceso a las restricciones de medida (`maxWidth`, `maxHeight`, etc.) disponibles para ese composable específico, permitiendo decidir el layout internamente según el espacio real que tiene ese componente - útil cuando el mismo componente se reutiliza en contextos con distinto espacio (ej. dentro de un diálogo vs a pantalla completa).

```kotlin
@Composable
fun AdaptiveCard(content: @Composable () -> Unit) {
    BoxWithConstraints {
        if (maxWidth < 400.dp) {
            Column { content() }   // apilado vertical si hay poco espacio
        } else {
            Row { content() }      // en fila si hay suficiente espacio
        }
    }
}
```

---

## 3. `Modifier.weight` y `fillMaxWidth(fraction)`: proporciones flexibles

Para repartir espacio de forma proporcional dentro de un `Row`/`Column`, sin hardcodear tamaños fijos:

```kotlin
Row(modifier = Modifier.fillMaxWidth()) {
    Box(modifier = Modifier.weight(1f).background(Color.Red))    // ocupa 1/3
    Box(modifier = Modifier.weight(2f).background(Color.Green))  // ocupa 2/3
}
```

`fillMaxWidth(fraction)` sirve para que un elemento ocupe una fracción específica del ancho disponible (ej. `0.6f` para el 60%), muy usado para paneles de detalle que no deben ocupar toda la pantalla incluso en espacios grandes.

---

## 4. Window Size Classes: decisiones a nivel de pantalla completa

Mientras `BoxWithConstraints` resuelve adaptación a nivel de un componente puntual, las **Window Size Classes** (de la librería `androidx.compose.material3.windowsizeclass` / `androidx.window`) categorizan el tamaño de ventana completo en tres buckets estándar, pensados para decisiones de navegación y estructura general de pantalla:

| Clase | Ancho | Ejemplo típico |
|---|---|---|
| **Compact** | < 600dp | Teléfono en vertical |
| **Medium** | 600dp - 840dp | Tablet pequeña, teléfono en horizontal, foldable plegado |
| **Expanded** | > 840dp | Tablet grande, foldable abierto, ventana de escritorio |

```kotlin
val windowSizeClass = calculateWindowSizeClass(activity)

when (windowSizeClass.widthSizeClass) {
    WindowWidthSizeClass.Compact -> CompactLayout()
    WindowWidthSizeClass.Medium -> MediumLayout()
    WindowWidthSizeClass.Expanded -> ExpandedLayout()
}
```

**La diferencia práctica con `BoxWithConstraints`:** Window Size Classes se calcula una sola vez por ventana y se usa para decisiones estructurales grandes (qué tipo de navegación mostrar, cuántos paneles); `BoxWithConstraints` se evalúa localmente, componente por componente, para adaptaciones más finas.

---

## 5. Navegación adaptativa: `NavigationBar` → `NavigationRail` → `PermanentNavigationDrawer`

El patrón estándar de Material 3 para navegación es ir cambiando el componente de navegación según crece el espacio disponible:

```kotlin
when (windowSizeClass.widthSizeClass) {
    WindowWidthSizeClass.Compact -> {
        // barra inferior, ocupa poco espacio vertical
        Scaffold(bottomBar = { NavigationBar { /* items */ } }) { /* content */ }
    }
    WindowWidthSizeClass.Medium -> {
        // rail lateral compacto, deja más espacio horizontal al contenido
        Row {
            NavigationRail { /* items */ }
            /* content */
        }
    }
    WindowWidthSizeClass.Expanded -> {
        // drawer permanente, visible siempre, usa el espacio horizontal extra
        Row {
            PermanentNavigationDrawer(drawerContent = { /* items */ }) { /* content */ }
        }
    }
}
```

La lógica: en pantallas angostas, la navegación no puede "robarle" espacio horizontal al contenido, así que va abajo (`NavigationBar`). A medida que hay más ancho disponible, la navegación se mueve a un costado, primero como rail compacto y después como drawer permanente de ancho completo.

---

## 6. El patrón list-detail

Un caso de uso muy común: una lista de elementos y el detalle de uno de ellos. En espacio compacto, solo se puede mostrar uno a la vez (navegando entre ambos); en espacio expandido, se pueden mostrar los dos simultáneamente, lado a lado.

```kotlin
when (windowSizeClass.widthSizeClass) {
    WindowWidthSizeClass.Compact -> {
        // una sola pantalla a la vez, navegación push/pop entre lista y detalle
        if (selectedItem == null) ListScreen(onItemClick = { selectedItem = it })
        else DetailScreen(selectedItem, onBack = { selectedItem = null })
    }
    else -> {
        // dos paneles simultáneos, sin necesidad de navegación
        Row {
            ListScreen(modifier = Modifier.weight(1f), onItemClick = { selectedItem = it })
            DetailScreen(selectedItem, modifier = Modifier.weight(2f))
        }
    }
}
```

Para este patrón específico, Google mantiene la librería **`material3-adaptive`**, que provee `ListDetailPaneScaffold` - un componente que maneja automáticamente la transición entre una y dos columnas, incluyendo las animaciones y el manejo del estado de navegación entre lista y detalle, sin tener que escribir esa lógica a mano.

---

## 7. Una mención breve: foldables

Los dispositivos plegables añaden una variable más: la **bisagra** (hinge) puede partir la pantalla en dos regiones físicas, incluso dentro de la misma Window Size Class. La librería `androidx.window` expone información sobre el estado del plegado (`FoldingFeature`) para, por ejemplo, evitar poner contenido importante justo sobre la bisagra, o usarla intencionalmente como divisor natural en un layout de dos paneles.

---

## Resumen - tabla de decisión rápida

| Necesito... | Herramienta |
|---|---|
| Adaptar un componente puntual al espacio que tiene disponible | `BoxWithConstraints` |
| Repartir espacio proporcionalmente en una Row/Column | `Modifier.weight`, `fillMaxWidth(fraction)` |
| Decidir estructura de pantalla completa (navegación, paneles) | Window Size Classes |
| Cambiar el tipo de navegación según el ancho | `NavigationBar` → `NavigationRail` → `PermanentNavigationDrawer` |
| Mostrar lista y detalle en una o dos columnas según el espacio | Patrón list-detail, o `ListDetailPaneScaffold` de `material3-adaptive` |
| Tener en cuenta la bisagra de un foldable | `androidx.window` (`FoldingFeature`) |

---

## Frases para entrevista

- "El diseño responsivo en Compose se basa en el espacio disponible, no en el tipo de dispositivo - el mismo teléfono puede estar en multi-ventana o split-screen."
- "BoxWithConstraints adapta un componente puntual; Window Size Classes decide la estructura de pantalla completa."
- "El patrón estándar de navegación adaptativa es NavigationBar en compact, NavigationRail en medium, y un drawer permanente en expanded."
- "material3-adaptive provee ListDetailPaneScaffold para no reimplementar a mano la transición entre una y dos columnas."

## Trampas típicas de entrevista

**"¿Cómo decides si mostrar un layout de una o dos columnas?"** - Según el ancho disponible de la ventana (Window Size Class), no según si el dispositivo es "teléfono" o "tablet" - un teléfono en landscape o multi-ventana puede caer en Medium o Expanded igual que una tablet.

**"¿Cuál es la diferencia entre BoxWithConstraints y Window Size Classes?"** - BoxWithConstraints se evalúa a nivel de un componente específico y es útil para adaptaciones locales; Window Size Classes se calcula una vez para toda la ventana y se usa para decisiones estructurales grandes como qué tipo de navegación mostrar.

**"¿Por qué no hardcodear `if (isTablet())` para decidir el layout?"** - Porque el tamaño físico del dispositivo no refleja el espacio real disponible en ese momento (multi-ventana, split-screen, foldable plegado/desplegado); hay que decidir según las dimensiones actuales de la ventana.
