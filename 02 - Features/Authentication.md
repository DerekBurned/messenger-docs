 # Authentication

Phone-number sign-in with SMS one-time code, backed by Firebase Auth. Also supports linking a
phone number to an existing account.

## Pieces

- `AuthViewModel` (`presentation/viewmodel/AuthViewModel.kt:39`) - MVI ViewModel
  (`MviViewModel<AuthUiState, AuthIntent, AuthEffect>`).
- `FirebaseAuthService` (`data/remote/auth/FirebaseAuthService.kt`) - wraps
  `FirebaseAuth` phone verification.
- Use cases (`domain/usecase/auth/`): `LoginWithPhoneNumberUseCase`, `LinkPhoneUseCase`,
  `LogoutUseCase`, `ObserveAuthStateUseCase`, `GetCurrentUserUseCase`.
- `AuthRepositoryImpl` (`IAuthRepository`) - repository seam over the auth service.
- UI: `AuthScreen` (`presentation/screens/AuthScreen.kt`) with components in
  `presentation/components/auth/` (`AuthComponents`, `AuthModeToggle`, `CountryCodePicker`,
  `Countries`).

## Sign-in flow

1. The user enters phone number and (on register) username. `AuthScreen` dispatches
   `AuthIntent.SendVerificationCode(activity, phoneNumber, username, isRegister)`.
   `handleIntent` (`AuthViewModel.kt:52`) routes it to `sendVerificationCode`.
2. `sendVerificationCode` (`AuthViewModel.kt:82`) stashes `pendingUsername`/`pendingIsRegister`,
   sets `isLoading`, and calls `firebaseAuthService.sendVerificationCode(phoneNumber, activity)`.
   The result is a `VerificationResult`:
   - `CodeSent(verificationId)` -> set `codeSent = true`; the UI shows the OTP field.
   - `AutoVerified(credential)` -> Android auto-retrieved the SMS; go straight to
     `signInWithPhone` (`AuthViewModel.kt:103`).
3. The user types the code; the UI dispatches `AuthIntent.VerifyOtpAndLogin(otp)`.
   `verifyOtpAndLogin` (`AuthViewModel.kt:141`) calls `firebaseAuthService.verifyCode(otp)` to
   build a `PhoneAuthCredential`, then `signInWithPhone`.
4. `signInWithPhone` (`AuthViewModel.kt:166`) calls
   `loginWithPhoneNumberUseCase(credential, pendingUsername, pendingIsRegister)`, which signs in
   and, on first registration, creates the Firestore user profile. On `Resource.Success` it:
   - brings presence online: `presenceManager.goOnline(viewModelScope)` (see
     [[Presence and Typing]]),
   - syncs the FCM token: `syncFcmTokenForCurrentUser` (`AuthViewModel.kt:202`) fetches the
     token via `FirebaseMessagingManager` and writes it to Firestore (see [[Notifications]]),
   - updates state with `currentUser`,
   - emits `AuthEffect.AuthSucceeded`.

`resendCode` (`AuthViewModel.kt:117`) mirrors step 2 for the resend button.

## Auth state as the app's top-level switch

`observeAuthStatus` (`AuthViewModel.kt:66`) collects `observeAuthStateUseCase()` and sets
`isAuthenticated = user != null`. `AppNavigation` collects `AuthViewModel.state`
(`presentation/navigation/navigation.kt:82`) and shows `MainDisplay` when authenticated,
`AuthDisplay` otherwise - so login/logout flips the whole navigation tree with no explicit
navigate call. See [[Navigation]].

## Linking a phone to an existing account

`AuthIntent.VerifyOtpAndLink` -> `verifyOtpAndLink` (`AuthViewModel.kt:155`) -> `linkPhone`
(`AuthViewModel.kt:220`) collects `LinkPhoneUseCase(credential)` and emits
`AuthEffect.PhoneLinked` on success. Used by the change-phone flow (`ChangePhoneScreen`,
`ChangePhoneViewModel`).

## Logout

`logout` (`AuthViewModel.kt:74`) removes presence
(`presenceManager.disconnect(viewModelScope)`), runs `LogoutUseCase()` (Firebase sign-out plus
local session cleanup via `ISessionCleaner`/`AndroidSessionCleaner`), and resets state to a
fresh `AuthUiState()`. Because navigation keys off `isAuthenticated`, the app returns to
`AuthDisplay`.

## Error handling

Every failure becomes a `UiText` in state - either the exception message
(`result.exceptionOrNull()?.message?.toUiText()`) or a resource fallback such as
`UiText.StringResource(R.string.auth_error_invalid_code)` (`AuthViewModel.kt:150`). The screen
resolves it at render time. See [[Presentation Layer (MVI)]] for `UiText`.

## Why this design

- Firebase phone auth handles SMS delivery, auto-retrieval, and credential management, so the
  app carries no OTP backend.
- Keeping presence and FCM-token side effects inside the success branch of `signInWithPhone`
  means a session is fully bootstrapped (online + reachable by push) the moment login
  completes.
- Driving navigation from observed auth state avoids imperative navigate-on-login/logout and
  keeps a single source of truth.

Alternatives (email/password, Google sign-in) are not present in the code; phone-number is the
only implemented method.

## Related

[[Presentation Layer (MVI)]] - [[Presence and Typing]] - [[Notifications]] - [[Navigation]] -
[[Data Layer]]
