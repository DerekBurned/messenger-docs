# Authentication - Code

Companion to [[Authentication]]. This note gathers the actual source for the authentication
feature and explains it block by block. Conceptual overview and rationale live in the main note;
here the focus is the code itself. All snippets are copied verbatim from the repo.

Files covered:
- MVI contract: `presentation/state/AuthUiState.kt`, `presentation/intent/AuthIntent.kt`,
  `presentation/effect/AuthEffect.kt`
- `presentation/viewmodel/AuthViewModel.kt`
- Use cases: `domain/usecase/auth/{LoginWithPhoneNumberUseCase, LinkPhoneUseCase, LogoutUseCase,
  ObserveAuthStateUseCase, GetCurrentUserUseCase}.kt`
- `data/repository/AuthRepositoryImpl.kt`
- `data/remote/auth/FirebaseAuthService.kt`
- UI (described, not embedded): `presentation/screens/AuthScreen.kt` and
  `presentation/components/auth/*`

---

## 1. MVI contract

`presentation/state/AuthUiState.kt` - the entire screen state:

```kotlin
data class AuthUiState(
    val isLoading: Boolean = false,
    val currentUser: User? = null,
    val error: UiText? = null,
    val isAuthenticated: Boolean = false,
    val codeSent: Boolean = false,
) : UiState
```

Five fields carry the whole flow: `isLoading` (spinner), `codeSent` (switches the UI from the
phone field to the OTP field), `isAuthenticated` (the top-level nav switch), `error` as a
`UiText`, and `currentUser` after success.

`presentation/intent/AuthIntent.kt` - every action the screen can send:

```kotlin
sealed interface AuthIntent : UiIntent {
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
SMS auto-retrieval (`PhoneAuthOptions.setActivity`). `isRegister` distinguishes login from
registration so the repository can enforce the "already registered / not registered" rules.

`presentation/effect/AuthEffect.kt` - one-shot signals up to the UI:

```kotlin
sealed interface AuthEffect : UiEffect {
    data object AuthSucceeded : AuthEffect
    data object PhoneLinked : AuthEffect
}
```

---

## 2. AuthViewModel

`presentation/viewmodel/AuthViewModel.kt` in full:

```kotlin
@HiltViewModel
class AuthViewModel @Inject constructor(
    private val loginWithPhoneNumberUseCase: LoginWithPhoneNumberUseCase,
    private val linkPhoneUseCase: LinkPhoneUseCase,
    private val logoutUseCase: LogoutUseCase,
    private val observeAuthStateUseCase: ObserveAuthStateUseCase,
    private val firebaseAuthService: FirebaseAuthService,
    private val presenceManager: PresenceManager,
    private val firebaseMessagingManager: FirebaseMessagingManager,
    private val firestoreService: FirestoreService,
) : MviViewModel<AuthUiState, AuthIntent, AuthEffect>(
    initialState = AuthUiState(isAuthenticated = firebaseAuthService.isAuthenticated()),
) {

    private var pendingUsername: String? = null
    private var pendingIsRegister: Boolean = false

    init {
        Log.d(TAG, "init: isAuthenticated=${firebaseAuthService.isAuthenticated()}")
        observeAuthStatus()
    }

    override fun handleIntent(intent: AuthIntent) {
        Log.d(TAG, "handleIntent: ${intent::class.simpleName}")
        when (intent) {
            is AuthIntent.SendVerificationCode ->
                sendVerificationCode(intent.activity, intent.phoneNumber, intent.username, intent.isRegister)
            is AuthIntent.ResendCode -> resendCode(intent.activity, intent.phoneNumber)
            is AuthIntent.VerifyOtpAndLogin -> verifyOtpAndLogin(intent.otpCode)
            is AuthIntent.VerifyOtpAndLink -> verifyOtpAndLink(intent.otpCode)
            AuthIntent.Logout -> logout()
            AuthIntent.ClearError -> setState { copy(error = null) }
            AuthIntent.EditPhoneNumber -> setState { copy(codeSent = false, error = null) }
        }
    }

    private fun observeAuthStatus() {
        viewModelScope.launch {
            observeAuthStateUseCase().collectLatest { user ->
                setState { copy(isAuthenticated = user != null) }
            }
        }
    }

    private fun logout() {
        viewModelScope.launch {
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
                    is VerificationResult.CodeSent ->
                        setState { copy(isLoading = false, codeSent = true) }
                    is VerificationResult.AutoVerified ->
                        signInWithPhone(verifyResult.credential)
                }
            } else {
                val errorMsg: UiText = result.exceptionOrNull()?.message?.toUiText()
                    ?: UiText.StringResource(R.string.auth_error_verification_failed)
                setState { copy(isLoading = false, error = errorMsg) }
            }
        }
    }

    private fun resendCode(activity: Activity, phoneNumber: String) {
        setState { copy(isLoading = true, error = null) }
        viewModelScope.launch {
            val result = firebaseAuthService.resendVerificationCode(phoneNumber, activity)
            if (result.isSuccess) {
                when (val verifyResult = result.getOrThrow()) {
                    is VerificationResult.CodeSent -> setState { copy(isLoading = false, codeSent = true) }
                    is VerificationResult.AutoVerified -> signInWithPhone(verifyResult.credential)
                }
            } else {
                val errorMsg: UiText = result.exceptionOrNull()?.message?.toUiText()
                    ?: UiText.StringResource(R.string.auth_error_verification_failed)
                setState { copy(isLoading = false, error = errorMsg) }
            }
        }
    }

    private fun verifyOtpAndLogin(otpCode: String) {
        viewModelScope.launch {
            val credentialResult = firebaseAuthService.verifyCode(otpCode)
            if (credentialResult.isSuccess) {
                signInWithPhone(credentialResult.getOrThrow())
            } else {
                setState { copy(error = UiText.StringResource(R.string.auth_error_invalid_code)) }
            }
        }
    }

    private fun verifyOtpAndLink(otpCode: String) {
        viewModelScope.launch {
            val credentialResult = firebaseAuthService.verifyCode(otpCode)
            if (credentialResult.isSuccess) {
                linkPhone(credentialResult.getOrThrow())
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
            when (result) {
                is Resource.Success -> {
                    presenceManager.goOnline(viewModelScope)
                    syncFcmTokenForCurrentUser()
                    setState { copy(isLoading = false, currentUser = result.data, error = null) }
                    emitEffect(AuthEffect.AuthSucceeded)
                }
                is Resource.Error -> setState { copy(isLoading = false, error = result.message.toUiText()) }
                is Resource.Failure -> setState { copy(isLoading = false, error = result.exception.message?.toUiText()) }
                is Resource.Loading -> {}
            }
        }
    }

    private suspend fun syncFcmTokenForCurrentUser() {
        val uid = firebaseAuthService.getCurrentUserId() ?: return
        firebaseMessagingManager.getFcmToken()
            .onSuccess { token -> firestoreService.updateFcmToken(uid, token) }
            .onFailure { e -> Log.w(TAG, "FCM token sync skipped: ${e.message}", e) }
    }

    private fun linkPhone(credential: PhoneAuthCredential) {
        viewModelScope.launch {
            linkPhoneUseCase(credential).collect { resource ->
                when (resource) {
                    is Resource.Loading -> setState { copy(isLoading = true, error = null) }
                    is Resource.Success -> {
                        setState { copy(isLoading = false, error = null) }
                        emitEffect(AuthEffect.PhoneLinked)
                    }
                    is Resource.Error -> setState { copy(isLoading = false, error = resource.message.toUiText()) }
                    is Resource.Failure -> setState { copy(isLoading = false, error = resource.exception.message?.toUiText()) }
                }
            }
        }
    }

    private companion object { const val TAG = "AUTHFLOW_VM" }
}
```

Explanation of the moving parts (verbose logging lines trimmed above; they exist in the source):

- Initial state seeds `isAuthenticated` synchronously from `firebaseAuthService.isAuthenticated()`
  so a returning, already-signed-in user lands on the main app on the first frame without a
  flash of the auth screen.
- `pendingUsername`/`pendingIsRegister` are held between the "send code" step and the eventual
  `signInWithPhone`, because the code arrives asynchronously (or auto-verifies) and the username
  is only needed at sign-in time.
- `observeAuthStatus` uses `collectLatest` on the auth-state flow: if Firebase emits a new
  auth state, any in-flight handling of the previous one is cancelled. This is what makes
  logout/login flip the whole app (see `AppNavigation` in [[Navigation]]).
- Both `CodeSent` and `AutoVerified` are handled: Android can auto-retrieve the SMS
  (`AutoVerified`) and skip the manual OTP entry entirely, jumping straight to
  `signInWithPhone`.
- The success branch of `signInWithPhone` is the session bootstrap: `presenceManager.goOnline`,
  FCM token sync, then `emitEffect(AuthEffect.AuthSucceeded)`. See [[Presence and Typing]] and
  [[Notifications]].
- Errors are always turned into a `UiText` (either the exception message or a string-resource
  fallback), never a raw thrown exception into the UI. See `UiText` in
  [[Presentation Layer (MVI)]].

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

`LinkPhoneUseCase.kt`:

```kotlin
class LinkPhoneUseCase @Inject constructor(
    private val auth: IAuthRepository
) {
    suspend operator fun invoke(credential: PhoneAuthCredential): Flow<Resource<Unit>> =
        auth.linkPhoneToAccount(credential)
}
```

`ObserveAuthStateUseCase.kt`:

```kotlin
class ObserveAuthStateUseCase @Inject constructor(
    private val authRepository: IAuthRepository
) {
    operator fun invoke(): Flow<DomainUser?> = authRepository.observeAuthState()
}
```

`GetCurrentUserUseCase.kt`:

```kotlin
class GetCurrentUserUseCase @Inject constructor(
    private val authRepository: IAuthRepository
) {
    operator fun invoke(): User? = authRepository.getCurrentUser()
}
```

`LogoutUseCase.kt` - the one with real orchestration:

```kotlin
class LogoutUseCase @Inject constructor(
    private val authRepository: IAuthRepository,
    private val userRepository: IUserRepository,
    private val localRepository: ILocalRepository,
    private val sessionCleaner: ISessionCleaner,
) {
    suspend operator fun invoke() {
        userRepository.updateUserStatus(isOnline = false)
        val uid = authRepository.getCurrentUser()?.id
        sessionCleaner.clearOnLogout(uid)
        authRepository.logout()
        localRepository.resetDB()
    }
}
```

Logout is ordered deliberately: mark offline while still authenticated, clear device session
(`ISessionCleaner` / `AndroidSessionCleaner`), sign out of Firebase, then wipe the local
ObjectBox cache (`resetDB`). Doing the offline write first means the presence update still has a
valid auth token.

---

## 4. AuthRepositoryImpl

`data/repository/AuthRepositoryImpl.kt` - the login/registration rules live here:

```kotlin
class AuthRepositoryImpl @Inject constructor(
    private val auth: FirebaseAuthService,
    private val firestore: FirestoreService,
    private val e2eeBootstrap: E2eeBootstrap,
) : IAuthRepository {

    override suspend fun loginWithPhone(
        credential: PhoneAuthCredential,
        username: String?,
        isRegistration: Boolean,
    ): Resource<User> {
        return try {
            val firebaseUser = auth.signInWithPhone(credential).getOrThrow()
            val existing = firestore.getUserProfile(firebaseUser.uid).getOrNull()

            if (existing != null) {
                if (isRegistration) {
                    auth.logout()
                    return Resource.Error("An account with this number already exists. Please log in instead.")
                }
                if (existing.usernameLower.isNullOrBlank() && !existing.username.isNullOrBlank()) {
                    runCatching {
                        firestore.updateUserProfile(existing.id,
                            mapOf("usernameLower" to existing.username!!.lowercase()))
                    }
                }
                e2eeBootstrap.publishKeys()
                return Resource.Success(existing)
            }

            if (!isRegistration) {
                auth.deleteAccount()
                    .onFailure { Log.e(TAG, "failed to delete orphan auth user", it) }
                return Resource.Error("User does not exist. Please register first.")
            }

            val resolvedUsername = username?.trim()?.takeIf { it.isNotBlank() }
                ?: firebaseUser.displayName?.takeIf { it.isNotBlank() && !it.startsWith("+") }
            val user = User(
                id = firebaseUser.uid,
                username = resolvedUsername,
                usernameLower = resolvedUsername?.lowercase(),
                email = null, phoneNumber = null,
                avatarUrl = firebaseUser.photoUrl?.toString(),
                lastSeen = System.currentTimeMillis(),
                isOnline = true, fcmToken = null,
            ).also {
                firestore.createUserProfile(it).getOrThrow()
                e2eeBootstrap.publishKeys()
            }
            Resource.Success(user)
        } catch (e: Exception) {
            Resource.Error("${e.message}, Phone sign-in failed")
        }
    }

    override suspend fun linkPhoneToAccount(credential: PhoneAuthCredential): Flow<Resource<Unit>> = flow {
        try {
            emit(Resource.Loading)
            auth.linkPhone(credential).getOrThrow()
            emit(Resource.Success(Unit))
        } catch (e: Exception) {
            emit(Resource.Error("${e.message}, Internal error occurred"))
        }
    }

    override suspend fun logout() = auth.logout()

    override fun getCurrentUser(): User? {
        val firebaseUser = auth.getCurrentUser()
        return firebaseUser?.let {
            User(id = it.uid, username = firebaseUser.displayName, email = null,
                phoneNumber = null, avatarUrl = firebaseUser.photoUrl?.toString(),
                lastSeen = System.currentTimeMillis(), isOnline = false, fcmToken = null)
        }
    }

    override fun observeAuthState(): Flow<DomainUser?> =
        auth.observeAuthState().map { firebaseUser ->
            firebaseUser?.let {
                DomainUser(id = it.uid, displayName = it.displayName,
                    phoneNumber = it.phoneNumber, photoUrl = it.photoUrl?.toString())
            }
        }

    override suspend fun updateUserProfile(username: String, avatarUrl: String?): Result<Unit> =
        auth.updateProfile(username, avatarUrl)

    private companion object { const val TAG = "AUTHFLOW_REPO" }
}
```

The important logic:

- Firebase phone sign-in always succeeds first (it just proves phone ownership); the
  registration/login decision is then made against the Firestore profile:
  - profile exists + `isRegistration` -> reject ("already exists"), and sign back out so no
    half-logged-in state remains;
  - profile exists + login -> success, plus a one-time backfill of `usernameLower` (used for
    prefix search) for older profiles;
  - no profile + login -> reject and `deleteAccount()` the just-created orphan Firebase auth
    user so it does not linger;
  - no profile + `isRegistration` -> create the Firestore profile.
- `e2eeBootstrap.publishKeys()` runs on every successful sign-in path, so the device's public
  E2EE key is published as soon as the user is in. See [[End-to-End Encryption]].
- `observeAuthState` maps the raw `FirebaseUser` flow to the domain `DomainUser`, keeping
  Firebase types out of the domain.

---

## 5. FirebaseAuthService

`data/remote/auth/FirebaseAuthService.kt` - the Firebase SDK wrapper. The interesting parts are
the two `suspendCancellableCoroutine` bridges that turn Firebase's callback-based phone
verification into `suspend` functions.

```kotlin
suspend fun sendVerificationCode(
    phoneNumber: String,
    activity: Activity
): Result<VerificationResult> = suspendCancellableCoroutine { continuation ->
    val callbacks = object : PhoneAuthProvider.OnVerificationStateChangedCallbacks() {
        override fun onVerificationCompleted(credential: PhoneAuthCredential) {
            if (continuation.isActive) continuation.resume(Result.success(VerificationResult.AutoVerified(credential)))
        }
        override fun onVerificationFailed(e: FirebaseException) {
            if (continuation.isActive) continuation.resume(Result.failure(e))
        }
        override fun onCodeSent(verId: String, token: PhoneAuthProvider.ForceResendingToken) {
            verificationId = verId
            resendToken = token
            if (continuation.isActive) continuation.resume(Result.success(VerificationResult.CodeSent(verId)))
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
callback arrives. `verificationId` and `resendToken` are cached on the service so `verifyCode`
and `resendVerificationCode` can use them later.

```kotlin
suspend fun verifyCode(code: String): Result<PhoneAuthCredential> {
    return try {
        val verId = verificationId ?: return Result.failure(Exception("No verification ID found"))
        Result.success(PhoneAuthProvider.getCredential(verId, code))
    } catch (e: Exception) { Result.failure(e) }
}

suspend fun signInWithPhone(credential: PhoneAuthCredential): Result<FirebaseUser> {
    return try {
        val result = auth.signInWithCredential(credential).await()
        Result.success(result.user ?: throw Exception("Sign in failed"))
    } catch (e: Exception) { Result.failure(e) }
}
```

`verifyCode` just builds a `PhoneAuthCredential` from the cached verification id + the typed
code; `signInWithPhone` exchanges the credential for a session using
`kotlinx-coroutines-play-services` `await()` on the Firebase `Task`.

```kotlin
fun observeAuthState(): Flow<FirebaseUser?> = callbackFlow {
    val listener = FirebaseAuth.AuthStateListener { auth -> trySend(auth.currentUser) }
    auth.addAuthStateListener(listener)
    awaitClose { auth.removeAuthStateListener(listener) }
}

suspend fun logout() {
    repository.resetDB()
    auth.signOut()
    verificationId = null
    resendToken = null
}
```

`observeAuthState` is another `callbackFlow` bridge (see the family of these in
[[Async and Reactivity]]): it emits the current user on every auth-state change and detaches the
listener on `awaitClose`. `logout` resets local DB, signs out, and clears the cached verification
tokens.

The service also exposes `linkPhone`, `isPhoneLinked`, `updateProfile`, `deleteAccount`,
`reauthenticateWithPhone`, `updatePhoneNumber`, and `resendVerificationCode` (a second
`suspendCancellableCoroutine` bridge identical in shape to `sendVerificationCode` but seeded with
the cached `resendToken`). These back the change-phone and account-management flows in
[[Settings]].

---

## 6. UI (not embedded)

The Compose UI is `presentation/screens/AuthScreen.kt` plus `presentation/components/auth/`
(`AuthComponents.kt`, `AuthModeToggle.kt` for login/register switch, `CountryCodePicker.kt` +
`Countries.kt` for the dialling-code picker, `ChangePhoneComponents.kt`). The screen collects
`AuthViewModel.state` with `collectAsStateWithLifecycle`, renders either the phone-entry or the
OTP-entry UI based on `codeSent`, observes `AuthEffect` via `ObserveAsEvents`, and dispatches the
intents above. It is a pure function of `AuthUiState`; see the Root/Screen split in
[[Presentation Layer (MVI)]].

---

## Related

[[Authentication]] - [[Presentation Layer (MVI)]] - [[Domain Layer]] - [[Data Layer]] -
[[Presence and Typing]] - [[Notifications]] - [[End-to-End Encryption]]
