# Authentication - Code

Companion to [[Authentication]]. This note gathers the actual source for the authentication
feature and explains it block by block. Conceptual overview and rationale live in the main note;
here the focus is the code itself. Snippets are copied from the repo with `Log.d` lines and blank
lines trimmed.

Files covered:
- MVI contract: `presentation/screens/auth/AuthUiState.kt`, `presentation/screens/auth/AuthIntent.kt`,
  `presentation/screens/auth/AuthEffect.kt`
- `presentation/screens/auth/AuthViewModel.kt`
- Use cases: `domain/usecase/auth/{LoginWithPhoneNumberUseCase, LinkPhoneUseCase, LogoutUseCase,
  ObserveAuthStateUseCase, ObserveSessionStateUseCase, GetCurrentUserUseCase}.kt`
- `data/repository/AuthRepositoryImpl.kt`
- `data/remote/auth/FirebaseAuthService.kt`
- UI (described, not embedded): `presentation/screens/auth/` and `presentation/components/auth/*`

---

## 1. MVI contract

`presentation/screens/auth/AuthUiState.kt` - the entire screen state:

```kotlin
data class AuthUiState(
    val isLoading: Boolean = false,
    val currentUser: User? = null,
    val error: UiText? = null,
    val session: SessionState = SessionState.SignedOut(),
    val codeSent: Boolean = false,
    val verificationId: String? = null,
    val mode: AuthMode = AuthMode.REGISTER,
    val country: Country = Countries.default,
    val nationalNumber: String = "",
    val phoneValid: Boolean = false,
    val username: String = "",
    val otp: String = "",
) : UiState
```

The first six fields carry the SMS flow: `isLoading` (spinner), `codeSent` (switches the UI from the
phone field to the OTP field), `verificationId` (the id of this flow's SMS session, passed back
to `verifyCode` and `resendVerificationCode`), `session` (the top-level nav switch, see
[[Navigation]]), `error` as a `UiText`, and `currentUser` after success. Since PR #39 the form itself
(`mode`, `country`, `nationalNumber`, `phoneValid`, `username`, `otp`) also lives here, so it survives
recomposition and, through `SavedStateHandle`, process death.

`presentation/screens/auth/AuthIntent.kt` - every action the screen can send:

```kotlin
sealed interface AuthIntent : UiIntent {
    data class SelectMode(val mode: AuthMode) : AuthIntent
    data class SelectCountry(val country: Country) : AuthIntent
    data class ChangeNationalNumber(val input: String) : AuthIntent
    data class ChangeUsername(val username: String) : AuthIntent
    data class ChangeOtp(val otp: String) : AuthIntent

    data class SendVerificationCode(
        val activity: Activity,
        val phoneNumber: String,
        val username: String?,
        val isRegister: Boolean,
    ) : AuthIntent

    data class ResendCode(
        val activity: Activity,
        val phoneNumber: String,
    ) : AuthIntent

    data class VerifyOtpAndLogin(val otpCode: String) : AuthIntent
    data class VerifyOtpAndLink(val otpCode: String) : AuthIntent
    data object Logout : AuthIntent
    data object ClearError : AuthIntent
    data object EditPhoneNumber : AuthIntent
}
```

The `Activity` rides inside the intent because Firebase phone auth needs one for reCAPTCHA /
SMS auto-retrieval (`PhoneAuthOptions.setActivity`); the screen obtains it with
`LocalActivity.current`. `isRegister` distinguishes login from registration so the repository
can enforce the "already registered / not registered" rules.

`presentation/screens/auth/AuthEffect.kt` - one-shot signals up to the UI:

```kotlin
sealed interface AuthEffect : UiEffect {
    data object AuthSucceeded : AuthEffect
    data object PhoneLinked : AuthEffect
}
```

---

## 2. AuthViewModel

`presentation/screens/auth/AuthViewModel.kt` (logging trimmed):

