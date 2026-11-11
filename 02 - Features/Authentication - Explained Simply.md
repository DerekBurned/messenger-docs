# Authentication - Explained Simply

The plain-language version of [[Authentication]]. How logging in actually works, step by step.

(The comments in the code are teaching comments for this doc only. The real source has none.)

---

## The big idea in one sentence

You prove you own a phone number by typing a code that gets texted to you; once proven, the whole
app switches from the login screen to the real app - automatically.

Analogy: it is like getting into a building with a one-time code. The front desk texts you a
number, you type it in, the door unlocks. You never picked a password.

---

## Step by step: what happens to the data

Alice opens the app and wants to register with phone `+123` and username `alice`.

1. Alice types her number and taps send. The screen fires an "intent" (a described action):
   `AuthIntent.SendVerificationCode(...)`. The ViewModel receives it in `handleIntent`
   (`presentation/screens/auth/AuthViewModel.kt:73`).

2. The app asks Firebase to text a code: `firebaseAuthService.sendVerificationCode(...)`
   (`data/remote/auth/FirebaseAuthService.kt:38`). The screen shows a spinner
   (`isLoading = true`) and then an OTP box (`codeSent = true`). Firebase also hands back a
   ticket number for this SMS (`verificationId`), which the screen's state keeps.

3. Alice gets the SMS and types the 6-digit code. The screen fires
   `AuthIntent.VerifyOtpAndLogin(code)`.

4. The app turns the code plus the ticket number into proof (`verifyCode(code, verificationId)` ->
   a `PhoneAuthCredential`) and signs in: `signInWithPhone` (`AuthViewModel.kt:254`) calls
   `loginWithPhoneNumberUseCase`.

5. The rules run (`AuthRepositoryImpl.loginWithPhone`,
   `data/repository/AuthRepositoryImpl.kt:36`): if this is a brand-new number and Alice chose
   register, it creates her profile in Firebase. (If the number already existed, the account was
   deleted, or she picked login for an unknown number, it politely refuses.) While this runs the
   app holds a "sign-in in progress" flag so nothing else decides she is signed out half way.

6. On success the app "wakes up" the session: marks her online, restarts syncing, pulls the
   recent history of her chats, uploads her push token, and fires a one-time event
   `AuthEffect.AuthSucceeded`.

7. The whole app flips. Navigation watches one value, the session: "still checking", "signed out"
   or "active". It turns "active" only once her profile is confirmed in Firestore, and at that
   moment the login screen is replaced by the main app - no manual "go to home screen" call.

---

## Picture: the login handshake

```mermaid
sequenceDiagram
    participant U as Alice
    participant App as App
    participant FB as Firebase Auth
    U->>App: enter phone +123
    App->>FB: please text a code
    FB-->>U: SMS code 481920
    U->>App: type 481920
    App->>FB: here is the code, sign me in
    FB-->>App: ok, you are user uid_abc
    Note over App: mark online, restore history, upload push token
    Note over App: profile confirmed -> session Active -> show the real app
```

---

## The code, with plain-English comments

The app-wide switch (`AuthViewModel.kt:111`-`:118`):

```kotlin
private fun observeSession() {
    viewModelScope.launch {
        // Listen to "what is the session?" forever: Resolving, SignedOut or Active.
        observeSessionStateUseCase().collect { session ->
            // Store it. Navigation reacts to it and swaps the whole screen tree.
            setState { copy(session = session) }
            // Signed out for a reason (profile gone, account deleted)? Clean up this device too.
            (session as? SessionState.SignedOut)?.reason?.let { endInvalidSession(it) }
        }
    }
}
```

Turning Firebase's old-style callbacks into a clean `suspend` call
(`data/remote/auth/FirebaseAuthService.kt:38`):

```kotlin
suspend fun sendVerificationCode(phoneNumber, activity) = suspendCancellableCoroutine { cont ->
    // Firebase reports the outcome through THREE possible callbacks.
    // suspendCancellableCoroutine lets us "wait here" and resume once, with whichever fires.
    val callbacks = object : OnVerificationStateChangedCallbacks() {
        override fun onCodeSent(id, token) { cont.resume(CodeSent(id)) }        // normal: code texted
        override fun onVerificationCompleted(cred) { cont.resume(AutoVerified(cred)) } // phone auto-read the SMS
        override fun onVerificationFailed(e) { cont.resume(failure(e)) }        // something went wrong
    }
    PhoneAuthProvider.verifyPhoneNumber(options.setCallbacks(callbacks))
}
```

---

## A worked example

- Alice, new user, phone `+123`, username `alice`, picks Register.
- Firebase texts `481920`. Alice types it.
- `loginWithPhone` sees no existing profile + register -> creates `users/uid_abc` with
  `username = alice`. Success.
- App marks her online, uploads her push token, shows the chat list.
- Bob later logs in with an existing number, picks Login -> profile exists -> straight in.
- If Bob had typed a number with no account and picked Login, the app refuses ("register first")
  and deletes the just-created empty auth record so nothing dangles.

---

## Why, in one line each

- Why a texted code instead of a password? Firebase handles it; the app stores no passwords.
- Why follow a session value instead of navigating? One source of truth - login and logout both
  just change it, and the UI follows; and "active" waits for the profile, so the app never opens
  for an account that does not exist yet.
- Why mark online + upload push token right after login? So the session is fully ready
  (visible to others, reachable by notifications) the instant you are in.

---

## Related

[[Authentication]] - [[Authentication - Code]] - [[Reading the Code - Simple Patterns]] -
[[Presence and Typing - Explained Simply]] - [[Notifications - Explained Simply]] - [[Home]]
