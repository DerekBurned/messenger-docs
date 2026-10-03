---
tags:
  - _UA
---
# Налаштування

Вкладка налаштувань і її підекрани, збережена тема оформлення та видалення акаунта.

## Складові

- `SettingsScreen` (`presentation/screens/settings/SettingsScreen.kt`, з `SettingsScreenContent`,
  `SettingsScaffold`, `SettingsScreenPreview`) - хаб налаштувань зі спільними рядками в
  `presentation/components/common/SettingsComponents.kt`.
- Підекрани в `presentation/screens/settings/`: `NotificationsScreen`, `PrivacyScreen`,
  `SecurityScreen` і теки `appearance/` (`AppearanceScreen` та його частини шпалер/інтенсивності),
  `datastorage/`, `language/` і `deleteaccount/`.
- Профіль: `ProfileScreen`, `EditProfileScreen`, `ChangeAccountScreen`, `ChangePhoneScreen` (теки
  в `presentation/screens/`) з `ProfileViewModel`, `EditProfileViewModel`, `ChangeAccountViewModel`,
  `ChangePhoneViewModel`.
- Тема: `AppearanceViewModel` (`presentation/viewmodel/AppearanceViewModel.kt:25`),
  `ThemePreferenceStore` (`data/local/prefs/ThemePreferenceStore.kt:27`) і система теми в
  `presentation/screens/ui/theme/`.
- Видалення акаунта: `DeleteAccountViewModel`, `DeleteAccountUseCase`
  (`domain/usecase/auth/DeleteAccountUseCase.kt`), `AccountDeletionService`
  (`data/remote/auth/AccountDeletionService.kt`) і Cloud Function `deleteAccount`
  (`functions/src/index.ts:251`).

## Навігація до налаштувань

`SettingsScreen` надає колбеки (`onProfileClick`, `onPrivacyClick`, `onAppearanceClick`, ...), що
пушать відповідні маршрути; кожен маршрут підекрана зареєстрований у нав-графі
(`presentation/navigation/graph/MainDisplay.kt:324`-`:356`) і виштовхується через `onBack`. Див.
[[Navigation_UA|Навігація]].

## Тема оформлення (DataStore)

Режим теми (SYSTEM / LIGHT / DARK) зберігається через Jetpack DataStore Preferences
(`datastore` `1.1.7`, `gradle/libs.versions.toml:46`). `ThemePreferenceStore`
(`data/local/prefs/ThemePreferenceStore.kt:27`) - це `@Singleton` над
`preferencesDataStore(name = "theme_prefs")`, що також тримає шпалери, інтенсивність, позицію
бару, розмір тексту й шрифт (див. [[Appearance and Theming_UA|Вигляд і теми]]):

```kotlin
val themeMode: Flow<ThemeMode> = context.themeDataStore.data.map { prefs ->
    prefs[themeModeKey]
        ?.let { runCatching { ThemeMode.valueOf(it) }.getOrNull() }
        ?: ThemeMode.SYSTEM
}
suspend fun setThemeMode(mode: ThemeMode) {
    context.themeDataStore.edit { prefs -> prefs[themeModeKey] = mode.name }
}
```

Мета: тривке, реактивне налаштування теми. Читання - це холодний `Flow<ThemeMode>` (DataStore
випромінює на кожну зміну), тож UI перетемовується в мить запису налаштування;
невідомі/застарілі значення відкочуються до `SYSTEM` через `runCatching`.

`AppearanceViewModel` (`presentation/viewmodel/AppearanceViewModel.kt:30`) перетворює той холодний
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

- `PrivacyScreen` - видимість номера телефону (`PhoneVisibility`, зберігається в профілі),
  пов'язані перемикачі приватності та вхід до видалення акаунта.
- `SecurityScreen` - виводить E2EE safety number для контакту через `SafetyNumberViewModel` /
  `SafetyNumberSection`; див. [[End-to-End Encryption_UA|Наскрізне шифрування]].
- `NotificationsScreen` - налаштування сповіщень (канали створюються під час запуску, див.
  [[Notifications_UA|Сповіщення]]).
- `DataStorageScreen` - керування медіа-кешем / сховищем (`MediaCache`, див. [[Data Layer_UA|Шар даних]]).
- `LanguageScreen` - вибір мови застосунку.

## Видалення акаунта

Налаштування > Приватність > Delete account відкриває `DeleteAccountRoute`. Екран має два кроки
(`DeleteAccountStep.CONFIRM`, `VERIFY`):

1. Користувач вводить `DELETE` для підтвердження; інакше `DeleteAccountViewModel` не продовжує
   (`delete_account_error_confirm`), як і тоді, коли в акаунта немає номера телефону.
2. Він надсилає SMS-код на власний номер акаунта, тримає `verificationId` у своєму стані, потім
   повторно автентифікується кодом (`firebaseAuthService.reauthenticateWithPhone`). Firebase вимагає
   нещодавнього входу перед видаленням користувача, і цей крок також доводить, що видаляє саме
   власник телефону.
3. Він від'єднує присутність і запускає `DeleteAccountUseCase`, що викликає
   `authRepository.deleteAccount()` -> `AccountDeletionService` -> callable `deleteAccount`. Лише
   коли сервер підтвердить, пристрій очищується: `sessionCleaner.clearOnDelete(uid)` (усе, що
   прибирає `clearOnLogout`, плюс файл ключа ідентичності E2EE), вихід із Firebase і `resetDB()`.

Callable (`functions/src/index.ts:251`) працює з admin SDK, бо клієнтські правила не дозволяють
пристрою видалити власний документ користувача, повідомлення чи опубліковані ключі. Він:

- ставить надгробок на `users/{uid}`: `deleted = true`, `deletedAt`, очищені `username`,
  `usernameLower`, `avatarUrl`, `phoneNumber`, `isOnline = false`, прибраний `fcmToken`;
- за можливості, паралельно: видаляє підколекції `keys` і `contactAliases`, префікс Storage
  `avatars/{uid}/` та `presence`, `calls` і `callAcks` для uid у RTDB;
- видаляє користувача Firebase Auth останнім (уже відсутній користувач - не помилка).

`firestore.rules` відхиляють будь-яке оновлення профілю з надгробком і забороняють видалення
документа користувача, тож надгробок не можна скасувати з пристрою. Вхід відхиляє профіль із
надгробком ("This account has been deleted."), а шлюз сесії перетворює його на
`SignedOut(ACCOUNT_DELETED)`, що виводить акаунт на всіх інших пристроях. Співрозмовники бачать
акаунт як "Unknown user" (див. [[Chat and Messaging_UA|Чат і обмін повідомленнями]] та
[[Conversations List_UA|Список розмов]]).

## Чому такий дизайн

- DataStore (а не SharedPreferences) дає реактивний `Flow` для теми, тож зміни теми пропагуються
  через Compose без ручної інвалідації, і це поточний рекомендований API налаштувань.
- Відкриття налаштування як `StateFlow` через `stateIn` дає UI синхронне поточне значення, тримаючи
  єдине реактивне джерело істини.
- Видалення акаунта серверне, бо лише admin SDK може прибрати дані, які правила захищають від
  клієнтів; надгробок на профілі (замість видалення) дозволяє іншим пристроям і співрозмовникам
  побачити, що акаунта більше немає.

## Пов'язане

[[End-to-End Encryption_UA|Наскрізне шифрування]] - [[Notifications_UA|Сповіщення]] -
[[Data Layer_UA|Шар даних]] - [[Navigation_UA|Навігація]] -
[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]] -
[[Authentication_UA|Автентифікація]]