```kotlin
@HiltViewModel
class AuthViewModel @Inject constructor(
    private val loginWithPhoneNumberUseCase: LoginWithPhoneNumberUseCase,
    private val linkPhoneUseCase: LinkPhoneUseCase,
    private val logoutUseCase: LogoutUseCase,
    private val observeSessionStateUseCase: ObserveSessionStateUseCase,
    private val firebaseAuthService: FirebaseAuthService,
    private val presenceManager: PresenceManager,
    private val firebaseMessagingManager: FirebaseMessagingManager,
    private val deviceRegistry: DeviceRegistry,
    private val syncCoordinator: SyncCoordinator,
    private val backfillRecentHistoryUseCase: BackfillRecentHistoryUseCase,
    private val savedStateHandle: SavedStateHandle,
) : MviViewModel<AuthUiState, AuthIntent, AuthEffect>(
    initialState = restoreForm(
        savedStateHandle,
        AuthUiState(
            session = if (firebaseAuthService.isAuthenticated()) {
                SessionState.Resolving
            } else {
                SessionState.SignedOut()
            },
        ),
    ),
) {
    private val phoneRules = PhoneNumberRules()
    private var pendingUsername: String? = null
    private var pendingIsRegister: Boolean = false

    init {
        observeSession()
    }

    override fun handleIntent(intent: AuthIntent) {
        when (intent) {
            is AuthIntent.SelectMode -> { /* save mode in SavedStateHandle; setState(mode, error = null) */ }
            is AuthIntent.SelectCountry -> updatePhone(intent.country, currentState.nationalNumber)
            is AuthIntent.ChangeNationalNumber -> updatePhone(currentState.country, intent.input)
            is AuthIntent.ChangeUsername -> { /* save + setState(username) */ }
            is AuthIntent.ChangeOtp -> setState { copy(otp = intent.otp) }
            is AuthIntent.SendVerificationCode ->
                sendVerificationCode(intent.activity, intent.phoneNumber, intent.username, intent.isRegister)
            is AuthIntent.ResendCode -> resendCode(intent.activity, intent.phoneNumber)
            is AuthIntent.VerifyOtpAndLogin -> verifyOtpAndLogin(intent.otpCode)
            is AuthIntent.VerifyOtpAndLink -> verifyOtpAndLink(intent.otpCode)
            AuthIntent.Logout -> logout()
            AuthIntent.ClearError -> setState { copy(error = null) }
            AuthIntent.EditPhoneNumber -> setState { copy(codeSent = false, error = null, otp = "") }
        }
    }

    private fun observeSession() {
        viewModelScope.launch {
            observeSessionStateUseCase().collect { session ->
                setState { copy(session = session) }
                (session as? SessionState.SignedOut)?.reason?.let { endInvalidSession(it) }
            }
        }
    }

    private suspend fun endInvalidSession(reason: SessionEndReason) {
        if (!firebaseAuthService.isAuthenticated()) return
        presenceManager.disconnect(viewModelScope)
        logoutUseCase()
        setState { copy(isLoading = false, error = reason.toUiText()) }
    }

    private fun logout() {
        viewModelScope.launch {
            syncCoordinator.stop()
            presenceManager.disconnect(viewModelScope)
            logoutUseCase()
            setState { AuthUiState() }
        }
    }

    private fun sendVerificationCode(
        activity: Activity,
        phoneNumber: String,
        username: String?,
        isRegister: Boolean,
    ) {
        pendingUsername = username
        pendingIsRegister = isRegister
        setState { copy(isLoading = true, error = null, codeSent = false) }

        viewModelScope.launch {
            val result = firebaseAuthService.sendVerificationCode(phoneNumber, activity)
            if (result.isSuccess) {
                when (val verifyResult = result.getOrThrow()) {
                    is VerificationResult.CodeSent -> {
                        setState {
                            copy(
                                isLoading = false,
                                codeSent = true,
                                verificationId = verifyResult.verificationId,
                            )
                        }
                    }
                    is VerificationResult.AutoVerified -> {
                        signInWithPhone(verifyResult.credential)
                    }
                }
            } else {
                val errorMsg: UiText = result.exceptionOrNull()?.toPhoneAuthUiText()
                    ?: UiText.StringResource(R.string.auth_error_verification_failed)
                setState { copy(isLoading = false, error = errorMsg) }
            }
        }
    }

    private fun verifyOtpAndLogin(otpCode: String) {
        viewModelScope.launch {
            val credentialResult = firebaseAuthService.verifyCode(otpCode, currentState.verificationId)
            if (credentialResult.isSuccess) {
                signInWithPhone(credentialResult.getOrThrow())
            } else {
                setState { copy(error = UiText.StringResource(R.string.auth_error_invalid_code)) }
            }
        }
    }

    private fun signInWithPhone(credential: PhoneAuthCredential) {
        viewModelScope.launch {
            setState { copy(isLoading = true) }
            val result = loginWithPhoneNumberUseCase(credential, pendingUsername, pendingIsRegister)
            pendingUsername = null
            pendingIsRegister = false
            when (result) {
                is Resource.Success -> {
                    presenceManager.goOnline(viewModelScope)
                    syncCoordinator.restart()
                    restoreRecentHistory()
                    syncFcmTokenForCurrentUser()
                    setState {
                        copy(
                            isLoading = false,
                            currentUser = result.data,
                            error = null,
                        )
                    }
                    emitEffect(AuthEffect.AuthSucceeded)
                }
                is Resource.Error -> {
                    setState { copy(isLoading = false, error = result.message.toUiText()) }
                }
                is Resource.Failure -> when (val exception = result.exception) {
                    is AuthAccountException -> {
                        // AlreadyRegistered -> message auth_error_number_registered, mode LOGIN
                        // NotRegistered     -> message auth_error_number_not_registered, mode REGISTER
                        // save the new mode, then back to step one:
                        setState { copy(isLoading = false, codeSent = false, verificationId = null, otp = "", mode = mode, error = UiText.StringResource(message)) }
                    }
                    else -> setState { copy(isLoading = false, error = exception.message?.toUiText()) }
                }
                is Resource.Loading -> {}
            }
        }
    }

    private fun restoreRecentHistory() {
        viewModelScope.launch {
            backfillRecentHistoryUseCase()
                .onSuccess { Log.d(TAG, "restoreRecentHistory: pulled $it messages") }
                .onFailure { Log.w(TAG, "restoreRecentHistory failed", it) }
        }
    }
}
```

