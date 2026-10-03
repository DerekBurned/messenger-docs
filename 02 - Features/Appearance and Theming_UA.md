---
tags:
  - _UA
---
# Вигляд і теми

Зовнішній вигляд застосунку: режим теми (світла/темна/системна), градієнтні фони плюс бібліотека
імпортованих користувачем шпалер, повзунок інтенсивності фону, позиція таб-бару, розмір тексту й
шрифт, шар токенів дизайну Material 3, багаторазові модифікатори анімації та набір тестів контрасту
WCAG, що стереже читабельність. Усі налаштування зберігаються через DataStore і застосовуються на
вершині дерева Compose, тож зміна наживо перетемовує весь застосунок. Основи налаштування теми є
також у [[Settings_UA|Налаштування]].

## Складові

- `AppearanceViewModel` (`presentation/viewmodel/AppearanceViewModel.kt:25`) - `@HiltViewModel`,
  віддає `themeMode`, `background`, `intensity`, `barPosition`, `textSize`, `appFont` і `wallpapers`
  як `StateFlow` та дії запису (`setDarkTheme`, `setBackground`, `previewIntensity`,
  `commitIntensity`, `setBarPosition`, `setTextSize`, `setAppFont`, `importWallpaper`,
  `renameWallpaper`, `deleteWallpaper`).
- `ThemePreferenceStore` (`data/local/prefs/ThemePreferenceStore.kt:27`) - `@Singleton` над
  `preferencesDataStore("theme_prefs")`, ключі `theme_mode`, `app_background`, `theme_intensity`,
  `bar_position`, `text_size`, `app_font` і `wallpapers_json` (список збережених шпалер).
- `MessengerTheme` (`presentation/screens/ui/theme/Theme.kt:45`) - відображає `ThemeMode` на
  світлу/темну `ColorScheme` і набір токенів, масштабує типографіку за розміром тексту й шрифтом і
  надає `LocalMessengerTokens` та `LocalContentColor`.
- `AppBackgrounds` (`presentation/screens/ui/theme/AppBackgrounds.kt:8`) - шість градієнтних
  пресетів, `appBackgroundFor`, `CUSTOM_BACKGROUND_PREFIX`, `backgroundKeyFor`, `wallpaperIdOf`,
  `wallpaperFileById` і `customWallpaperFile`.
- `WallpaperBackdrop` / `WallpaperBackground`
  (`presentation/components/common/WallpaperBackground.kt:35`, `:89`) - малюють градієнт,
  необов'язкове імпортоване зображення й затемнення інтенсивності за вмістом.
- `MotionModifiers` (`presentation/components/common/MotionModifiers.kt`) - `Modifier.pressScale`,
  `Modifier.springEnter`; пружини живуть у `MotionSprings`
  (`presentation/components/common/model/MotionSprings.kt:6`).
- UI: `presentation/screens/settings/appearance/` (`AppearanceScreen`, `WallpaperGrid`,
  `BackgroundTile`, `SavedWallpaperTile`, `RenameWallpaperDialog`, `IntensityCard`,
  `LabeledSegmented`). Ручний DI-шов: `UiEntryPoints` (`di/UiEntryPoints.kt:12`).

## Режим теми (системна / світла / темна)

`MessengerTheme` (`Theme.kt:45`) перетворює режим на boolean і обирає схему та токени:

```kotlin
val darkTheme = when (themeMode) {
    ThemeMode.SYSTEM -> isSystemInDarkTheme()
    ThemeMode.LIGHT -> false
    ThemeMode.DARK -> true
}
```

Корінь композиції збирає збережені налаштування й передає їх усередину, тож запис налаштування
перетемовує все дерево без ручної інвалідації (`MainActivity.kt:83` читає
`themePreferenceStore.themeMode`, `:91` викликає
`MessengerTheme(themeMode = themeMode, textSize = textSize, appFont = appFont)`).
`AppearanceViewModel` пише його через `setDarkTheme` (`AppearanceViewModel.kt:51`). Механіка
stateIn/DataStore та сама, що в [[Settings_UA|Налаштування]], і стисло показана в
[[Appearance and Theming - Basic Example]].

## Фони й імпортовані шпалери

`AppBackgrounds` (`AppBackgrounds.kt:8`) - шість пресетів (`neutral`, `aurora`, `plum`, `teal`,
`midnight`, `mono`), кожен із темним і світлим списком кольорів. `WallpaperBackdrop`
(`WallpaperBackground.kt:35`) малює `Brush.verticalGradient(colors)`, обираючи темний чи світлий
список за `tokens.isDark`, потім накладає Coil `AsyncImage`, коли ключ фону вказує на імпортовані
шпалери.

Імпортовані шпалери утворюють бібліотеку. `importWallpaper` (`AppearanceViewModel.kt:81`) копіює
обраний `Uri` у `filesDir/wallpapers/wallpaper_<id>.jpg` на `Dispatchers.IO` (id - поточний час),
додає `SavedWallpaper(id, "Wallpaper N")` у збережений список і обирає його ключем `"custom:<id>"`.
`renameWallpaper` і `deleteWallpaper` редагують список; видалення обраних шпалер повертає на
`neutral`. `customWallpaperFile` (`AppBackgrounds.kt:64`) перетворює ключ `custom:` назад на файл;
будь-який інший ключ дає null, і показується градієнт пресету.

## Інтенсивність фону

Інтенсивність затемнює фон, щоб вміст переднього плану лишався читабельним на строкатих шпалерах.
`WallpaperBackdrop` обчислює затемнення (`WallpaperBackground.kt:76`):

