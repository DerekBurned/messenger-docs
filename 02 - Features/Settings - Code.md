# Settings - Code

Companion to [[Settings]]. The source for the settings feature's logic. The settings sub-screens
are mostly presentational Compose (navigated to from `SettingsScreen`; see [[Navigation]]); the
one piece with real state logic is the persisted theme, shown here in full. Snippets are verbatim
from the repo.

Files covered:
- `presentation/theme/ThemeMode.kt`
- `data/local/prefs/ThemePreferenceStore.kt`
- `presentation/screens/settings/appearance/AppearanceViewModel.kt`

---

## 1. ThemeMode

`presentation/theme/ThemeMode.kt`:

```kotlin
enum class ThemeMode {
    SYSTEM,
    LIGHT,
    DARK,
}
```

---

## 2. ThemePreferenceStore (DataStore)

`data/local/prefs/ThemePreferenceStore.kt` in full:

```kotlin
private val Context.themeDataStore by preferencesDataStore(name = "theme_prefs")

@Singleton
class ThemePreferenceStore @Inject constructor(
    @ApplicationContext private val context: Context,
) {
    private val themeModeKey = stringPreferencesKey("theme_mode")

    val themeMode: Flow<ThemeMode> = context.themeDataStore.data.map { prefs ->
        prefs[themeModeKey]
            ?.let { runCatching { ThemeMode.valueOf(it) }.getOrNull() }
            ?: ThemeMode.SYSTEM
    }

    suspend fun setThemeMode(mode: ThemeMode) {
        context.themeDataStore.edit { prefs -> prefs[themeModeKey] = mode.name }
    }
}
```

`themeMode` is a cold `Flow<ThemeMode>` off the DataStore, so the UI re-themes the moment the
preference changes. Unknown / legacy stored values fall back to `SYSTEM` via the `runCatching`.
`preferencesDataStore` is a property delegate, so there is exactly one DataStore instance per
process. Writes go through `edit`.

---

## 3. AppearanceViewModel

`presentation/screens/settings/appearance/AppearanceViewModel.kt` in full:

```kotlin
@HiltViewModel
class AppearanceViewModel @Inject constructor(
    private val themePreferenceStore: ThemePreferenceStore,
) : ViewModel() {

    val themeMode: StateFlow<ThemeMode> = themePreferenceStore.themeMode
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), ThemeMode.SYSTEM)

    fun setDarkTheme(enabled: Boolean) {
        viewModelScope.launch {
            themePreferenceStore.setThemeMode(if (enabled) ThemeMode.DARK else ThemeMode.LIGHT)
        }
    }
}
```

`stateIn(viewModelScope, WhileSubscribed(5_000), SYSTEM)` converts the cold DataStore flow into a
hot `StateFlow` with an initial value for the first frame; `WhileSubscribed(5_000)` keeps the
upstream reader alive for 5s after the last subscriber leaves, so a config change does not
re-open the DataStore reader. `setDarkTheme` writes back on `viewModelScope`. The applied theme is
consumed at the top of the Compose tree by `MessengerTheme`
(`presentation/theme/Theme.kt`), which maps `ThemeMode` to a Material 3 light/dark
color scheme. Why each choice was made: [[Settings]].

---

## 4. The rest of settings

The other settings areas are primarily Compose screens under `presentation/screens/settings/`
and their small ViewModels:

- `PrivacyScreen` - phone-number visibility (`PhoneVisibility` on the profile).
- `SecurityScreen` - E2EE safety number via `SafetyNumberViewModel` / `SafetyNumberSection`; its
  code is in [[End-to-End Encryption - Code]].
- `NotificationsScreen`, `DataStorageScreen` (`MediaCache`), `LanguageScreen`.
- Profile editing: `ProfileViewModel`, `EditProfileViewModel`, `ChangeAccountViewModel`,
  `ChangePhoneViewModel` (the last reuses the phone-verification code from
  [[Authentication - Code]]).

These are conventional Compose + `StateFlow` screens; the theme flow above is the representative
piece of persisted-settings logic.

---

## Related

[[Settings]] - [[End-to-End Encryption - Code]] - [[Authentication - Code]] -
[[Async and Reactivity]] - [[Presentation Layer (MVI)]] - [[Navigation]]
