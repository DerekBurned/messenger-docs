# Appearance and Theming - Basic Example

The inherited pattern behind [[Appearance and Theming]], in its plain textbook form - not the real
`MessengerTheme`/`ThemePreferenceStore`/`WallpaperBackdrop`, but the three ideas they specialise.
Then read [[Appearance and Theming - Code]] for the real source.

(Teaching comments only; the real source has none.)

---

## What is inherited

Theming is three base ideas: a **reactive preference** (DataStore Flow -> `MaterialTheme`, so a
setting change re-themes the tree), a **`CompositionLocal`** carrying app-specific design tokens
alongside the Material scheme, and a **`Modifier.composed`** animation modifier (the reusable
per-widget motion in `MotionModifiers`). The preference half is the same DataStore pattern as
[[Settings - Basic Example]].

## The basic example (reactive theme + tokens local + a composed modifier)

```kotlin
// 1) A reactive theme: read the persisted mode, decide dark/light, provide it to the tree.
@Composable
fun AppTheme(mode: ThemeMode, content: @Composable () -> Unit) {
    val dark = when (mode) {
        ThemeMode.SYSTEM -> isSystemInDarkTheme()
        ThemeMode.LIGHT  -> false
        ThemeMode.DARK   -> true
    }
    CompositionLocalProvider(LocalTokens provides if (dark) DarkTokens else LightTokens) {
        MaterialTheme(colorScheme = if (dark) DarkScheme else LightScheme, content = content)
    }
}

// The root collects the persisted mode and passes it in; changing the setting re-themes everything.
setContent {
    val mode by store.themeMode.collectAsState(ThemeMode.SYSTEM)
    AppTheme(mode) { App() }
}

// 2) App tokens travel through a CompositionLocal, read anywhere as `tokens`.
val LocalTokens = staticCompositionLocalOf { LightTokens }
val tokens: AppTokens @Composable get() = LocalTokens.current

// 3) A reusable motion modifier: press-to-shrink, spring back. composed{} gives it its own state.
fun Modifier.pressScale(interaction: InteractionSource, pressed: Float = 0.95f): Modifier = composed {
    val isPressed by interaction.collectIsPressedAsState()
    val scale by animateFloatAsState(if (isPressed) pressed else 1f, spring())
    graphicsLayer { scaleX = scale; scaleY = scale }
}
```

## How the real code specialises it

- The theme composable is `MessengerTheme` (`presentation/theme/Theme.kt:44`), fed the
  persisted mode, text size and font from the composition root (`MainActivity.kt:100`); it
  provides `LocalMessengerTokens` (read as `messengerTokens`) and `LocalContentColor`, so plain
  `Text` is readable in both themes.
- The reactive preference is `ThemePreferenceStore` (`data/local/prefs/ThemePreferenceStore.kt:27`),
  holding theme mode, a background key, an intensity float (with a live-preview overlay), bar
  position, text size, font and the list of imported wallpapers.
- The composed modifiers are `pressScale` and `springEnter` in `MotionModifiers.kt`, built on the
  shared `MotionSprings` specs (`presentation/components/common/model/MotionSprings.kt:6`).
- The wallpaper layer (`WallpaperBackdrop`, `WallpaperBackground.kt:34`) reads the store through a
  manual Hilt `@EntryPoint` (`UiEntryPoints`, `di/UiEntryPoints.kt:13`) because it is a composable,
  not a ViewModel - full detail in [[Appearance and Theming]].

## Related

[[Appearance and Theming]] - [[Appearance and Theming - Code]] -
[[Appearance and Theming - Explained Simply]] - [[Settings - Basic Example]] -
[[Reading the Code - Simple Patterns]] - [[Home]]
