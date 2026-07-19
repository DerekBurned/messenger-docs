# Appearance and Theming - Code

The real source behind [[Appearance and Theming]], verified against the repo and explained block by
block. The plain-language version is [[Appearance and Theming - Explained Simply]]; the minimal
inherited pattern is [[Appearance and Theming - Basic Example]].

(Teaching comments only; the real source has none - that is a project rule.)

---

## 1. The persisted settings (`ThemePreferenceStore.kt:17`)

Three settings live in one DataStore. Reads are cold `Flow`s; writes go through `edit { }`.

```kotlin
private val Context.themeDataStore by preferencesDataStore(name = "theme_prefs")

@Singleton
class ThemePreferenceStore @Inject constructor(
    @ApplicationContext private val context: Context,
) {
    private val themeModeKey  = stringPreferencesKey("theme_mode")
    private val backgroundKey = stringPreferencesKey("app_background")
    private val intensityKey  = floatPreferencesKey("theme_intensity")

    val themeMode: Flow<ThemeMode> = context.themeDataStore.data.map { prefs ->
        prefs[themeModeKey]
            ?.let { runCatching { ThemeMode.valueOf(it) }.getOrNull() }   // unknown/legacy -> null
            ?: ThemeMode.SYSTEM                                            // ...falls back to SYSTEM
    }

    val background: Flow<String> = context.themeDataStore.data.map { prefs ->
        prefs[backgroundKey] ?: "neutral"
    }
```

Intensity is special: a `liveIntensity` `MutableStateFlow` is `combine`d over the stored value so a
drag can preview instantly without a write:

```kotlin
    private val liveIntensity = MutableStateFlow<Float?>(null)

    val intensity: Flow<Float> = combine(
        context.themeDataStore.data.map { prefs -> prefs[intensityKey] ?: 1f },  // persisted
        liveIntensity,                                                            // live drag (or null)
    ) { stored, live -> live ?: stored }                                         // live wins if set

    fun previewIntensity(value: Float) { liveIntensity.value = value.coerceIn(0f, 1f) }  // no write

    suspend fun setIntensity(value: Float) {                                     // commit on release
        val clamped = value.coerceIn(0f, 1f)
        liveIntensity.value = clamped
        context.themeDataStore.edit { prefs -> prefs[intensityKey] = clamped }
    }
}
```

## 2. Cold flows -> UI state (`AppearanceViewModel.kt:21`)

`stateIn` gives the screen a value on its first frame; `WhileSubscribed(5_000)` keeps the DataStore
reader alive briefly across recompositions/rotation.

```kotlin
@HiltViewModel
class AppearanceViewModel @Inject constructor(
    @ApplicationContext private val context: Context,
    private val themePreferenceStore: ThemePreferenceStore,
) : ViewModel() {

    val themeMode: StateFlow<ThemeMode> = themePreferenceStore.themeMode
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), ThemeMode.SYSTEM)
    val background: StateFlow<String> = themePreferenceStore.background
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), "neutral")
    val intensity: StateFlow<Float> = themePreferenceStore.intensity
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), 1f)
```

Importing a wallpaper copies the picked file, drops any previous one, then flips the key
(`AppearanceViewModel.kt:57`):

```kotlin
    fun importWallpaper(uri: Uri) {
        viewModelScope.launch(Dispatchers.IO) {                     // file IO off the main thread
            runCatching {
                val dir = File(context.filesDir, "wallpapers").apply { mkdirs() }
                val timestamp = System.currentTimeMillis()
                val file = File(dir, "wallpaper_$timestamp.jpg")
                context.contentResolver.openInputStream(uri)?.use { input ->
                    file.outputStream().use { output -> input.copyTo(output) }
                } ?: return@runCatching
                dir.listFiles()?.filter { it.name != file.name }?.forEach { it.delete() }  // keep one
                themePreferenceStore.setBackground("$CUSTOM_BACKGROUND_PREFIX$timestamp")
            }
        }
    }
```

## 3. Applying the mode (`Theme.kt:41`)

`MessengerTheme` maps `ThemeMode` to dark/light, selects a `ColorScheme` and token set, and provides
the tokens through a `CompositionLocal`:

```kotlin
@Composable
fun MessengerTheme(themeMode: ThemeMode = ThemeMode.SYSTEM, content: @Composable () -> Unit) {
    val darkTheme = when (themeMode) {
        ThemeMode.SYSTEM -> isSystemInDarkTheme()
        ThemeMode.LIGHT  -> false
        ThemeMode.DARK   -> true
    }
    val colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme
    val tokens      = if (darkTheme) DarkTokens      else LightTokens
    CompositionLocalProvider(LocalMessengerTokens provides tokens) {
        MaterialTheme(colorScheme = colorScheme, typography = Typography,
            shapes = MessengerMaterialShapes, content = content)
    }
}
```

