---
tags:
  - _UA
---
# Автентифікація

Вхід за номером телефону з одноразовим кодом по SMS, на основі Firebase Auth. Також підтримує
прив'язку номера телефону до наявного акаунта.

## Складові

- `AuthViewModel` (`presentation/viewmodel/AuthViewModel.kt:39`) - MVI-ViewModel
  (`MviViewModel<AuthUiState, AuthIntent, AuthEffect>`).
- `FirebaseAuthService` (`data/remote/auth/FirebaseAuthService.kt`) - обгортає phone-верифікацію
  `FirebaseAuth`.
- Use case (`domain/usecase/auth/`): `LoginWithPhoneNumberUseCase`, `LinkPhoneUseCase`,
  `LogoutUseCase`, `ObserveAuthStateUseCase`, `GetCurrentUserUseCase`.
- `AuthRepositoryImpl` (`IAuthRepository`) - шов репозиторію над сервісом автентифікації.
- UI: `AuthScreen` (`presentation/screens/AuthScreen.kt`) з компонентами в
  `presentation/components/auth/` (`AuthComponents`, `AuthModeToggle`, `CountryCodePicker`,
  `Countries`).

## Потік входу

1. Користувач вводить номер телефону і (під час реєстрації) ім'я користувача. `AuthScreen`
   надсилає `AuthIntent.SendVerificationCode(activity, phoneNumber, username, isRegister)`.
   `handleIntent` (`AuthViewModel.kt:52`) маршрутизує це до `sendVerificationCode`.
2. `sendVerificationCode` (`AuthViewModel.kt:82`) відкладає `pendingUsername`/`pendingIsRegister`,
   встановлює `isLoading` і викликає `firebaseAuthService.sendVerificationCode(phoneNumber, activity)`.
   Результат - це `VerificationResult`:
   - `CodeSent(verificationId)` -> встановити `codeSent = true`; UI показує поле OTP.
   - `AutoVerified(credential)` -> Android авто-отримав SMS; одразу переходимо до
     `signInWithPhone` (`AuthViewModel.kt:103`).
3. Користувач вводить код; UI надсилає `AuthIntent.VerifyOtpAndLogin(otp)`.
   `verifyOtpAndLogin` (`AuthViewModel.kt:141`) викликає `firebaseAuthService.verifyCode(otp)`, щоб
   збудувати `PhoneAuthCredential`, потім `signInWithPhone`.
4. `signInWithPhone` (`AuthViewModel.kt:166`) викликає
   `loginWithPhoneNumberUseCase(credential, pendingUsername, pendingIsRegister)`, який виконує вхід
   і, при першій реєстрації, створює профіль користувача у Firestore. При `Resource.Success` він:
   - виводить присутність онлайн: `presenceManager.goOnline(viewModelScope)` (див.
     [[Presence and Typing_UA|Присутність і набір тексту]]),
   - синхронізує FCM-токен: `syncFcmTokenForCurrentUser` (`AuthViewModel.kt:202`) отримує токен
     через `FirebaseMessagingManager` і записує його у Firestore (див.
     [[Notifications_UA|Сповіщення]]),
   - оновлює стан із `currentUser`,
   - випромінює `AuthEffect.AuthSucceeded`.

`resendCode` (`AuthViewModel.kt:117`) віддзеркалює крок 2 для кнопки повторного надсилання.

## Стан автентифікації як верхньорівневий перемикач застосунку

`observeAuthStatus` (`AuthViewModel.kt:66`) збирає `observeAuthStateUseCase()` і встановлює
`isAuthenticated = user != null`. `AppNavigation` збирає `AuthViewModel.state`
(`presentation/navigation/navigation.kt:82`) і показує `MainDisplay`, коли автентифіковано, інакше
`AuthDisplay` - тож вхід/вихід перемикає все дерево навігації без явного виклику навігації. Див.
[[Navigation_UA|Навігація]].

## Прив'язка телефону до наявного акаунта

`AuthIntent.VerifyOtpAndLink` -> `verifyOtpAndLink` (`AuthViewModel.kt:155`) -> `linkPhone`
(`AuthViewModel.kt:220`) збирає `LinkPhoneUseCase(credential)` і випромінює `AuthEffect.PhoneLinked`
при успіху. Використовується потоком зміни телефону (`ChangePhoneScreen`, `ChangePhoneViewModel`).

## Вихід

`logout` (`AuthViewModel.kt:74`) прибирає присутність (`presenceManager.disconnect(viewModelScope)`),
виконує `LogoutUseCase()` (вихід із Firebase плюс локальне очищення сесії через
`ISessionCleaner`/`AndroidSessionCleaner`) і скидає стан до свіжого `AuthUiState()`. Оскільки
навігація тримається на `isAuthenticated`, застосунок повертається до `AuthDisplay`.

## Обробка помилок

Кожен збій стає `UiText` у стані - або повідомлення винятку
(`result.exceptionOrNull()?.message?.toUiText()`), або запасний ресурс, як-от
`UiText.StringResource(R.string.auth_error_invalid_code)` (`AuthViewModel.kt:150`). Екран резолвить
його під час рендеру. Див. [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] щодо `UiText`.

## Чому такий дизайн

- Firebase phone auth опрацьовує доставку SMS, авто-отримання й керування credential, тож застосунок
  не несе жодного OTP-бекенду.
- Тримання побічних ефектів присутності й FCM-токена всередині гілки успіху `signInWithPhone`
  означає, що сесія повністю ініціалізована (онлайн + досяжна пушем) у мить завершення входу.
- Керування навігацією зі спостережуваного стану автентифікації уникає імперативної
  навігації-при-вході/виході й тримає єдине джерело істини.

Альтернативи (email/пароль, вхід через Google) у коді відсутні; вхід за номером телефону - єдиний
реалізований метод.

## Пов'язане

[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] -
[[Presence and Typing_UA|Присутність і набір тексту]] - [[Notifications_UA|Сповіщення]] -
[[Navigation_UA|Навігація]] - [[Data Layer_UA|Шар даних]]
