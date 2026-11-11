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

- `AppearanceViewModel` (`presentation/screens/settings/appearance/AppearanceViewModel.kt:28`) - `@HiltViewModel`,
  віддає `themeMode`, `background`, `intensity`, `barPosition`, `textSize`, `appFont` і `wallpapers`
  як `StateFlow` та дії запису (`setDarkTheme`, `setBackground`, `previewIntensity`,
  `commitIntensity`, `setBarPosition`, `setTextSize`, `setAppFont`, `importWallpaper`,
  `renameWallpaper`, `deleteWallpaper`).
- `ThemePreferenceStore` (`data/local/prefs/ThemePreferenceStore.kt:27`) - `@Singleton` над
  `preferencesDataStore("theme_prefs")`, ключі `theme_mode`, `app_background`, `theme_intensity`,
  `bar_position`, `text_size`, `app_font` і `wallpapers_json` (список збережених шпалер).
- `MessengerTheme` (`presentation/theme/Theme.kt:44`) - відображає `ThemeMode` на
  світлу/темну `ColorScheme` і набір токенів, масштабує типографіку за розміром тексту й шрифтом і
  надає `LocalMessengerTokens` та `LocalContentColor`.
- `AppBackgrounds` (`presentation/theme/AppBackgrounds.kt:8`) - шість градієнтних
  пресетів, `appBackgroundFor`, `CUSTOM_BACKGROUND_PREFIX`, `backgroundKeyFor`, `wallpaperIdOf`,
  `wallpaperFileById` і `customWallpaperFile`.
- `WallpaperBackdrop` / `WallpaperBackground`
  (`presentation/components/common/WallpaperBackground.kt:34`, `:88`) - малюють градієнт,
  необов'язкове імпортоване зображення й затемнення інтенсивності за вмістом.
- `MotionModifiers` (`presentation/components/common/MotionModifiers.kt`) - `Modifier.pressScale`,
  `Modifier.springEnter`; пружини живуть у `MotionSprings`
  (`presentation/components/common/model/MotionSprings.kt:6`).
- UI: `presentation/screens/settings/appearance/` (`AppearanceScreen`, `WallpaperGrid`,
  `BackgroundTile`, `SavedWallpaperTile`, `RenameWallpaperDialog`, `IntensityCard`,
  `LabeledSegmented`). Ручний DI-шов: `UiEntryPoints` (`di/UiEntryPoints.kt:13`).

## Режим теми (системна / світла / темна)

`MessengerTheme` (`Theme.kt:44`) перетворює режим на boolean і обирає схему та токени:

```kotlin
val darkTheme = when (themeMode) {
    ThemeMode.SYSTEM -> isSystemInDarkTheme()
    ThemeMode.LIGHT -> false
    ThemeMode.DARK -> true
}
```

Корінь композиції збирає збережені налаштування й передає їх усередину, тож запис налаштування
перетемовує все дерево без ручної інвалідації (`MainActivity.kt:91` читає
`themePreferenceStore.themeMode`, `:100` викликає
`MessengerTheme(themeMode = themeMode, textSize = textSize, appFont = appFont)`).
`AppearanceViewModel` пише його через `setDarkTheme` (`AppearanceViewModel.kt:55`). Механіка
stateIn/DataStore та сама, що в [[Settings_UA|Налаштування]], і стисло показана в
[[Appearance and Theming - Basic Example]].

## Фони й імпортовані шпалери

`AppBackgrounds` (`AppBackgrounds.kt:8`) - шість пресетів (`neutral`, `aurora`, `plum`, `teal`,
`midnight`, `mono`), кожен із темним і світлим списком кольорів. `WallpaperBackdrop`
(`WallpaperBackground.kt:65`) малює `Brush.verticalGradient(colors)`, обираючи темний чи світлий
список за `tokens.isDark`, потім накладає Coil `AsyncImage`, коли ключ фону вказує на імпортовані
шпалери.

Імпортовані шпалери утворюють бібліотеку. `importWallpaper` (`AppearanceViewModel.kt:92`) копіює
обраний `Uri` у `filesDir/wallpapers/wallpaper_<id>.jpg` на `Dispatchers.IO` (id - поточний час),
додає `SavedWallpaper(id, "Wallpaper N")` у збережений список і обирає його ключем `"custom:<id>"`.
`renameWallpaper` і `deleteWallpaper` редагують список; видалення обраних шпалер повертає на
`neutral`. `customWallpaperFile` (`AppBackgrounds.kt:64`) перетворює ключ `custom:` назад на файл;
будь-який інший ключ дає null, і показується градієнт пресету.

## Інтенсивність фону

Інтенсивність затемнює фон, щоб вміст переднього плану лишався читабельним на строкатих шпалерах.
`WallpaperBackdrop` обчислює затемнення (`WallpaperBackground.kt:75`):

```kotlin
val scrimAlpha = ((1f - intensity) * 0.85f).coerceIn(0f, 1f)
```

1.0 - без затемнення, 0.0 - затемнення на 85%; у темному режимі воно чорне, у світлому - біле.
Повзунок використовує двоетапний запис, тож перетягування плавне, а зберігається лише відпущене
значення: `previewIntensity` (`AppearanceViewModel.kt:67`) встановлює `MutableStateFlow`
`liveIntensity`, що `combine`-иться поверх збереженого значення (`ThemePreferenceStore.kt:74`), а
`commitIntensity` (`AppearanceViewModel.kt:72`, через
`ThemePreferenceStore.setIntensity`, `:91`) пише його в DataStore.

