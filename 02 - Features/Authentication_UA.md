---
tags:
  - _UA
---
# Автентифікація

Вхід за номером телефону з одноразовим кодом по SMS, на основі Firebase Auth. Також підтримує
прив'язку номера телефону до наявного акаунта. Чи показувати основний UI, вирішує розв'язана сесія
(ідентичність Firebase плюс підтверджений профіль у Firestore), а не лише прапорець Firebase.

## Складові

- `AuthViewModel` (`presentation/screens/auth/AuthViewModel.kt:37`) - MVI-ViewModel
  (`MviViewModel<AuthUiState, AuthIntent, AuthEffect>`).
- `FirebaseAuthService` (`data/remote/auth/FirebaseAuthService.kt`) - обгортає phone-верифікацію
  `FirebaseAuth`.
- Use case (`domain/usecase/auth/`): `LoginWithPhoneNumberUseCase`, `LinkPhoneUseCase`,
  `LogoutUseCase`, `ObserveAuthStateUseCase`, `ObserveSessionStateUseCase`,
  `GetCurrentUserUseCase`; плюс `BackfillRecentHistoryUseCase` у `domain/usecase/sync/`.
- `AuthRepositoryImpl` (`IAuthRepository`) - шов репозиторію над сервісом автентифікації.
- `IAuthFlowGate` / `InMemoryAuthFlowGate` (`data/service/InMemoryAuthFlowGate.kt`) - позначає
  реєстрацію чи вхід як такі, що тривають, тож шлюз сесії чекає.
- `PhoneNumberRules` (`presentation/components/auth/PhoneNumberRules.kt:11`) - правила введення
  номера для кожної країни (очищення, ліміт довжини, валідність, E.164) на основі libphonenumber.
- `AuthAccountException` (`domain/model/AuthAccountException.kt:3`) - типізована помилка
  невідповідності реєстрації/входу за номером (`AlreadyRegistered`, `NotRegistered`).
- `DeviceRegistry` (`data/device/DeviceRegistry.kt:14`) - записує цей пристрій (і його FCM-токен) під
  користувачем після входу та позначає його вийшовшим при виході.
- `PhoneAuthErrors.kt` (`presentation/base/PhoneAuthErrors.kt:8`) - відображає коди помилок
  phone-автентифікації Firebase на рядки для користувача.
- UI: `presentation/screens/auth/` (`AuthScreen`, `AuthScreenContent`) з
  компонентами в `presentation/components/auth/` (`AuthComponents`, `AuthModeToggle`,
  `CountryCodePicker`, `Countries`).

## Потік входу

1. Користувач обирає країну, вводить національний номер і (під час реєстрації) ім'я користувача.
   Форма живе в `AuthUiState` (`country`, `nationalNumber`, `phoneValid`, `username`, `otp`, `mode`)
   і оновлюється через `SelectMode`, `SelectCountry`, `ChangeNationalNumber`, `ChangeUsername`,
   `ChangeOtp` та `EditPhoneNumber` (див. "Введення телефону" нижче). Коли користувач тисне
   "надіслати", `AuthScreen` перетворює номер на E.164 через `phoneRules.toE164` і нічого не робить,
   якщо він недійсний (`presentation/screens/auth/AuthScreen.kt:74`-`:75`), знаходить Activity-хост
   через `LocalActivity.current` (`:32`) і надсилає `AuthIntent.SendVerificationCode(activity, phoneNumber, username, isRegister)`.
   `handleIntent` (`AuthViewModel.kt:73`) маршрутизує це до `sendVerificationCode`. (Раніше тут був
   каст `LocalContext.current as? Activity`, що дає null за будь-якою `ContextWrapper`, тож кнопка
   мовчки нічого не робила.)
2. `sendVerificationCode` (`AuthViewModel.kt:154`) відкладає `pendingUsername`/`pendingIsRegister`,
   встановлює `isLoading` і викликає `firebaseAuthService.sendVerificationCode(phoneNumber, activity)`.
   Результат - це `VerificationResult`:
   - `CodeSent(verificationId)` -> встановити `codeSent = true` і зберегти `verificationId` в
     `AuthUiState` (`presentation/screens/auth/AuthUiState.kt:11`); UI показує поле OTP.
   - `AutoVerified(credential)` -> Android авто-отримав SMS; одразу переходимо до
     `signInWithPhone` (`AuthViewModel.kt:254`).
   Поки код очікується, `AuthScreen` веде 120-секундне кільце зворотного відліку
   (`AuthScreen.kt:34`-`:44`); коли воно спливає, екран пропонує надіслати повторно.
