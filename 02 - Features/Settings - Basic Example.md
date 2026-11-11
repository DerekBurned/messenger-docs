# Settings - Basic Example

The inherited pattern behind [[Settings]] (the persisted theme preference), in its plain textbook
form - not the real `ThemePreferenceStore`/`AppearanceViewModel`, but the DataStore idea they
specialise. Then read [[Settings - Code]] for the real source.

(Teaching comments only; the real source has none.)

---

## What is inherited

A durable setting is Jetpack **DataStore Preferences**: a key/value store that reads back as a
cold `Flow` (it re-emits on every write) and writes through `edit { }`. The one app pattern on top
is **`stateIn`**, which turns that cold flow into a `StateFlow` so a screen has a value on its
first frame.

## The basic example (canonical DataStore preference as reactive state)

```kotlin
// One DataStore for the app, addressed by typed keys.
private val Context.dataStore by preferencesDataStore(name = "settings")
private val THEME = stringPreferencesKey("theme_mode")

class SettingsStore(private val context: Context) {
    // Read: a cold Flow that emits the current value and every change after it.
    val themeMode: Flow<ThemeMode> = context.dataStore.data.map { prefs ->
        prefs[THEME]?.let { runCatching { ThemeMode.valueOf(it) }.getOrNull() } ?: ThemeMode.SYSTEM
    }
    // Write: edit {} suspends, persists, and causes the Flow above to re-emit.
    suspend fun setThemeMode(mode: ThemeMode) {
        context.dataStore.edit { it[THEME] = mode.name }
    }
}

// Turn the cold Flow into UI state with a current value on the first frame.
val theme: StateFlow<ThemeMode> = store.themeMode
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), ThemeMode.SYSTEM)
```

Why `WhileSubscribed(5_000)`: keep the DataStore reader alive for 5s after the last subscriber, so
a recomposition or rotation does not re-open it, while still stopping when the screen is truly gone.

## How the real code specialises it

- The store is `ThemePreferenceStore` (`data/local/prefs/ThemePreferenceStore.kt:27`), a
  `@Singleton` over `preferencesDataStore("theme_prefs")`; the `runCatching` fallback to `SYSTEM`
  is exactly `themeMode` (`:40`).
- `AppearanceViewModel` (`presentation/screens/settings/appearance/AppearanceViewModel.kt:28`) does the `stateIn`.
- The store holds more settings than the theme (background, intensity, bar position, text size,
  font, saved wallpapers) and layers a live-preview overlay on intensity - see
  [[Appearance and Theming - Basic Example]] and [[Appearance and Theming]].
- Other settings areas (privacy, account deletion, security/safety numbers, notifications,
  storage, language) hang off the same settings hub - see [[Settings]].

## Related

[[Settings]] - [[Settings - Code]] - [[Settings - Explained Simply]] -
[[Appearance and Theming]] - [[Reading the Code - Simple Patterns]] - [[Async and Reactivity]] -
[[Home]]