### Збережені шпалери лишаються придатними, коли власні шпалери вимкнено

`appearance_custom_wallpaper` раніше ховав плитки збережених шпалер разом із плиткою Add, тож
користувач не міг обрати шпалери, які вже зберіг, зокрема й ті, що використовувались. Від PR #65 (злито
2026-11-07, виправляє #10) перемикач ховає лише плитку Add і меню Rename/Delete за довгим натисканням.
`WallpaperGrid` завжди показує збережені плитки (`customTiles = saved + Add, якщо customEnabled`,
`presentation/screens/settings/appearance/WallpaperGrid.kt:38`) і передає `editable = customEnabled` у
кожну `SavedWallpaperTile` (`:68`), де `onLongClick` дорівнює null, а випадне меню не компонується, коли
воно false (`presentation/screens/settings/appearance/SavedWallpaperTile.kt:50`, `:70`-`:74`, `:101`).
Вибір збережених шпалер і далі йде через `setBackground`, що обмежується лише батьківським прапорцем
`appearance_wallpapers`, а `importWallpaper`, `renameWallpaper` і `deleteWallpaper` у ViewModel
відмовляють, коли власні шпалери вимкнено (`AppearanceViewModel.kt:93`, `:109`, `:115`).

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

### Прохід по контрасту після виправлення поля входу (PR #39 і PR #44)

Два пізніші проходи розширили набір і виправили знайдене (злито 2026-11-03):

- Новий токен `textOnFieldMuted` (`TextOnFieldMuted`, `presentation/theme/Color.kt:61`, оголошений у
  `presentation/theme/Tokens.kt:24`) - приглушений колір для вмісту на завжди майже білій заливці поля.
  Плейсхолдери входу, доріжка кільця OTP і стрілка вибору країни використовували `textMuted`, що в темній
  темі білий, тож було біле на білому; плейсхолдер композера повідомлень, повідомлення про видаленого
  співрозмовника та іконка скасування відповіді мали ту саму проблему, а неактивна фішка фільтра вибору
  медіа використовувала `Color.Gray` (2.69:1).
- Вторинний текст, що притемнював `textPrimary` чи `textMuted` до 60% або використовував `Color.Gray`,
  тепер використовує `textMuted` (4.6:1 і краще в обох темах): плейсхолдер і результати пошуку, рядок
  "код надіслано", `@username` контакту, рядок "був у мережі" профілю, рядок версії налаштувань і
  порожні стани чатів, контактів та пересилання. Див. `MessengerTextFields.kt`,
  `SearchResultsOverlay.kt`, `ChatsEmpty.kt` і `MessageComposer.kt` у diff PR #44.
- Нові тести покривають порожні `MessengerInputField` і `OtpCodeField`, композер, профіль чату та екран
  налаштувань (`AuthComponentContrastTest`, `ChatComponentContrastTest`, `ProfileScreenContrastTest`,
  `SettingsScreenContrastTest`, `ListComponentContrastTest` і `CommonComponentContrastTest` в
  `app/src/androidTest/.../presentation/contrast/`).

Перелік решти збоїв нижче походить із 0.1.13.1 і не перезапускався для 0.2.2.4.

Решта збоїв - палітра аватарів, акцентні токени й кольори надісланої бульбашки, тобто дизайнерські
рішення, відкладені на потім; голосові бульбашки падають на тих самих спільних токенах, що й
текстова. Набір запускається через `adb shell am instrument`, ніколи не через
`connectedDebugAndroidTest` на особистому пристрої (той видаляє застосунок разом із даними й
ключами).

## Feature flag

Кожен елемент керування вигляду має власний прапорець (див. [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]]):
`APPEARANCE_DARK_THEME`, `APPEARANCE_BAR_POSITION`, `APPEARANCE_TEXT_SIZE`, `APPEARANCE_FONT`,
`APPEARANCE_WALLPAPERS` з дочірніми `APPEARANCE_CUSTOM_WALLPAPER` і `APPEARANCE_WALLPAPER_INTENSITY`.
Вони застосовуються у двох шарах:

- UI: `AppearanceScreen` випускає кожну картку, а також плитку Add і меню перейменування/видалення в
  `WallpaperGrid`, лише коли її прапорець увімкнено
  (`presentation/screens/settings/appearance/AppearanceScreen.kt:61`-`:64`, `:74`, `:88`, `:104`, `:132`).
- ViewModel: кожен запис в `AppearanceViewModel` починається з `featureFlags.refuses(...)`:
  `setDarkTheme` (`:56`), `setBackground` (`:63`, на `APPEARANCE_WALLPAPERS`), `previewIntensity` і
  `commitIntensity` (`:68`, `:73`), `setBarPosition` (`:78`), `setTextSize` (`:83`), `setAppFont` (`:88`)
  і три дії з власними шпалерами (`:93`, `:109`, `:115`).

Вимкнення прапорця лише зупиняє зміни; збережене значення й далі застосовується через `MessengerTheme`.

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

[[Settings_UA|Налаштування]] - [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]] - [[Appearance and Theming - Code]] -
[[Appearance and Theming - Explained Simply]] - [[Appearance and Theming - Basic Example]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]] -
[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] - [[Voice Messages_UA|Голосові повідомлення]]