3. Користувач вводить код; UI надсилає `AuthIntent.VerifyOtpAndLogin(otp)`.
   `verifyOtpAndLogin` (`AuthViewModel.kt:229`) викликає
   `firebaseAuthService.verifyCode(otp, currentState.verificationId)`, щоб збудувати
   `PhoneAuthCredential`, потім `signInWithPhone`.
4. `signInWithPhone` (`AuthViewModel.kt:254`) викликає
   `loginWithPhoneNumberUseCase(credential, pendingUsername, pendingIsRegister)`, який виконує вхід
   і, при першій реєстрації, створює профіль користувача у Firestore. При `Resource.Success` він:
   - виводить присутність онлайн: `presenceManager.goOnline(viewModelScope)` (див.
     [[Presence and Typing_UA|Присутність і набір тексту]]),
   - перезапускає одноразову синхронізацію (`syncCoordinator.restart()`) і відновлює нещодавню
     історію (`restoreRecentHistory`, нижче),
   - синхронізує FCM-токен: `syncFcmTokenForCurrentUser` (`AuthViewModel.kt:323`) отримує токен
     через `FirebaseMessagingManager` і передає `DeviceRegistry.register`, який записує рядок цього
     пристрою в `users/{uid}/devices/{deviceId}` (назва, платформа, версія застосунку, `lastSeenAt`,
     `fcmToken`) і також оновлює застаріле поле `fcmToken` у документі користувача
     (`data/device/DeviceRegistry.kt:20`-`:24`, `:41`-`:55`, `:75`; див.
     [[Notifications_UA|Сповіщення]]),
   - оновлює стан із `currentUser`,
   - випромінює `AuthEffect.AuthSucceeded`.
   `Resource.Failure` з `AuthAccountException` обробляється окремо (наступний розділ).

`resendCode` (`AuthViewModel.kt:195`) віддзеркалює крок 2 для кнопки повторного надсилання й
передає збережений `verificationId`, тож використовується відповідний токен повторного надсилання.

## Введення телефону та валідація

Від PR #39 (злито 2026-11-03) поле телефону валідується для кожної країни, а не приймає будь-які
цифри. `AuthViewModel.updatePhone` (`AuthViewModel.kt:98`) пропускає кожну зміну країни чи цифр через
`PhoneNumberRules`: `sanitize` прибирає нецифрові символи й обмежує довжину найдовшим можливим
номером для коду країни з урахуванням початкового транкового префікса (`PhoneNumberRules.kt:15`-`:28`);
`isValid` питає libphonenumber, чи розібраний номер дійсний для регіону (`:30`, `:35`-`:36`), і
результат стає `AuthUiState.phoneValid`, що вмикає кнопку надсилання. `toE164` (`:32`) дає рядок
`+...` для Firebase. Після надсилання коду вибір країни й поле номера вимикаються
(`enabled = !codeSent`, `presentation/screens/auth/AuthScreenContent.kt:140`, `:148`), а посилання
"Change number" надсилає `AuthIntent.EditPhoneNumber`, що скидає `codeSent`, помилку й код
(`AuthViewModel.kt:94`), тож номер можна знову редагувати.

Форма переживає завершення процесу: режим, ISO-код країни, номер і ім'я користувача
віддзеркалюються в `SavedStateHandle` (`AuthViewModel.kt:77`, `:83`, `:100`-`:101`) і відновлюються
`restoreForm` (`:344`) при перестворенні ViewModel; вихід очищає ці ключі (`:149`). Виправлення
контрасту текстових полів з того ж PR описано в [[Appearance and Theming_UA|Вигляд і теми]].

## Невідповідність реєстрації/входу за номером

PR #53 (злито 2026-11-03) замінив два голі рядки помилок типізованою помилкою.
`AuthRepositoryImpl.performLoginWithPhone` (`data/repository/AuthRepositoryImpl.kt:36`) працює всередині
`authFlowGate.withAuthFlow` (`:32`) і тепер повертає
`Resource.Failure(AuthAccountException.AlreadyRegistered())`, коли хтось обирає Register з номером,
що вже має профіль (спершу виходить зі свіжої сесії), та
`Resource.Failure(AuthAccountException.NotRegistered())`, коли хтось обирає Login з номером без
профілю (видаляє осиротілого користувача Firebase і виходить, якщо видалення не вдалося).
`signInWithPhone` відображає їх (`AuthViewModel.kt:284`-`:304`): `AlreadyRegistered` показує
`auth_error_number_registered` і перемикає на `AuthMode.LOGIN`; `NotRegistered` показує
`auth_error_number_not_registered` і перемикає на `AuthMode.REGISTER`. В обох випадках також
повертається на перший крок (`codeSent = false`, `verificationId = null`, `otp = ""`) і зберігається
новий режим, тож користувач знає, що сталося, і може повторити з правильним режимом, а не дивитися на
мертвий екран OTP. Інші помилки й далі показують повідомлення винятку (`:305`-`:308`).

