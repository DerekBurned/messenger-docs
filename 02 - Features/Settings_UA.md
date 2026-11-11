---
tags:
  - _UA
---
# Налаштування

Вкладка налаштувань і її підекрани, збережена тема оформлення та видалення акаунта.

## Складові

- `SettingsScreen` (`presentation/screens/settings/SettingsScreen.kt:10`, з `SettingsScreenContent`,
  `SettingsSubScaffold`, `SettingsScreenPreview`) - хаб налаштувань зі спільними рядками в
  `presentation/components/common/SettingsComponents.kt`.
- Підекрани в `presentation/screens/settings/`: `NotificationsScreen`, `PrivacyScreen`,
  `SecurityScreen` і теки `appearance/` (`AppearanceScreen` та його частини шпалер/інтенсивності),
  `datastorage/`, `language/` і `deleteaccount/`.
- Профіль: `ProfileScreen`, `EditProfileScreen`, `ChangeAccountScreen`, `ChangePhoneScreen` (теки
  в `presentation/screens/`) з `ProfileViewModel`, `EditProfileViewModel`, `ChangeAccountViewModel`,
  `ChangePhoneViewModel`.
- Тема: `AppearanceViewModel` (`presentation/screens/settings/appearance/AppearanceViewModel.kt:28`),
  `ThemePreferenceStore` (`data/local/prefs/ThemePreferenceStore.kt:27`) і система теми в
  `presentation/theme/`.
- Видалення акаунта: `DeleteAccountViewModel`, `DeleteAccountUseCase`
  (`domain/usecase/auth/DeleteAccountUseCase.kt`), `AccountDeletionService`
  (`data/remote/auth/AccountDeletionService.kt`) і Cloud Function `deleteAccount`
  (`functions/src/index.ts:305`).

## Навігація до налаштувань

`SettingsScreen` надає колбеки (`onProfileClick`, `onPrivacyClick`, `onAppearanceClick`, ...), що
пушать відповідні маршрути; кожен маршрут підекрана зареєстрований у нав-графі
(`presentation/navigation/graph/MainDisplay.kt:338`-`:379`) і виштовхується через `onBack`. Див.
[[Navigation_UA|Навігація]].

## Тема оформлення (DataStore)

Режим теми (SYSTEM / LIGHT / DARK) зберігається через Jetpack DataStore Preferences
(`datastore` `1.1.7`, `gradle/libs.versions.toml:47`). `ThemePreferenceStore`
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

`AppearanceViewModel` (`presentation/screens/settings/appearance/AppearanceViewModel.kt:28`) перетворює той холодний
потік у стан UI через `stateIn`:

```kotlin
val themeMode: StateFlow<ThemeMode> = themePreferenceStore.themeMode
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), ThemeMode.SYSTEM)
```

Чому `stateIn(WhileSubscribed(5_000))`: екрану потрібне поточне значення на першому кадрі (звідси
`StateFlow` із початковим `SYSTEM`), а `WhileSubscribed(5_000)` тримає upstream-збирання DataStore
живим 5 с після виходу останнього підписника, тож швидка рекомпозиція чи зміна конфігурації не
перевідкриває читач DataStore. `setDarkTheme` пише назад на `viewModelScope`. Застосована тема
споживається у верхівці дерева Compose `MessengerTheme` (`presentation/theme/Theme.kt`),
що відображає `ThemeMode` на світлу/темну колірну схему.

## Система теми

`presentation/theme/` містить `Theme.kt` (composable `MessengerTheme` і вибір колірної
схеми), `Color.kt`, `Type.kt`, `Shape.kt`, `Dimens.kt`, `Motion.kt`, `Tokens.kt` (дизайн-токени) та
`ThemeMode.kt` (enum). Це стандартна тематизація Material 3 із проєктним шаром токенів згори.

## Інші області налаштувань

Більшість цих екранів - це каркас інтерфейсу: їхні перемикачі - локальний стан `rememberSaveable`, не
зберігаються й не підключені до поведінки, а більшість прихована в release-збірках feature flag-ами (див.
"Feature flag" нижче). Що екрани роблять насправді:

- `PrivacyScreen` (`presentation/screens/settings/PrivacyScreen.kt:17`) - локальні перемикачі "Last seen" і
  "Read receipts" та список рядків "Who can..." з порожніми обробниками кліку (усе за
  `SETTINGS_PRIVACY_CONTROLS`), плюс запис "Delete account" (завжди показаний), що відкриває
  `DeleteAccountRoute`. Налаштування видимості номера телефону він не зберігає.
