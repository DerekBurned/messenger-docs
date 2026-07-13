---
tags:
  - _UA
---
# Налаштування

Вкладка налаштувань і її підекрани, а також збережена тема оформлення.

## Складові

- `SettingsScreen` (`presentation/screens/SettingsScreen.kt`) - хаб налаштувань, із
  `SettingsScaffold` і спільними рядками в `presentation/components/common/SettingsComponents.kt`.
- Підекрани в `presentation/screens/settings/`: `AppearanceScreen`, `NotificationsScreen`,
  `PrivacyScreen`, `SecurityScreen`, `DataStorageScreen`, `LanguageScreen`.
- Профіль: `ProfileScreen`, `EditProfileScreen`, `ChangeAccountScreen`, `ChangePhoneScreen` з
  `ProfileViewModel`, `EditProfileViewModel`, `ChangeAccountViewModel`, `ChangePhoneViewModel`.
- Тема: `AppearanceViewModel` (`presentation/viewmodel/AppearanceViewModel.kt:15`),
  `ThemePreferenceStore` (`data/local/prefs/ThemePreferenceStore.kt:17`) і система теми в
  `presentation/screens/ui/theme/`.

## Навігація до налаштувань

`SettingsScreen` надає колбеки (`onProfileClick`, `onPrivacyClick`, `onAppearanceClick`, ...), що
пушать відповідні маршрути; кожен маршрут підекрана зареєстрований у нав-графі
(`presentation/navigation/navigation.kt:241`-`:260`) і виштовхується через `onBack`. Див.
[[Navigation_UA|Навігація]].

## Тема оформлення (DataStore)

Режим теми (SYSTEM / LIGHT / DARK) зберігається через Jetpack DataStore Preferences
(`datastore` `1.1.7`, `gradle/libs.versions.toml:46`). `ThemePreferenceStore`
(`data/local/prefs/ThemePreferenceStore.kt:17`) - це `@Singleton` над
`preferencesDataStore(name = "theme_prefs")`:

```kotlin
val themeMode: Flow<ThemeMode> = context.themeDataStore.data.map { prefs ->
    prefs[themeModeKey]?.let { runCatching { ThemeMode.valueOf(it) }.getOrNull() } ?: ThemeMode.SYSTEM
}
suspend fun setThemeMode(mode: ThemeMode) {
    context.themeDataStore.edit { prefs -> prefs[themeModeKey] = mode.name }
}
```

Мета: тривке, реактивне налаштування теми. Читання - це холодний `Flow<ThemeMode>` (DataStore
випромінює на кожну зміну), тож UI перетемовується в мить запису налаштування;
невідомі/застарілі значення відкочуються до `SYSTEM` через `runCatching`.

`AppearanceViewModel` (`presentation/viewmodel/AppearanceViewModel.kt:19`) перетворює той холодний
потік у стан UI через `stateIn`:

```kotlin
val themeMode: StateFlow<ThemeMode> = themePreferenceStore.themeMode
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), ThemeMode.SYSTEM)
```

Чому `stateIn(WhileSubscribed(5_000))`: екрану потрібне поточне значення на першому кадрі (звідси
`StateFlow` із початковим `SYSTEM`), а `WhileSubscribed(5_000)` тримає upstream-збирання DataStore
живим 5 с після виходу останнього підписника, тож швидка рекомпозиція чи зміна конфігурації не
перевідкриває читач DataStore. `setDarkTheme` пише назад на `viewModelScope`. Застосована тема
споживається у верхівці дерева Compose `MessengerTheme` (`presentation/screens/ui/theme/Theme.kt`),
що відображає `ThemeMode` на світлу/темну колірну схему.

## Система теми

`presentation/screens/ui/theme/` містить `Theme.kt` (composable `MessengerTheme` і вибір колірної
схеми), `Color.kt`, `Type.kt`, `Shape.kt`, `Dimens.kt`, `Motion.kt`, `Tokens.kt` (дизайн-токени) та
`ThemeMode.kt` (enum). Це стандартна тематизація Material 3 із проєктним шаром токенів згори.

## Інші області налаштувань

- `PrivacyScreen` - видимість номера телефону (`PhoneVisibility`, зберігається в профілі) та
  пов'язані перемикачі приватності.
- `SecurityScreen` - виводить E2EE safety number для контакту через `SafetyNumberViewModel` /
  `SafetyNumberSection`; див. [[End-to-End Encryption_UA|Наскрізне шифрування]].
- `NotificationsScreen` - налаштування сповіщень (канали створюються під час запуску, див.
  [[Notifications_UA|Сповіщення]]).
- `DataStorageScreen` - керування медіа-кешем / сховищем (`MediaCache`, див. [[Data Layer_UA|Шар даних]]).
- `LanguageScreen` - вибір мови застосунку.

## Чому такий дизайн

- DataStore (а не SharedPreferences) дає реактивний `Flow` для теми, тож зміни теми пропагуються
  через Compose без ручної інвалідації, і це поточний рекомендований API налаштувань.
- Відкриття налаштування як `StateFlow` через `stateIn` дає UI синхронне поточне значення, тримаючи
  єдине реактивне джерело істини.

## Пов'язане

[[End-to-End Encryption_UA|Наскрізне шифрування]] - [[Notifications_UA|Сповіщення]] -
[[Data Layer_UA|Шар даних]] - [[Navigation_UA|Навігація]] -
[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]]
