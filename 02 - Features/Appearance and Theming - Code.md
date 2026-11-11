# Appearance and Theming - Code

The real source behind [[Appearance and Theming]], verified against the repo and explained block by
block. The plain-language version is [[Appearance and Theming - Explained Simply]]; the minimal
inherited pattern is [[Appearance and Theming - Basic Example]].

(Teaching comments only; the real source has none - that is a project rule.)

---

## 1. The persisted settings (`ThemePreferenceStore.kt:27`)

All appearance settings live in one DataStore. Reads are cold `Flow`s; writes go through
`edit { }`. The theme, background and intensity keys are shown; bar position, text size, font and
the saved-wallpaper list (`wallpapers_json`) follow the same pattern.

```kotlin
private val Context.themeDataStore by preferencesDataStore(name = "theme_prefs")

@Singleton
class ThemePreferenceStore @Inject constructor(
    @ApplicationContext private val context: Context,
) {
    private val themeModeKey = stringPreferencesKey("theme_mode")
    private val backgroundKey = stringPreferencesKey("app_background")
    private val intensityKey = floatPreferencesKey("theme_intensity")

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

## 2. Cold flows -> UI state (`AppearanceViewModel.kt:28`)

`stateIn` gives the screen a value on its first frame; `WhileSubscribed(5_000)` keeps the DataStore
reader alive briefly across recompositions/rotation.

```kotlin
@HiltViewModel
class AppearanceViewModel @Inject constructor(
    @ApplicationContext private val context: Context,
    private val themePreferenceStore: ThemePreferenceStore,
    private val featureFlags: FeatureFlags,
) : ViewModel() {

    val themeMode: StateFlow<ThemeMode> = themePreferenceStore.themeMode
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), ThemeMode.SYSTEM)
    val background: StateFlow<String> = themePreferenceStore.background
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), "neutral")
    val intensity: StateFlow<Float> = themePreferenceStore.intensity
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), 1f)
```

Importing a wallpaper copies the picked file under a new id, adds it to the saved list, then
selects it (`AppearanceViewModel.kt:92`):

```kotlin
    fun importWallpaper(uri: Uri) {
        if (featureFlags.refuses(FeatureFlag.APPEARANCE_CUSTOM_WALLPAPER)) return   // flag gate, as in every writer
        viewModelScope.launch(Dispatchers.IO) {                     // file IO off the main thread
            runCatching {
                val id = System.currentTimeMillis().toString()
                val file = wallpaperFileById(context, id).apply { parentFile?.mkdirs() }
                context.contentResolver.openInputStream(uri)?.use { input ->
                    file.outputStream().use { output -> input.copyTo(output) }
                } ?: return@runCatching
                val name = "Wallpaper ${wallpapers.value.size + 1}"
                themePreferenceStore.addWallpaper(SavedWallpaper(id, name))   // keep a library
                themePreferenceStore.setBackground(backgroundKeyFor(id))       // "custom:<id>"
            }
        }
    }
```

## 3. Applying the mode (`Theme.kt:44`)

`MessengerTheme` maps `ThemeMode` to dark/light, selects a `ColorScheme` and token set, builds the
typography from the text size and font, and provides the tokens and the default content color
through `CompositionLocal`s:

```kotlin
@Composable
fun MessengerTheme(
    themeMode: ThemeMode = ThemeMode.SYSTEM,
    textSize: AppTextSize = AppTextSize.MEDIUM,
    appFont: AppFont = AppFont.DEFAULT,
    content: @Composable () -> Unit,
) {
    val darkTheme = when (themeMode) {
        ThemeMode.SYSTEM -> isSystemInDarkTheme()
        ThemeMode.LIGHT -> false
        ThemeMode.DARK -> true
    }
    val colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme
    val tokens = if (darkTheme) DarkTokens else LightTokens
    val typography = messengerTypography(scale = textSize.scale, family = appFont.toFontFamily())

    CompositionLocalProvider(
        LocalMessengerTokens provides tokens,
        LocalContentColor provides tokens.textPrimary,   // every bare Text reads this
    ) {
        MaterialTheme(
            colorScheme = colorScheme,
            typography = typography,
            shapes = MessengerMaterialShapes,
            content = content,
        )
    }
}
```

`LocalContentColor provides tokens.textPrimary` is the 0.1.13.1 contrast fix: without it every
`Text` with no explicit color fell back to Material's default black, invisible in dark theme.

The root reads the persisted settings and feeds them in, so a write re-themes everything
(`MainActivity.kt:91`):

```kotlin
val themeMode by themePreferenceStore.themeMode
    .collectAsStateWithLifecycle(initialValue = ThemeMode.SYSTEM)
val textSize by themePreferenceStore.textSize
    .collectAsStateWithLifecycle(initialValue = AppTextSize.MEDIUM)
val appFont by themePreferenceStore.appFont
    .collectAsStateWithLifecycle(initialValue = AppFont.DEFAULT)
val barPosition by themePreferenceStore.barPosition
    .collectAsStateWithLifecycle(initialValue = BarPosition.TOP)
MessengerTheme(themeMode = themeMode, textSize = textSize, appFont = appFont) {
  CompositionLocalProvider(LocalTabBarPosition provides barPosition) { /* app */ }
}
```

## 4. The backgrounds (`AppBackgrounds.kt:8`)

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

fun backgroundKeyFor(wallpaperId: String): String = "$CUSTOM_BACKGROUND_PREFIX$wallpaperId"

fun wallpaperIdOf(backgroundKey: String): String? =
    if (backgroundKey.startsWith(CUSTOM_BACKGROUND_PREFIX)) {
        backgroundKey.removePrefix(CUSTOM_BACKGROUND_PREFIX)
    } else {
        null                                                            // not a custom key
    }

fun wallpaperFileById(context: Context, wallpaperId: String): File =
    File(File(context.filesDir, "wallpapers"), "wallpaper_$wallpaperId.jpg")

fun customWallpaperFile(context: Context, backgroundKey: String): File? {
    val id = wallpaperIdOf(backgroundKey) ?: return null
    return wallpaperFileById(context, id)
}
```

## 5. Drawing the backdrop (`WallpaperBackground.kt:34`)

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

(In the source the store and the custom file are skipped in `LocalInspectionMode`, so previews
render the plain gradient.)

The `UiEntryPoints` seam (`di/UiEntryPoints.kt:13`):

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface UiEntryPoints {
    fun userRepository(): IUserRepository
    fun themePreferenceStore(): ThemePreferenceStore
    fun callService(): ICallService
}
```

## 6. Reusable motion (`model/MotionSprings.kt:6`, `MotionModifiers.kt`)

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
