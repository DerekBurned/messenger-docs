# Authentication - Basic Example

The inherited pattern behind [[Authentication]], in its plain textbook form - not the real
`AuthViewModel`/`FirebaseAuthService`, but the minimal shape they are built on. Read this first,
then [[Authentication - Code]] to see how the app specialises it.

(Teaching comments only; the real source has none - that is a project rule.)

---

## What is inherited

Phone-number sign-in is Firebase Auth's own `PhoneAuthProvider` flow: ask Firebase to send an
SMS, get a `verificationId` back through a callback, combine it with the typed code into a
`PhoneAuthCredential`, and sign in with that credential. On top of it sits one app pattern:
**the whole app is switched by observed auth state**, not by an imperative navigate-on-login.

## The basic example (canonical Firebase phone auth)

```kotlin
// 1) Ask Firebase to send the SMS. The result comes back through callbacks, not a return value.
val options = PhoneAuthOptions.newBuilder(FirebaseAuth.getInstance())
    .setPhoneNumber("+15555550123")
    .setTimeout(60L, TimeUnit.SECONDS)
    .setActivity(activity)
    .setCallbacks(object : PhoneAuthProvider.OnVerificationStateChangedCallbacks() {
        override fun onCodeSent(id: String, token: PhoneAuthProvider.ForceResendingToken) {
            verificationId = id                 // remember it; the UI now shows the code field
        }
        override fun onVerificationCompleted(credential: PhoneAuthCredential) {
            signIn(credential)                  // SMS auto-retrieved: skip typing the code
        }
        override fun onVerificationFailed(e: FirebaseException) { /* show error */ }
    })
    .build()
PhoneAuthProvider.verifyPhoneNumber(options)

// 2) User typed the 6 digits -> build a credential and sign in.
fun submitCode(code: String) {
    signIn(PhoneAuthProvider.getCredential(verificationId, code))
}

fun signIn(credential: PhoneAuthCredential) {
    FirebaseAuth.getInstance().signInWithCredential(credential)
        .addOnSuccessListener { /* signed in */ }
}
```

And the inherited "auth state is the top-level switch" idea, at its simplest:

```kotlin
// A cold stream of "are we signed in?", read once at the top of the UI tree.
val isAuthenticated: Flow<Boolean> = callbackFlow {
    val listener = FirebaseAuth.AuthStateListener { trySend(it.currentUser != null) }
    FirebaseAuth.getInstance().addAuthStateListener(listener)
    awaitClose { FirebaseAuth.getInstance().removeAuthStateListener(listener) }
}

// The router just reacts; nobody calls navigate("home") on login.
when (isAuthenticated.collectAsState(false).value) {
    true  -> MainScreens()
    false -> LoginScreen()
}
```

## How the real code specialises it

- The callback flow is wrapped into a suspend `VerificationResult` seam in `FirebaseAuthService`
  and driven from `AuthViewModel.sendVerificationCode` (`presentation/viewmodel/AuthViewModel.kt:112`)
  and `signInWithPhone` (`AuthViewModel.kt:212`), so the UI only ever sees MVI state/effects.
- The `verificationId` the textbook keeps in a variable lives in the ViewModel's `AuthUiState`
  and is passed back to the service on each call, so two flows (sign-in and change-phone) can
  never overwrite each other's id.
- On success the app also bootstraps the session: presence online, sync restart, recent-history
  backfill and FCM-token sync inside the success branch (`AuthViewModel.kt:212`).
- The "auth state switches the app" idea is real in `AppNavigation`
  (`presentation/navigation/graph/AppNavigation.kt:23`), with one refinement: it switches on a
  resolved `SessionState` (`Resolving` / `SignedOut` / `Active`) rather than the bare
  `currentUser != null`, so the main UI mounts only once the user's Firestore profile is
  confirmed. Login and logout still flip the whole tree with no navigate call.

## Related

[[Authentication]] - [[Authentication - Code]] - [[Authentication - Explained Simply]] -
[[Reading the Code - Simple Patterns]] - [[Navigation]] - [[Home]]