`resendCode` and `verifyOtpAndLink` follow the same shapes: `resendCode` calls
`firebaseAuthService.resendVerificationCode(phoneNumber, activity, currentState.verificationId)`
and stores the new id on `CodeSent`; `verifyOtpAndLink` builds the credential the same way and
hands it to `linkPhone`, which collects `LinkPhoneUseCase` and emits `AuthEffect.PhoneLinked`.
`syncFcmTokenForCurrentUser` fetches the FCM token and registers it with
`deviceRegistry.register(uid, token)` (a per-device row, see [[Notifications]]). `updatePhone` runs the
input through `PhoneNumberRules` (`sanitize`, `isValid`) and mirrors country and number into
`SavedStateHandle`; `restoreForm` rebuilds the form from it. The `AuthAccountException` branch of
`signInWithPhone` (PR #53) is in `AuthViewModel.kt:284`-`:304` and is explained in [[Authentication]].

Explanation of the moving parts:

- The initial `session` is `Resolving` for an already-signed-in user, so nothing is shown until
  the profile is confirmed, and `SignedOut()` otherwise, so the auth screen shows at once.
- `pendingUsername`/`pendingIsRegister` are held between the "send code" step and the eventual
  `signInWithPhone`, because the code arrives asynchronously (or auto-verifies) and the username
  is only needed at sign-in time.
- `observeSession` mirrors `ObserveSessionStateUseCase` into state; a `SignedOut` that carries a
  reason (profile missing, account deleted) also signs this device out through
  `endInvalidSession`.
- Both `CodeSent` and `AutoVerified` are handled: Android can auto-retrieve the SMS
  (`AutoVerified`) and skip the manual OTP entry entirely, jumping straight to
  `signInWithPhone`.
- The verification id lives in this ViewModel's state and is passed into the service on every
  call, so the change-phone flow, which has its own id, can never overwrite it.
- The success branch of `signInWithPhone` is the session bootstrap: presence online, sync
  restart, recent-history backfill, FCM token sync, then `AuthEffect.AuthSucceeded`. See
  [[Presence and Typing]] and [[Notifications]].
- Phone-verification errors go through `toPhoneAuthUiText()` (`presentation/base/PhoneAuthErrors.kt`),
  never a raw thrown exception into the UI. See `UiText` in [[Presentation Layer (MVI)]].

---

## 3. Use cases

Each is a thin, single-purpose wrapper over `IAuthRepository`.

`LoginWithPhoneNumberUseCase.kt`:

```kotlin
class LoginWithPhoneNumberUseCase @Inject constructor(
    private val authRepository: IAuthRepository,
) {
    suspend operator fun invoke(
        credential: PhoneAuthCredential,
        username: String? = null,
        isRegistration: Boolean = false,
    ): Resource<User> = authRepository.loginWithPhone(credential, username, isRegistration)
}
```

`LinkPhoneUseCase`, `ObserveAuthStateUseCase` and `GetCurrentUserUseCase` are one-line
delegations to `linkPhoneToAccount`, `observeAuthState` and `getCurrentUser`.
`ObserveSessionStateUseCase` (the session gate) is shown in [[Navigation]].

`LogoutUseCase.kt` - the one with real orchestration:

```kotlin
class LogoutUseCase @Inject constructor(
    private val authRepository: IAuthRepository,
    private val userRepository: IUserRepository,
    private val localRepository: ILocalRepository,
    private val sessionCleaner: ISessionCleaner,
) {

    suspend operator fun invoke() {
        runCatching { userRepository.updateUserStatus(isOnline = false) }
        val uid = authRepository.getCurrentUser()?.id
        sessionCleaner.clearOnLogout(uid)
        authRepository.logout()
        localRepository.resetDB()
    }
}
```

Logout is ordered deliberately: mark offline while still authenticated (a failure there does not
stop the logout), clear the device session (`ISessionCleaner` / `AndroidSessionCleaner`: voice
playback, recording and drafts, FCM token, notifications, media caches), sign out of Firebase,
then wipe the local ObjectBox cache (`resetDB`) exactly once. Doing the offline write first means
the presence update still has a valid auth token.

---

## 4. AuthRepositoryImpl

`data/repository/AuthRepositoryImpl.kt` - the login/registration rules live here (logging
trimmed):

```kotlin
class AuthRepositoryImpl @Inject constructor(
    private val auth: FirebaseAuthService,
    private val firestore: FirestoreService,
    private val e2eeBootstrap: E2eeBootstrap,
    private val authFlowGate: IAuthFlowGate,
    private val accountDeletion: AccountDeletionService,
) : IAuthRepository {

    override suspend fun loginWithPhone(
        credential: PhoneAuthCredential,
        username: String?,
        isRegistration: Boolean,
    ): Resource<User> = authFlowGate.withAuthFlow {
        performLoginWithPhone(credential, username, isRegistration)
    }

    private suspend fun performLoginWithPhone(
        credential: PhoneAuthCredential,
        username: String?,
        isRegistration: Boolean,
    ): Resource<User> {
        return try {
            val firebaseUser = auth.signInWithPhone(credential).getOrThrow()
            val existing = firestore.getUserProfile(firebaseUser.uid).getOrNull()

            if (existing?.deleted == true) {
                auth.logout()
                return Resource.Error("This account has been deleted.")
            }

            if (existing != null) {
                if (isRegistration) {
                    auth.logout()
                    return Resource.Failure(AuthAccountException.AlreadyRegistered())
                }
                if (existing.usernameLower.isNullOrBlank() && !existing.username.isNullOrBlank()) {
                    runCatching {
                        firestore.updateUserProfile(
                            existing.id,
                            mapOf("usernameLower" to existing.username!!.lowercase()),
                        )
                    }
                }
                e2eeBootstrap.refresh()
                return Resource.Success(existing)
            }

            if (!isRegistration) {
                auth.deleteAccount()
                    .onFailure {
                        Log.e(TAG, "loginWithPhone: failed to delete orphan auth user", it)
                        auth.logout()
                    }
                return Resource.Failure(AuthAccountException.NotRegistered())
            }

            val resolvedUsername = username?.trim()?.takeIf { it.isNotBlank() }
                ?: firebaseUser.displayName?.takeIf { it.isNotBlank() && !it.startsWith("+") }
            val user = User(
                id = firebaseUser.uid,
                username = resolvedUsername,
                usernameLower = resolvedUsername?.lowercase(),
                email = null,
                phoneNumber = null,
                avatarUrl = firebaseUser.photoUrl?.toString(),
                lastSeen = System.currentTimeMillis(),
                isOnline = true,
                fcmToken = null,
            ).also {
                firestore.createUserProfile(it).getOrThrow()
                e2eeBootstrap.refresh()
            }
            Resource.Success(user)
        } catch (e: Exception) {
            Resource.Error("${e.message}, Phone sign-in failed")
        }
    }

    override suspend fun deleteAccount(): Result<Unit> = accountDeletion.deleteAccount()
}
```

`linkPhoneToAccount`, `logout`, `getCurrentUser`, `observeAuthState` and `updateUserProfile` are
thin mappings over `FirebaseAuthService` (Firebase types become `User` / `DomainUser`).

The important logic:

- `authFlowGate.withAuthFlow` marks the login as in flight, so the session gate stays at
  `Resolving` and does not judge a half-created account.
- Firebase phone sign-in always succeeds first (it just proves phone ownership); the
  registration/login decision is then made against the Firestore profile:
  - profile marked `deleted` -> reject and sign back out;
  - profile exists + `isRegistration` -> `Resource.Failure(AuthAccountException.AlreadyRegistered())`
    and sign back out so no half-logged-in state remains (the ViewModel turns it into a message and
    switches to login mode);
  - profile exists + login -> success, plus a one-time backfill of `usernameLower` (used for
    prefix search) for older profiles;
  - no profile + login -> `Resource.Failure(AuthAccountException.NotRegistered())` and
    `deleteAccount()` the just-created orphan Firebase auth user so it does not linger (signing out
    if that delete fails);
  - no profile + `isRegistration` -> create the Firestore profile.
- `e2eeBootstrap.refresh()` runs on every successful sign-in path, which makes `E2eeBootstrap` resolve
  the account's key state (create and publish a key for a new account, or wait for a recovery code on
  an account that already has keys). See [[End-to-End Encryption]].
