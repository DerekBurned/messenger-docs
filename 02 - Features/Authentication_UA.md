---
tags:
  - _UA
---
# Автентифікація

Вхід за номером телефону з одноразовим кодом по SMS, на основі Firebase Auth. Також підтримує
прив'язку номера телефону до наявного акаунта. Чи показувати основний UI, вирішує розв'язана сесія
(ідентичність Firebase плюс підтверджений профіль у Firestore), а не лише прапорець Firebase.

## Складові

- `AuthViewModel` (`presentation/viewmodel/AuthViewModel.kt:34`) - MVI-ViewModel
  (`MviViewModel<AuthUiState, AuthIntent, AuthEffect>`).
- `FirebaseAuthService` (`data/remote/auth/FirebaseAuthService.kt`) - обгортає phone-верифікацію
  `FirebaseAuth`.
- Use case (`domain/usecase/auth/`): `LoginWithPhoneNumberUseCase`, `LinkPhoneUseCase`,
  `LogoutUseCase`, `ObserveAuthStateUseCase`, `ObserveSessionStateUseCase`,
  `GetCurrentUserUseCase`; плюс `BackfillRecentHistoryUseCase` у `domain/usecase/sync/`.
- `AuthRepositoryImpl` (`IAuthRepository`) - шов репозиторію над сервісом автентифікації.
- `IAuthFlowGate` / `InMemoryAuthFlowGate` (`data/service/InMemoryAuthFlowGate.kt`) - позначає
  реєстрацію чи вхід як такі, що тривають, тож шлюз сесії чекає.
- `PhoneAuthErrors.kt` (`presentation/base/PhoneAuthErrors.kt:8`) - відображає коди помилок
  phone-автентифікації Firebase на рядки для користувача.
- UI: `presentation/screens/auth/` (`AuthScreen`, `AuthScreenContent`, `AuthScreenPreviews`) з
  компонентами в `presentation/components/auth/` (`AuthComponents`, `AuthModeToggle`,
  `CountryCodePicker`, `Countries`).

## Потік входу

1. Користувач вводить номер телефону і (під час реєстрації) ім'я користувача. `AuthScreen` знаходить
   Activity-хост через `LocalActivity.current` (`presentation/screens/auth/AuthScreen.kt:40`) і
   надсилає `AuthIntent.SendVerificationCode(activity, phoneNumber, username, isRegister)`.
   `handleIntent` (`AuthViewModel.kt:64`) маршрутизує це до `sendVerificationCode`. (Раніше тут був
   каст `LocalContext.current as? Activity`, що дає null за будь-якою `ContextWrapper`, тож кнопка
   мовчки нічого не робила.)
2. `sendVerificationCode` (`AuthViewModel.kt:112`) відкладає `pendingUsername`/`pendingIsRegister`,
   встановлює `isLoading` і викликає `firebaseAuthService.sendVerificationCode(phoneNumber, activity)`.
   Результат - це `VerificationResult`:
   - `CodeSent(verificationId)` -> встановити `codeSent = true` і зберегти `verificationId` в
     `AuthUiState` (`presentation/state/AuthUiState.kt:14`); UI показує поле OTP.
   - `AutoVerified(credential)` -> Android авто-отримав SMS; одразу переходимо до
     `signInWithPhone` (`AuthViewModel.kt:212`).
3. Користувач вводить код; UI надсилає `AuthIntent.VerifyOtpAndLogin(otp)`.
   `verifyOtpAndLogin` (`AuthViewModel.kt:187`) викликає
   `firebaseAuthService.verifyCode(otp, currentState.verificationId)`, щоб збудувати
   `PhoneAuthCredential`, потім `signInWithPhone`.
4. `signInWithPhone` (`AuthViewModel.kt:212`) викликає
   `loginWithPhoneNumberUseCase(credential, pendingUsername, pendingIsRegister)`, який виконує вхід
   і, при першій реєстрації, створює профіль користувача у Firestore. При `Resource.Success` він:
   - виводить присутність онлайн: `presenceManager.goOnline(viewModelScope)` (див.
     [[Presence and Typing_UA|Присутність і набір тексту]]),
   - перезапускає одноразову синхронізацію (`syncCoordinator.restart()`) і відновлює нещодавню
     історію (`restoreRecentHistory`, нижче),
   - синхронізує FCM-токен: `syncFcmTokenForCurrentUser` (`AuthViewModel.kt:259`) отримує токен
     через `FirebaseMessagingManager` і записує його у Firestore (див.
     [[Notifications_UA|Сповіщення]]),
   - оновлює стан із `currentUser`,
   - випромінює `AuthEffect.AuthSucceeded`.

`resendCode` (`AuthViewModel.kt:153`) віддзеркалює крок 2 для кнопки повторного надсилання й
передає збережений `verificationId`, тож використовується відповідний токен повторного надсилання.

## Один verification id на потік

