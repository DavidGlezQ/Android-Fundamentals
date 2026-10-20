# Responsive Design / Diseño responsivo

[Versión en español](15-compose-responsive.es.md)

---

## 1. The philosophy: adapt to available space, not the device

The classic mistake is thinking in terms of "if it's a phone, do X; if it's a tablet, do Y." The right approach in Compose is to think in terms of **the space available at that moment**, not the device type - because the same phone can be in multi-window mode, in split-screen, or be a foldable that's open or closed, and suddenly "phone" tells you nothing reliable about how much real space you actually have.

```kotlin
// BAD: assumes device type determines layout
if (isTablet()) {
    TwoPaneLayout()
} else {
    SinglePaneLayout()
}

// GOOD: decides based on space available right now
BoxWithConstraints {
    if (maxWidth >= 600.dp) {
        TwoPaneLayout()
    } else {
        SinglePaneLayout()
    }
}
```

---

## 2. `BoxWithConstraints`: component-level adaptation

`BoxWithConstraints` gives access to the measurement constraints (`maxWidth`, `maxHeight`, etc.) available to that specific composable, letting you decide the layout internally based on the actual space that component has - useful when the same component is reused in contexts with different space (e.g. inside a dialog vs full screen).

```kotlin
@Composable
fun AdaptiveCard(content: @Composable () -> Unit) {
    BoxWithConstraints {
        if (maxWidth < 400.dp) {
            Column { content() }   // stacked vertically if space is tight
        } else {
            Row { content() }      // in a row if there's enough space
        }
    }
}
```

---

## 3. `Modifier.weight` and `fillMaxWidth(fraction)`: flexible proportions

To distribute space proportionally inside a `Row`/`Column`, without hardcoding fixed sizes:

```kotlin
Row(modifier = Modifier.fillMaxWidth()) {
    Box(modifier = Modifier.weight(1f).background(Color.Red))    // takes 1/3
    Box(modifier = Modifier.weight(2f).background(Color.Green))  // takes 2/3
}
```

`fillMaxWidth(fraction)` is used to make an element take a specific fraction of the available width (e.g. `0.6f` for 60%), commonly used for detail panels that shouldn't take up the whole screen even in large spaces.

---

## 4. Window Size Classes: full-screen-level decisions

While `BoxWithConstraints` solves adaptation for a single component, **Window Size Classes** (from the `androidx.compose.material3.windowsizeclass` / `androidx.window` library) categorize the full window size into three standard buckets, meant for navigation and overall screen-structure decisions:

| Class | Width | Typical example |
|---|---|---|
| **Compact** | < 600dp | Phone in portrait |
| **Medium** | 600dp - 840dp | Small tablet, phone in landscape, folded foldable |
| **Expanded** | > 840dp | Large tablet, unfolded foldable, desktop window |

```kotlin
val windowSizeClass = calculateWindowSizeClass(activity)

when (windowSizeClass.widthSizeClass) {
    WindowWidthSizeClass.Compact -> CompactLayout()
    WindowWidthSizeClass.Medium -> MediumLayout()
    WindowWidthSizeClass.Expanded -> ExpandedLayout()
}
```

**The practical difference from `BoxWithConstraints`:** Window Size Class is calculated once per window and used for big structural decisions (which navigation type to show, how many panes); `BoxWithConstraints` is evaluated locally, component by component, for finer-grained adaptations.

---

## 5. Adaptive navigation: `NavigationBar` → `NavigationRail` → `PermanentNavigationDrawer`

Material 3's standard pattern for navigation is to swap the navigation component as available space grows:

```kotlin
when (windowSizeClass.widthSizeClass) {
    WindowWidthSizeClass.Compact -> {
        // bottom bar, takes up little vertical space
        Scaffold(bottomBar = { NavigationBar { /* items */ } }) { /* content */ }
    }
    WindowWidthSizeClass.Medium -> {
        // compact side rail, leaves more horizontal space for content
        Row {
            NavigationRail { /* items */ }
            /* content */
        }
    }
    WindowWidthSizeClass.Expanded -> {
        // permanent drawer, always visible, uses the extra horizontal space
        Row {
            PermanentNavigationDrawer(drawerContent = { /* items */ }) { /* content */ }
        }
    }
}
```

The logic: on narrow screens, navigation can't "steal" horizontal space from the content, so it goes at the bottom (`NavigationBar`). As more width becomes available, navigation moves to a side, first as a compact rail and then as a full-width permanent drawer.

---

## 6. The list-detail pattern

A very common use case: a list of items and the detail of one of them. On compact space, only one can be shown at a time (navigating between the two); on expanded space, both can be shown simultaneously, side by side.

```kotlin
when (windowSizeClass.widthSizeClass) {
    WindowWidthSizeClass.Compact -> {
        // one screen at a time, push/pop navigation between list and detail
        if (selectedItem == null) ListScreen(onItemClick = { selectedItem = it })
        else DetailScreen(selectedItem, onBack = { selectedItem = null })
    }
    else -> {
        // two panes simultaneously, no navigation needed
        Row {
            ListScreen(modifier = Modifier.weight(1f), onItemClick = { selectedItem = it })
            DetailScreen(selectedItem, modifier = Modifier.weight(2f))
        }
    }
}
```

For this specific pattern, Google maintains the **`material3-adaptive`** library, which provides `ListDetailPaneScaffold` - a component that automatically handles the transition between one and two columns, including animations and navigation-state management between list and detail, without having to write that logic by hand.

---

## 7. A brief mention: foldables

Foldable devices add one more variable: the **hinge** can split the screen into two physical regions, even within the same Window Size Class. The `androidx.window` library exposes information about the fold state (`FoldingFeature`) so you can, for example, avoid placing important content right over the hinge, or intentionally use it as a natural divider in a two-pane layout.

---

## Summary - quick decision table

| I need... | Tool |
|---|---|
| Adapt a specific component to its available space | `BoxWithConstraints` |
| Distribute space proportionally in a Row/Column | `Modifier.weight`, `fillMaxWidth(fraction)` |
| Decide full-screen structure (navigation, panes) | Window Size Classes |
| Switch navigation type based on width | `NavigationBar` → `NavigationRail` → `PermanentNavigationDrawer` |
| Show list and detail in one or two columns depending on space | List-detail pattern, or `ListDetailPaneScaffold` from `material3-adaptive` |
| Account for a foldable's hinge | `androidx.window` (`FoldingFeature`) |

---

## Interview phrases

- "Responsive design in Compose is based on available space, not device type - the same phone can be in multi-window or split-screen."
- "BoxWithConstraints adapts a specific component; Window Size Classes decides full-screen structure."
- "The standard adaptive navigation pattern is NavigationBar on compact, NavigationRail on medium, and a permanent drawer on expanded."
- "material3-adaptive provides ListDetailPaneScaffold so you don't reimplement the one-to-two-column transition by hand."

## Common interview traps

**"How do you decide whether to show a one-column or two-column layout?"** - Based on the window's available width (Window Size Class), not on whether the device is a "phone" or "tablet" - a phone in landscape or multi-window can fall into Medium or Expanded just like a tablet.

**"What's the difference between BoxWithConstraints and Window Size Classes?"** - BoxWithConstraints is evaluated at a specific component's level and is useful for local adaptations; Window Size Classes is calculated once for the whole window and used for big structural decisions like which navigation type to show.

**"Why not hardcode `if (isTablet())` to decide the layout?"** - Because the device's physical size doesn't reflect the actual space available at that moment (multi-window, split-screen, folded/unfolded foldable); the decision should be based on the window's current dimensions.