## Один verification id на потік

`verificationId` і токен повторного надсилання раніше були змінними полями singleton-а
`FirebaseAuthService`, спільними для `AuthViewModel` і `ChangePhoneViewModel`. Запуск верифікації
зміни телефону посеред входу (чи навпаки) перезаписував id іншого потоку, тож потік, що лишився,
будував credential із чужої сесії. Тепер кожен потік тримає отриманий id у власному UiState
(`AuthUiState.verificationId`, `ChangePhoneUiState.verificationId`) і передає його у
`verifyCode(code, verificationId)` (`FirebaseAuthService.kt:89`, повторне надсилання на `:210`). Токени повторного надсилання
живуть у мапі за verification id (`resendTokens`, `FirebaseAuthService.kt:32`), що очищується при
виході.

Отриманий SMS-код і verification id більше не пишуться в logcat. Разом вони і є credential
phone-автентифікації, і логувалися безумовно, а не лише в debug-збірках.

## Шлюз сесії

`AuthViewModel.observeSession` (`AuthViewModel.kt:111`) збирає `ObserveSessionStateUseCase` і
зберігає результат як `AuthUiState.session`. `AppNavigation` показує `MainDisplay` лише для
`SessionState.Active`, `AuthDisplay` - для `SignedOut`, і нічого, поки `Resolving`. `Active` вимагає
підтвердженого профілю у Firestore; реєстрація чи вхід, що тривають, тримають стан у `Resolving`
(`InMemoryAuthFlowGate`), тож екран OTP ніколи не руйнується посеред потоку. Профіль, якого, за
словами сервера, немає, або позначений як видалений, дає `SignedOut` з причиною, і
`endInvalidSession` (`AuthViewModel.kt:120`) відключає присутність, виконує `LogoutUseCase` і показує
`auth_error_account_not_found` або `auth_error_account_deleted`. Промах кешу офлайн лишає сесію на
ідентичності Firebase, тож офлайн-старт ніколи не змушує вийти. Подробиці в
[[Navigation_UA|Навігація]].

## Відновлення історії при вході

Вихід стирає ObjectBox, а `SyncCoordinator.start()` виконувався лише з `Application.onCreate`, тож
вихід і повторний вхід в одному процесі лишали список чатів із порожніми розмовами. При успіху
`signInWithPhone` тепер викликає `syncCoordinator.restart()` (`data/sync/SyncCoordinator.kt:31`) і
`restoreRecentHistory` (`AuthViewModel.kt:315`), що запускає `BackfillRecentHistoryUseCase`
(`domain/usecase/sync/BackfillRecentHistoryUseCase.kt`): синхронізує список розмов, потім тягне 50
останніх повідомлень для кожної з 30 останніх розмов через `backfillRecentMessages`, що поважає
персональну межу `clearedAt` і локальний стан прочитання. Невдала розмова рахується як нуль, а не
перериває весь прохід.

## Прив'язка телефону до наявного акаунта

`AuthIntent.VerifyOtpAndLink` -> `verifyOtpAndLink` (`AuthViewModel.kt:243`) -> `linkPhone`
(`AuthViewModel.kt:363`) збирає `LinkPhoneUseCase(credential)` і випромінює `AuthEffect.PhoneLinked`
при успіху. Використовується потоком зміни телефону (`ChangePhoneScreen`, `ChangePhoneViewModel`),
який тримає власний verification id.

## Вихід

`logout` (`AuthViewModel.kt:144`) зупиняє координатор синхронізації, від'єднує присутність
(`presenceManager.disconnect(viewModelScope)`, який тепер також скасовує збирач, що раніше
протікав), виконує `LogoutUseCase()` і скидає стан до свіжого `AuthUiState()`. `LogoutUseCase`
(`domain/usecase/auth/LogoutUseCase.kt`) позначає користувача офлайн, виконує
`sessionCleaner.clearOnLogout(uid)`, виходить із Firebase і скидає ObjectBox один раз
(дубльований `resetDB()` усередині `FirebaseAuthService.logout()` прибрано).
`AndroidSessionCleaner.clearOnLogout` (`data/service/AndroidSessionCleaner.kt:35`):