- `deleteAccount` delegates to `AccountDeletionService`; see [[Settings]].

---

## 5. FirebaseAuthService

`data/remote/auth/FirebaseAuthService.kt` - the Firebase SDK wrapper. The interesting parts are
the `suspendCancellableCoroutine` bridges that turn Firebase's callback-based phone verification
into `suspend` functions.

```kotlin
private val resendTokens = mutableMapOf<String, PhoneAuthProvider.ForceResendingToken>()

suspend fun sendVerificationCode(
    phoneNumber: String,
    activity: Activity
): Result<VerificationResult> = suspendCancellableCoroutine { continuation ->
    val callbacks = object : PhoneAuthProvider.OnVerificationStateChangedCallbacks() {
        override fun onVerificationCompleted(credential: PhoneAuthCredential) {
            if (continuation.isActive) {
                continuation.resume(
                    Result.success(VerificationResult.AutoVerified(credential))
                )
            }
        }
        override fun onVerificationFailed(e: FirebaseException) {
            if (continuation.isActive) {
                continuation.resume(Result.failure(e))
            }
        }
        override fun onCodeSent(
            verId: String,
            token: PhoneAuthProvider.ForceResendingToken
        ) {
            resendTokens[verId] = token
            if (continuation.isActive) {
                continuation.resume(
                    Result.success(VerificationResult.CodeSent(verId))
                )
            }
        }
    }
    val options = PhoneAuthOptions.newBuilder(auth)
        .setPhoneNumber(phoneNumber)
        .setTimeout(60L, TimeUnit.SECONDS)
        .setActivity(activity)
        .setCallbacks(callbacks)
        .build()
    PhoneAuthProvider.verifyPhoneNumber(options)
}
```