- `SecurityScreen` (`presentation/screens/settings/SecurityScreen.kt:13`) - рядок "Recovery code", що показує,
  чи код збережено, і відкриває `RecoveryCodeRoute` (`MainDisplay.kt:370`-`:376`; див.
  [[End-to-End Encryption_UA|Наскрізне шифрування]]), плюс рядки-заглушки (двоетапна верифікація,
  блокування кодом, активні сеанси) за `SETTINGS_SECURITY_PLACEHOLDERS`. E2EE safety number тут немає:
  `SafetyNumberSection` показується на профілі співрозмовника в чаті
  (`ChatUserProfileScreenContent.kt:175`, див. [[Chat Profile and Shared Media_UA|Профіль чату та спільні медіа]]).
- `NotificationsScreen` - чотири локальні перемикачі (показ сповіщень, звук, вібрація, прев'ю
  повідомлення), що не змінюють спосіб побудови сповіщень; справжня поведінка сповіщень описана в
  [[Notifications_UA|Сповіщення]].
- `DataStorageScreen` - локальні перемикачі "over Wi-Fi" / "over mobile" і віджети легенди сховища; він не
  викликає `MediaCache` (див. [[Data Layer_UA|Шар даних]]).
- `LanguageScreen` - локально запам'ятований індекс вибору; він не змінює мову застосунку.

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

Callable (`functions/src/index.ts:305`) працює з admin SDK, бо клієнтські правила не дозволяють
пристрою видалити власний документ користувача, повідомлення чи опубліковані ключі. Він:

- ставить надгробок на `users/{uid}`: `deleted = true`, `deletedAt`, очищені `username`,
  `usernameLower`, `avatarUrl`, `phoneNumber`, `isOnline = false`, прибраний `fcmToken`;
- за можливості, паралельно: видаляє підколекції `keys`, `recovery`, `devices` і `contactAliases`
  (опубліковані E2EE-ключі, сховище відновлення й рядки FCM на пристрій), префікс Storage
  `avatars/{uid}/` та `presence`, `calls` і `callAcks` для uid у RTDB;
- видаляє користувача Firebase Auth останнім (уже відсутній користувач - не помилка).

`firestore.rules` відхиляють будь-яке оновлення профілю з надгробком і забороняють видалення
документа користувача, тож надгробок не можна скасувати з пристрою. Вхід відхиляє профіль із
надгробком ("This account has been deleted."), а шлюз сесії перетворює його на
`SignedOut(ACCOUNT_DELETED)`, що виводить акаунт на всіх інших пристроях. Співрозмовники бачать
акаунт як "Unknown user" (див. [[Chat and Messaging_UA|Чат і обмін повідомленнями]] та
[[Conversations List_UA|Список розмов]]).

## Feature flag

Налаштування - найбільш прапорцева область; кожен перемикач задокументовано в `remoteconfig.template.json` і
описано в [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]]. Застосування - у UI-composable, бо екрани-заглушки не мають
ViewModel; екрани з ViewModel перевіряють і там:

- `SETTINGS_SECURITY`, `SETTINGS_NOTIFICATION_CONTROLS`, `SETTINGS_DATA_STORAGE`, `SETTINGS_LANGUAGE` -
  `SettingsScreenContent` додає кожен рядок лише коли його прапорець увімкнено
  (`presentation/screens/settings/SettingsScreenContent.kt:178`-`:181`).
- `SETTINGS_PRIVACY_CONTROLS` - `PrivacyScreen.kt:22`; `SETTINGS_SECURITY_PLACEHOLDERS` -
  `SecurityScreen.kt:28`.
- `ACCOUNTS_SWITCH` - рядок "Switch account" (`SettingsScreenContent.kt:143`); див. також
  [[Authentication_UA|Автентифікація]].
- Вигляд: див. [[Appearance and Theming_UA|Вигляд і теми]] (шлюзи в UI і ViewModel).
- Профіль: `PROFILE_EDIT` ховає кнопку редагування (`ProfileScreenContent.kt:79`) і змушує
  `EditProfileViewModel` відхиляти `Save` (`EditProfileViewModel.kt:58`); `PROFILE_AVATAR_UPLOAD` і
  `PROFILE_AVATAR_CAMERA` обмежують дії з аватаром у `ProfileScreen.kt:34`-`:35` і `ProfileViewModel.kt:76`,
  `:81`; `PROFILE_EDIT_CHANGE_PHOTO` і `PROFILE_CHANGE_PHONE` обмежують елементи в
  `EditProfileScreenContent.kt:51`, `:75`.
- У release-типі збірки застосунок стартує з цими вимкненими за замовчуванням (`FEATURES_OFF_BY_DEFAULT` із
  `featuresOffInRelease` в `app/build.gradle.kts:29`-`:39`): `profile_edit_change_photo`, `accounts_switch`,
  `partner_profile_mute`, `partner_profile_block`, `settings_privacy_controls`,
  `settings_security_placeholders`, `settings_notification_controls`, `settings_data_storage` і
  `settings_language`. Remote Config усе одно може їх увімкнути.

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
[[Authentication_UA|Автентифікація]] - [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]]