```kotlin
val scrimAlpha = ((1f - intensity) * 0.85f).coerceIn(0f, 1f)
```

1.0 - без затемнення, 0.0 - затемнення на 85%; у темному режимі воно чорне, у світлому - біле.
Повзунок використовує двоетапний запис, тож перетягування плавне, а зберігається лише відпущене
значення: `previewIntensity` (`AppearanceViewModel.kt:61`) встановлює `MutableStateFlow`
`liveIntensity`, що `combine`-иться поверх збереженого значення (`ThemePreferenceStore.kt:74`), а
`commitIntensity` (`:65`) пише його в DataStore.

## Розкладка, розмір тексту й шрифт

`barPosition` (угорі чи внизу) переміщує плаваючий таб-бар; екрани читають його через
`LocalTabBarPosition` і відповідно дають відступи вмісту (наприклад нижній відступ списку чатів).
`textSize` масштабує всю типографіку, а `appFont` міняє сімейство шрифтів - обидва через
`messengerTypography(scale, family)` у `MessengerTheme`.

## Токени дизайну й анімація

`MessengerTheme` надає `LocalMessengerTokens` (читається будь-де як `messengerTokens`), тож
компоненти читають кольори застосунку (`trackFill`, `pillFill`, `accent`, `textPrimary`,
`textMuted`, `isDark`, ...) поряд зі схемою Material. `MotionModifiers.kt` постачає багаторазову
анімацію віджетів: `Modifier.pressScale` зменшує віджет під час натискання й пружно повертає
(`MotionSprings.press`); `Modifier.springEnter` проявляє й масштабує віджет при першій композиції
(`MotionSprings.settle`). Обидва - `Modifier.composed { }`, тож кожне місце виклику має власний
стан анімації.

## Читабельність: набір тестів контрасту WCAG

0.1.13.1 додав інструментовану обв'язку в `app/src/androidTest/.../presentation/contrast/`
(`ContrastHarness`, `BitmapTextProbe`, `ContrastMath`, `ContrastFixtures`) і тести контрасту для
компонентів списку, загальних, чату (включно з голосовою бульбашкою, смугою запису, смугою чернетки
й міні-плеєром), автентифікації та дзвінків. Кожен тест рендерить компонент в обох темах на його
справжньому фоні, знімає bitmap і перевіряє контраст кожного текстового вузла з тим, що реально під
ним: фон - найчастіший колір в області вузла, передній план - колір, чия яскравість найдальша від
нього й займає щонайменше 0.5% області, що відкидає згладжування.

Знайдені й виправлені вади:

- `MessengerTheme` ніколи не надавав `LocalContentColor`, а жоден екран не обгортає вміст у
  `Surface`, тож кожен `Text` без явного кольору був чорним - 1.15:1 на темному фоні. Тепер він
  надає `tokens.textPrimary`.
- `PillButton` робив свій напівпрозорий контейнер непрозорим, перетворюючи його на суцільний білий
  під білим підписом (1.00:1).
- `AccountRow` використовував колір чорнила поля на темному фоні (1.00:1).
- `AuthModeToggle` мав жорстко білі підписи на біло-тонованій доріжці (1.39:1 у світлій темі).
- `CallControlButton` використовував колір тексту теми на завжди чорній поверхні дзвінка (1.11:1 у
  світлій темі).
- `ContactRow`, `SearchUserItem`, `ForwardConversationRow` і `EditChatRow` малювали власний сірий
  аватар і жорстко чорний/сірий текст; тепер використовують `MessengerAvatar` і токени.
- Сірі плейсхолдери на білому (3.54:1) тепер використовують `OnSurfaceMuted` (4.83:1), а
  `TextLightMuted` змінив альфу з `0x99` на `0xA6`, щоб перевищити 4.5:1 на світлому фоні.

Решта збоїв - палітра аватарів, акцентні токени й кольори надісланої бульбашки, тобто дизайнерські
рішення, відкладені на потім; голосові бульбашки падають на тих самих спільних токенах, що й
текстова. Набір запускається через `adb shell am instrument`, ніколи не через
`connectedDebugAndroidTest` на особистому пристрої (той видаляє застосунок разом із даними й
ключами).

## Чому такий дизайн

- DataStore дає кожне налаштування як холодний `Flow`, тож зміни вигляду поширюються через Compose
  без ручної інвалідації - та сама причина реактивних налаштувань, що й у
  [[Settings_UA|Налаштування]].
- Читання сховища шпалер із composable через Hilt `@EntryPoint` (`UiEntryPoints`) лишає
  `WallpaperBackdrop` придатним поза будь-якою ViewModel, проходячи все одно через граф DI - див.
  [[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]].
- Накладка попереднього перегляду інтенсивності відокремлює "що я тягну" від "що збережено", тож
  повзунок плавний без запису DataStore на кожен кадр.
- Централізація анімації в `MotionModifiers` тримає кожну натискну поверхню узгодженою й дешевою у
  застосуванні.
- Вимірювання контрасту з відрендерених пікселів на справжньому фоні ловить те, що пропускає огляд
  токенів (успадкований чорний текст, випадково непрозорі заливки), без списку винятків.

## Пов'язане

[[Settings_UA|Налаштування]] - [[Appearance and Theming - Code]] -
[[Appearance and Theming - Explained Simply]] - [[Appearance and Theming - Basic Example]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]] -
[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] - [[Voice Messages_UA|Голосові повідомлення]]