`suspendCancellableCoroutine` is the right primitive here: `verifyPhoneNumber` reports its
outcome through one of three callbacks, and we want to `resume` the coroutine exactly once with
whichever fires. The `continuation.isActive` guard prevents a double-resume if more than one
callback arrives. The service no longer remembers "the" verification id: it hands the id back in
`CodeSent` and keeps only the resend token, keyed by that id. Neither the id nor the SMS code is
logged.

```kotlin
fun verifyCode(code: String, verificationId: String?): Result<PhoneAuthCredential> {
    return try {
        val verId = verificationId?.takeIf { it.isNotBlank() }
            ?: return Result.failure(Exception("No verification ID found"))

        val credential = PhoneAuthProvider.getCredential(verId, code)
        Result.success(credential)
    } catch (e: Exception) {
        Result.failure(e)
    }
}

suspend fun signInWithPhone(credential: PhoneAuthCredential): Result<FirebaseUser> {
    return try {
        val result = auth.signInWithCredential(credential).await()
        val user = result.user ?: throw Exception("Sign in failed")
        Result.success(user)
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

`verifyCode` builds a `PhoneAuthCredential` from the caller's verification id and the typed code;
`signInWithPhone` exchanges the credential for a session using `kotlinx-coroutines-play-services`
`await()` on the Firebase `Task`.

```kotlin
fun observeAuthState(): Flow<FirebaseUser?> = callbackFlow {
    val listener = FirebaseAuth.AuthStateListener { auth ->
        trySend(auth.currentUser)
    }
    auth.addAuthStateListener(listener)
    awaitClose {
        auth.removeAuthStateListener(listener)
    }
}