`verificationId` і токен повторного надсилання раніше були змінними полями singleton-а
`FirebaseAuthService`, спільними для `AuthViewModel` і `ChangePhoneViewModel`. Запуск верифікації
зміни телефону посеред входу (чи навпаки) перезаписував id іншого потоку, тож потік, що лишився,
будував credential із чужої сесії. Тепер кожен потік тримає отриманий id у власному UiState
(`AuthUiState.verificationId`, `ChangePhoneUiState.verificationId`) і передає його у
`verifyCode(code, verificationId)` (`FirebaseAuthService.kt:89`). Токени повторного надсилання
живуть у мапі за verification id (`resendTokens`, `FirebaseAuthService.kt:32`), що очищується при
виході.

Отриманий SMS-код і verification id більше не пишуться в logcat. Разом вони і є credential
phone-автентифікації, і логувалися безумовно, а не лише в debug-збірках.

## Шлюз сесії

`AuthViewModel.observeSession` (`AuthViewModel.kt:78`) збирає `ObserveSessionStateUseCase` і
зберігає результат як `AuthUiState.session`. `AppNavigation` показує `MainDisplay` лише для
`SessionState.Active`, `AuthDisplay` - для `SignedOut`, і нічого, поки `Resolving`. `Active` вимагає
підтвердженого профілю у Firestore; реєстрація чи вхід, що тривають, тримають стан у `Resolving`
(`InMemoryAuthFlowGate`), тож екран OTP ніколи не руйнується посеред потоку. Профіль, якого, за
словами сервера, немає, або позначений як видалений, дає `SignedOut` з причиною, і
`endInvalidSession` (`AuthViewModel.kt:87`) виводить пристрій і показує
`auth_error_account_not_found` або `auth_error_account_deleted`. Промах кешу офлайн лишає сесію на
ідентичності Firebase, тож офлайн-старт ніколи не змушує вийти. Подробиці в
[[Navigation_UA|Навігація]].

## Відновлення історії при вході

Вихід стирає ObjectBox, а `SyncCoordinator.start()` виконувався лише з `Application.onCreate`, тож
вихід і повторний вхід в одному процесі лишали список чатів із порожніми розмовами. При успіху
`signInWithPhone` тепер викликає `syncCoordinator.restart()` (`data/sync/SyncCoordinator.kt:31`) і
`restoreRecentHistory` (`AuthViewModel.kt:251`), що запускає `BackfillRecentHistoryUseCase`
(`domain/usecase/sync/BackfillRecentHistoryUseCase.kt`): синхронізує список розмов, потім тягне 50
останніх повідомлень для кожної з 30 останніх розмов через `backfillRecentMessages`, що поважає
персональну межу `clearedAt` і локальний стан прочитання. Невдала розмова рахується як нуль, а не
перериває весь прохід.

## Прив'язка телефону до наявного акаунта

`AuthIntent.VerifyOtpAndLink` -> `verifyOtpAndLink` (`AuthViewModel.kt:201`) -> `linkPhone`
(`AuthViewModel.kt:277`) збирає `LinkPhoneUseCase(credential)` і випромінює `AuthEffect.PhoneLinked`
при успіху. Використовується потоком зміни телефону (`ChangePhoneScreen`, `ChangePhoneViewModel`),
який тримає власний verification id.

## Вихід

`logout` (`AuthViewModel.kt:103`) зупиняє координатор синхронізації, від'єднує присутність
(`presenceManager.disconnect(viewModelScope)`, який тепер також скасовує збирач, що раніше
протікав), виконує `LogoutUseCase()` і скидає стан до свіжого `AuthUiState()`. `LogoutUseCase`
(`domain/usecase/auth/LogoutUseCase.kt`) позначає користувача офлайн, виконує
`sessionCleaner.clearOnLogout(uid)`, виходить із Firebase і скидає ObjectBox один раз
(дубльований `resetDB()` усередині `FirebaseAuthService.logout()` прибрано).
`AndroidSessionCleaner.clearOnLogout` (`data/service/AndroidSessionCleaner.kt:31`):

- зупиняє відтворення голосу й скасовує запис у головному потоці, потім очищає голосові чернетки,
- видаляє FCM-токен у Firestore і на пристрої, скасовує всі сповіщення,
- стирає кеш розшифрованих медіа, outbox і чернетки голосових кліпів та дисковий кеш Coil
  (`:47`-`:50`).

Робота, що переживає екран, теж зупиняється: `ChatsViewModel` запускає слухача розмов усередині
`flatMapLatest` над станом автентифікації (`presentation/viewmodel/ChatsViewModel.kt:84`), тож вихід
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
`UiText.StringResource(R.string.auth_error_invalid_code)` (`AuthViewModel.kt:196`). Екран резолвить
його під час рендеру. Див. [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] щодо `UiText`.

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

Альтернативи (email/пароль, вхід через Google) у коді відсутні; вхід за номером телефону - єдиний
реалізований метод.

## Пов'язане

[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] -
[[Presence and Typing_UA|Присутність і набір тексту]] - [[Notifications_UA|Сповіщення]] -
[[Navigation_UA|Навігація]] - [[Data Layer_UA|Шар даних]] - [[Settings_UA|Налаштування]] -
[[Voice Messages_UA|Голосові повідомлення]]
