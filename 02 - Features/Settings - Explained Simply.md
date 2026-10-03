# Settings - Explained Simply

The plain-language version of [[Settings]]. How the app remembers your theme and repaints itself
the instant you change it.

(Teaching comments only; the real source has none.)

---

## The big idea in one sentence

Your theme choice (System / Light / Dark) is saved to a tiny on-disk sticky note (DataStore); the
app watches that note as a live stream, so changing it repaints the whole app immediately - and it
is remembered next launch.

Analogy: a sticky note on the fridge that says "Dark". The app keeps glancing at it; the moment
you change the note, the app redecorates. The note is still there tomorrow.

---

## Step by step: switching to dark mode

1. On the Appearance screen you flip the dark switch -> `AppearanceViewModel.setDarkTheme(true)`
   (`presentation/viewmodel/AppearanceViewModel.kt:51`).

2. That writes `DARK` to the sticky note: `ThemePreferenceStore.setThemeMode(DARK)`
   (`data/local/prefs/ThemePreferenceStore.kt:79`).

3. The sticky note is a live stream. Because the app is watching it, the new value flows out
   immediately.

4. The theme at the very top of the app (`MessengerTheme`) receives the new value and re-colors
   every screen at once.

5. Next launch, the note still says `DARK`, so the app starts dark - no flash of the wrong theme.

---

## The code, with plain-English comments

The sticky note (`data/local/prefs/ThemePreferenceStore.kt:40`):

```kotlin
// Reading returns a LIVE stream: it emits again every time the value changes on disk.
val themeMode: Flow<ThemeMode> = context.themeDataStore.data.map { prefs ->
    // If the stored value is missing or unrecognizable, fall back to SYSTEM safely.
    prefs[themeModeKey]?.let { runCatching { ThemeMode.valueOf(it) }.getOrNull() } ?: ThemeMode.SYSTEM
}

suspend fun setThemeMode(mode: ThemeMode) {
    context.themeDataStore.edit { prefs -> prefs[themeModeKey] = mode.name } // write the note
}
```

Giving the screen a value ready on the first frame
(`presentation/viewmodel/AppearanceViewModel.kt:30`):

```kotlin
val themeMode: StateFlow<ThemeMode> = themePreferenceStore.themeMode
    // stateIn turns the live stream into "always has a current value" (starts at SYSTEM),
    // so the screen has something to show immediately, and keeps the reader alive 5s after
    // the last screen leaves (so a rotate doesn't re-open the file reader).
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), ThemeMode.SYSTEM)
```

---

## A worked example

- You open Appearance and turn on Dark. `setThemeMode(DARK)` writes the note.
- Because the app watches the note, `MessengerTheme` gets `DARK` and every screen recolors at
  once - no restart.
- You close and reopen the app tomorrow -> it reads `DARK` from the note and starts dark.

---

## Why, in one line each

- Why DataStore instead of plain saved settings? It hands back a live stream, so the UI recolors
  itself with no manual "refresh".
- Why `stateIn` with a starting value? So the screen has a value to render on frame one and does
  not re-open the file reader on every little recomposition.
- Why fall back to SYSTEM on a bad value? So a corrupt or old stored value never breaks theming.

---

## The other settings screens

Most other settings pages (Privacy, Notifications, Security, Data storage, Language, and the
profile editors) are ordinary screens that read a value and write it back the same way. The one
with real cryptography is Security's safety number - see
[[End-to-End Encryption - Explained Simply]].

---

## Related

[[Settings]] - [[Settings - Code]] - [[End-to-End Encryption - Explained Simply]] -
[[Reading the Code - Simple Patterns]] - [[Home]]