The root reads the persisted mode and feeds it in, so a write re-themes everything
(`MainActivity.kt:65`):

```kotlin
val themeMode by themePreferenceStore.themeMode.collectAsState(initial = ThemeMode.SYSTEM)
MessengerTheme(themeMode = themeMode) { /* app */ }
```

## 4. The backgrounds (`AppBackgrounds.kt:14`)

Six presets, each with dark and light gradient stops, plus the custom-wallpaper key helpers:

```kotlin
data class AppBackground(val key: String, val label: String,
    val darkColors: List<Color>, val lightColors: List<Color>)

val AppBackgrounds = listOf(
    AppBackground("neutral", "Neutral",
        darkColors = listOf(Color(0xFF1C1A20), Color(0xFF16141B)),
        lightColors = listOf(Color(0xFFEDE7F4), Color(0xFFD7CEE6))),
    /* aurora, plum, teal, midnight, mono ... */
)

fun appBackgroundFor(key: String?): AppBackground =
    AppBackgrounds.firstOrNull { it.key == key } ?: AppBackgrounds.first()

const val CUSTOM_BACKGROUND_PREFIX = "custom:"

fun customWallpaperFile(context: Context, backgroundKey: String): File? {
    val timestamp = backgroundKey.removePrefix(CUSTOM_BACKGROUND_PREFIX)
    if (timestamp == backgroundKey) return null                         // not a custom key
    return File(File(context.filesDir, "wallpapers"), "wallpaper_$timestamp.jpg")
}
```

## 5. Drawing the backdrop (`WallpaperBackground.kt:35`)

`WallpaperBackdrop` reads the store through a Hilt `@EntryPoint` (it is a composable, not a
ViewModel), paints the gradient, overlays a custom image if present, then applies the intensity
scrim:

```kotlin
val store = remember {
    EntryPointAccessors.fromApplication(context.applicationContext, UiEntryPoints::class.java)
        .themePreferenceStore()                                         // manual DI reach-in
}
val backgroundKey = store.background.collectAsState(initial = "neutral").value
val intensity     = store.intensity.collectAsState(initial = 1f).value
val preset = appBackgroundFor(backgroundKey)
val colors = if (tokens.isDark) preset.darkColors else preset.lightColors

Box(Modifier.fillMaxSize().background(Brush.verticalGradient(colors)))   // gradient
if (customFile != null) AsyncImage(model = customFile, contentScale = ContentScale.Crop, ...)  // image
val scrimAlpha = ((1f - intensity) * 0.85f).coerceIn(0f, 1f)             // dim
if (scrimAlpha > 0.005f) {
    val scrim = if (tokens.isDark) Color.Black else Color.White
    Box(Modifier.fillMaxSize().background(scrim.copy(alpha = scrimAlpha)))
}
```

The `UiEntryPoints` seam (`di/UiEntryPoints.kt:11`):

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface UiEntryPoints {
    fun userRepository(): IUserRepository
    fun themePreferenceStore(): ThemePreferenceStore
}
```

## 6. Reusable motion (`MotionModifiers.kt:16`)

Shared spring specs and two `composed` modifiers so every widget gets its own animation state:

```kotlin
object MotionSprings {
    val press:  SpringSpec<Float> = spring(dampingRatio = 0.6f,  stiffness = 900f)
    val settle: SpringSpec<Float> = spring(dampingRatio = 0.8f,  stiffness = 420f)
    val pop:    SpringSpec<Float> = spring(dampingRatio = 0.55f, stiffness = 520f)
}

fun Modifier.pressScale(interactionSource: InteractionSource, pressedScale: Float = 0.95f): Modifier =
    composed {
        val pressed by interactionSource.collectIsPressedAsState()
        val scale by animateFloatAsState(if (pressed) pressedScale else 1f, MotionSprings.press, label = "pressScale")
        graphicsLayer { scaleX = scale; scaleY = scale }
    }
```

## Related

[[Appearance and Theming]] - [[Appearance and Theming - Explained Simply]] -
[[Appearance and Theming - Basic Example]] - [[Settings - Code]] -
[[Dependency Injection (Hilt)]] - [[Async and Reactivity]] - [[Home]]