suspend fun logout() {
    auth.signOut()
    resendTokens.clear()
}
```

`observeAuthState` is another `callbackFlow` bridge (see the family of these in
[[Async and Reactivity]]): it emits the current user on every auth-state change and detaches the
listener on `awaitClose`. `logout` signs out and drops the resend tokens; it no longer resets the
local DB (that happens once, in `LogoutUseCase`).

The service also exposes `linkPhone`, `isPhoneLinked`, `updateProfile`, `deleteAccount`,
`reauthenticateWithPhone`, `updatePhoneNumber`, and `resendVerificationCode(phoneNumber,
activity, verificationId)` (a second `suspendCancellableCoroutine` bridge identical in shape to
`sendVerificationCode`, seeded with `resendTokens[verificationId]`). These back the change-phone
and account-management flows in [[Settings]].

---

## 6. UI (not embedded)

The Compose UI is `presentation/screens/auth/` (`AuthScreen.kt`, `AuthScreenContent.kt`) plus `presentation/components/auth/` (`AuthComponents.kt`,
`AuthModeToggle.kt` for the login/register switch, `CountryCodePicker.kt` + `model/Countries.kt`
for the dialling-code picker, `ChangePhoneComponents.kt`). `AuthScreen` resolves the Activity with
`LocalActivity.current`, collects `AuthViewModel.state` with `collectAsStateWithLifecycle`,
renders either the phone-entry or the OTP-entry UI based on `codeSent`, observes `AuthEffect` via
`ObserveAsEvents`, and dispatches the intents above. The content composable is a pure function of
`AuthUiState`; see the Root/Screen split in [[Presentation Layer (MVI)]].

---

## Related

[[Authentication]] - [[Presentation Layer (MVI)]] - [[Domain Layer]] - [[Data Layer]] -
[[Navigation]] - [[Presence and Typing]] - [[Notifications]] - [[End-to-End Encryption]]
