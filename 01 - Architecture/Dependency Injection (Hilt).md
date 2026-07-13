# Dependency Injection (Hilt)

## What and version

Hilt `2.59.2` (`gradle/libs.versions.toml:19`), processed via KSP
(`app/build.gradle.kts:138`, `ksp(libs.hilt.compiler)`). The Hilt Gradle plugin is applied at
`app/build.gradle.kts:9`. `androidx.hilt:hilt-navigation-compose` `1.3.0` supplies
`hiltViewModel()` for Compose, and `androidx.hilt:hilt-work` + `androidx.hilt:hilt-compiler`
supply `@HiltWorker` support.

## Entry points into the graph

- `MessengerApplication` is `@HiltAndroidApp` (`MessengerApplication.kt:34`) - the root that
  triggers component generation and holds the `SingletonComponent`.
- Activities are `@AndroidEntryPoint` (`MainActivity`, `IncomingCallActivity`,
  `BubbleActivity`).
- `CallForegroundService` is `@AndroidEntryPoint` (`data/remote/call/CallForegroundService.kt:45`),
  so it can `@Inject` `ICallService`, `ICallSignalingService`, `MissedCallRecorder`,
  `AgoraTokenProvider`.
- ViewModels are `@HiltViewModel` with `@Inject constructor` (`ChatViewModel`,
  `AuthViewModel`, `ConversationsViewModel`, `CallViewModel`); the UI obtains them with
  `hiltViewModel()` (`presentation/navigation/navigation.kt:81`).

## The modules (`di/`, 10 files)

All installed in `SingletonComponent`, so bindings are process-wide singletons.

- `AppModule` (`di/AppModule.kt`) - `@Provides` the application `Context` and the IO
  `CoroutineDispatcher` (`Dispatchers.IO`).
- `ObjectBoxModule` (`di/ObjectBoxModule.kt`) - `@Provides @Singleton` the `BoxStore` (with the
  debug schema-recovery fallback, `:29`) and one `Box<Obx*>` per entity (`:42` onward).
- `RepositoryModule` (`di/RepositoryModule.kt`) - an `abstract class` using `@Binds @Singleton`
  to bind each `*Impl` to its `I*Repository` interface (`AuthRepositoryImpl -> IAuthRepository`,
  etc., plus `LocalRepositoryImpl -> ILocalRepository`).
- `CallModule` (`di/CallModule.kt`) - `@Binds` `AgoraCallService -> ICallService` and
  `FirebaseCallSignalingService -> ICallSignalingService`.
- `CryptoModule` (`di/CryptoModule.kt`) - `@Provides` `KeystoreKeyWrapper`, the
  `IdentityKeyStore` rooted at `filesDir/e2ee`, and the `ObxParticipantResolver`.
- `FirebaseModule` - `@Provides` the Firebase singletons (`FirebaseAuth`, `FirebaseFirestore`,
  `FirebaseDatabase`, `FirebaseStorage`, `FirebaseMessaging`).
- `RealtimeModule` - presence/typing service bindings.
- `NetworkModule` - the network observer binding.
- `UseCaseModule` - use-case provisioning (most are constructor-injected).
- `WorkManagerModule` - WorkManager-related provisions for the sync worker.

## `@Binds` vs `@Provides` - the rule used here

- `@Binds` (in `abstract class` modules) is used when the thing being provided is an interface
  whose implementation is a Hilt-constructable class - no factory code needed, just "when
  someone asks for `IMessageRepository`, give them the injected `MessageRepositoryImpl`". This
  is `RepositoryModule` and `CallModule`.
- `@Provides` (in `object` modules) is used when construction needs logic or a type Hilt cannot
  build itself: the `BoxStore` (needs a builder and error handling), the individual `Box<T>`
  (built from the store), Firebase singletons (obtained from static getters), the
  `IdentityKeyStore` (needs a `File` path). This is `ObjectBoxModule`, `AppModule`,
  `CryptoModule`, `FirebaseModule`.

## Scoping

Everything is `@Singleton` in the `SingletonComponent`, which matches the app's design: the
`BoxStore`, repositories, Firebase services, `AgoraCallService`, and the crypto codec are all
long-lived and shared. ViewModels are scoped to their navigation entry by `@HiltViewModel` +
the Navigation 3 ViewModel store decorator (see [[Navigation]]), not by a Hilt scope.

## Manual EntryPoint for framework-instantiated classes

`MyFirebaseMessagingService` cannot use `@AndroidEntryPoint` cleanly because the OS constructs
`FirebaseMessagingService`. Instead it declares a Hilt `@EntryPoint` interface and pulls
dependencies out of the graph on demand
(`data/remote/firebase/MyFirebaseMessagingService.kt:23`):

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface FcmEntryPoint {
    fun firestoreService(): FirestoreService
    fun telecomCallManager(): TelecomCallManager
    fun messageRepository(): IMessageRepository
    fun e2eeCodec(): E2eeMessageCodec
}
// ...
val entryPoint = EntryPointAccessors.fromApplication(applicationContext, FcmEntryPoint::class.java)
```

This is the standard Hilt escape hatch for a component you do not construct. See
[[Notifications]].

## Related

[[Architecture Overview]] - [[Data Layer]] - [[Domain Layer]] - [[Navigation]] -
[[Notifications]] - [[Tech Stack and Versions]]