- зупиняє відтворення кліпів і скасовує будь-який голосовий чи відеозапис у головному потоці,
  потім очищає чернетки кліпів і зупиняє E2EE bootstrap (`:36`-`:42`),
- виводить цей пристрій у Firestore (`DeviceRegistry.signOut`: прибирає `fcmToken` рядка пристрою і
  ставить `signedOutAt`, а також прибирає застаріле поле токена, якщо воно ще містить цей токен),
  видаляє FCM-токен на пристрої і скасовує всі сповіщення (`:44`-`:51`),
- стирає кеш розшифрованих медіа, outbox і чернетки кліпів та дисковий кеш Coil
  (`:53`-`:56`).

Робота, що переживає екран, теж зупиняється: `ChatsViewModel` запускає слухача розмов усередині
`flatMapLatest` над станом автентифікації (`presentation/screens/chats/ChatsViewModel.kt:87`), тож вихід
скасовує його, а не дає записати розмови старого акаунта назад після скидання, а `SyncWorker`
одразу повертається, коли ніхто не ввійшов (`data/sync/SyncWorker.kt:21`). Оскільки сесія стає
`SignedOut`, застосунок повертається до `AuthDisplay`.

Рядок "Add account" на екрані зміни акаунта прибрано: він пушив `AuthRoute` у стек `MainDisplay`,
де для нього немає запису, тож падав при натисканні. Мульти-акаунт - заглушка, а сховище ObjectBox
розраховане на одного користувача, тож рядок повернеться лише разом із мульти-акаунтом.

## Обробка помилок

Кожен збій стає `UiText` у стані. Збої phone-верифікації проходять через
`Throwable.toPhoneAuthUiText()` (`presentation/base/PhoneAuthErrors.kt:8`), що відображає
`FirebaseTooManyRequestsException` і коди `FirebaseAuthException` (скасована reCAPTCHA, вебконтекст
уже показано, застосунок не верифіковано, перевищено квоту, неправильний номер, неправильний код,
...) на рядкові ресурси; невідомі коди й далі показують повідомлення винятку, а null дає запасний
`auth_error_verification_failed`. Неправильний OTP встановлює
`UiText.StringResource(R.string.auth_error_invalid_code)` (`AuthViewModel.kt:238`). Екран резолвить
його під час рендеру. Див. [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] щодо `UiText`.

## Feature flag

Вхід, реєстрація й вихід не за feature flag: жоден `FeatureFlag` не читається в `AuthViewModel`,
`AuthScreen` чи `AuthScreenContent` (див. [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]]). Дві сусідні функції
акаунта мають прапорці, обидві лише в UI: `PROFILE_CHANGE_PHONE` ховає кнопку "Change", що
запускає потік зміни телефону
(`presentation/screens/editprofile/EditProfileScreenContent.kt:75`), а `ACCOUNTS_SWITCH` ховає рядок
"Switch account" (`EditProfileScreenContent.kt:86`) і змушує
`ChangeAccountViewModel.selectAccount` одразу повертатися
(`presentation/screens/changeaccount/ChangeAccountViewModel.kt:32`). Перевірки `PROFILE_CHANGE_PHONE`
у `ChangePhoneViewModel` не знайдено.

## Чому такий дизайн

- Firebase phone auth опрацьовує доставку SMS, авто-отримання й керування credential, тож застосунок
  не несе жодного OTP-бекенду.
- Тримання побічних ефектів присутності, синхронізації й FCM-токена всередині гілки успіху
  `signInWithPhone` означає, що сесія повністю ініціалізована (онлайн, історію відновлено, досяжна
  пушем) у мить завершення входу.
- Керування навігацією зі спостережуваної розв'язаної сесії уникає імперативної
  навігації-при-вході й тримає єдине джерело істини, ніколи не показуючи основний UI акаунту без
  профілю.
- Verification id на потік прибирає спільний змінний стан із singleton-а, яким користуються два
  екрани.
- Типізовані помилки `AuthAccountException` (замість рядків) дозволяють ViewModel вирішити, що
  робити далі (змінити режим, повернутися на перший крок), а не просто друкувати текст.

Альтернативи (email/пароль, вхід через Google) у коді відсутні; вхід за номером телефону - єдиний
реалізований метод.

## Пов'язане

[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] -
[[Presence and Typing_UA|Присутність і набір тексту]] - [[Notifications_UA|Сповіщення]] -
[[Navigation_UA|Навігація]] - [[Data Layer_UA|Шар даних]] - [[Settings_UA|Налаштування]] -
[[Voice Messages_UA|Голосові повідомлення]]
