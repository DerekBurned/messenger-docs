# Dependency Injection (Hilt)

## What and version

Hilt `2.59.2` (`gradle/libs.versions.toml:20`), processed via KSP
(`app/build.gradle.kts:163`, `ksp(libs.hilt.compiler)`). The Hilt Gradle plugin is applied at
`app/build.gradle.kts:10` (`alias(libs.plugins.hilt.android)`). `androidx.hilt:hilt-navigation-compose` `1.3.0` supplies
`hiltViewModel()` for Compose, and `androidx.hilt:hilt-work` + `androidx.hilt:hilt-compiler`
supply `@HiltWorker` support.

## Entry points into the graph

- `MessengerApplication` is `@HiltAndroidApp` (`MessengerApplication.kt:36`) - the root that
  triggers component generation and holds the `SingletonComponent`.
- Activities are `@AndroidEntryPoint` (`MainActivity`, `IncomingCallActivity`,
  `BubbleActivity`).
- `CallForegroundService` is `@AndroidEntryPoint` (`data/remote/call/CallForegroundService.kt:49`),
  so it can `@Inject` `ICallService`, `ICallSignalingService`, `MissedCallRecorder`,
  `AgoraTokenProvider`.
- ViewModels are `@HiltViewModel` with `@Inject constructor` (`ChatViewModel`,
  `AuthViewModel`, `ChatsViewModel`, `CallViewModel`, `ClipPlayerViewModel`, ...); the UI
  obtains them with `hiltViewModel()` (`presentation/navigation/graph/AppNavigation.kt:31`).
- Classes with an `@Inject constructor` and `@Singleton` need no module entry at all. The voice
  pipeline is built this way: `VoiceRecorder` (`data/media/VoiceRecorder.kt:99`),
  `VideoNoteRecorder` (`data/media/VideoNoteRecorder.kt:93`), `ClipPlaybackController`
  (`data/media/ClipPlaybackController.kt:102`) and `VoiceDraftStore`
  (`data/media/VoiceDraftStore.kt:33`).

## The modules (`di/`, 12 files)

All installed in `SingletonComponent`, so bindings are process-wide singletons.

- `AppModule` (`di/AppModule.kt`) - `@Provides` the application `Context` and the IO
  `CoroutineDispatcher` (`Dispatchers.IO`).
- `ObjectBoxModule` (`di/ObjectBoxModule.kt`) - `@Provides @Singleton` the `BoxStore` (with the
  debug schema-recovery fallback, `:30`) and one `Box<Obx*>` per entity (`:43` onward).
- `RepositoryModule` (`di/RepositoryModule.kt`) - an `abstract class` using `@Binds @Singleton`
  to bind each `*Impl` to its `I*Repository` interface (`AuthRepositoryImpl -> IAuthRepository`,
  `ClipRepositoryImpl -> IClipRepository` at `:54`, etc., plus
  `LocalRepositoryImpl -> ILocalRepository`).
- `CallModule` (`di/CallModule.kt`) - `@Binds` `AgoraCallService -> ICallService` and
  `FirebaseCallSignalingService -> ICallSignalingService`.
- `CryptoModule` (`di/CryptoModule.kt`) - `@Provides` `KeystoreKeyWrapper`, the
  `IdentityKeyStore` and `RecoveryCodeStore` rooted at `filesDir/e2ee`, the Block Store
  `IdentityBackup`, the `LocalDevice` and the `ObxParticipantResolver`.
- `FirebaseModule` - `@Provides` the Firebase singletons (`FirebaseAuth`, `FirebaseFirestore`,
  `FirebaseDatabase`, `FirebaseStorage`, `FirebaseMessaging`, `FirebaseCrashlytics`,
  `FirebaseRemoteConfig`) and `FirebaseFunctions` pinned to
  `europe-west1` (`di/FirebaseModule.kt:53`), used by `AccountDeletionService` (`deleteAccount`)
  and `AgoraTokenProvider` (`getRtcToken`).
- `RealtimeModule` - presence/typing service bindings, plus `AndroidSessionCleaner -> ISessionCleaner`
  and `InMemoryAuthFlowGate -> IAuthFlowGate` (`di/RealtimeModule.kt:35`) for the session gate.
- `FeatureFlagModule` (`di/FeatureFlagModule.kt`) - `@Binds @Singleton` `RemoteConfigFeatureFlags ->
  FeatureFlags`; see [[Feature Flags and Remote Config]].
- `NetworkModule` - the network observer binding.
- `UseCaseModule` - `@Provides` for use cases whose constructors need wiring (26 functions; for
  example `DeleteMessageUseCase(messageRepository, clipRepository)` and the session/account
  use cases); the rest are constructor-injected.
- `WorkManagerModule` - WorkManager-related provisions for the sync worker.
- `UiEntryPoints` (`di/UiEntryPoints.kt`) - a Hilt `@EntryPoint` giving composables that are not
  ViewModel-backed (`ActiveCallBar`, `CallBarOverlay`, `WallpaperBackground`) and the notification
  code access to `IUserRepository`, `ThemePreferenceStore`, `ICallService` and `FeatureFlags`.

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

`dagger.Lazy` defers construction where a singleton is needed only on a rare path.
`AndroidSessionCleaner` injects `Lazy<ClipPlaybackController>`, `Lazy<VoiceRecorder>`,
`Lazy<VideoNoteRecorder>`, `Lazy<VoiceDraftStore>` and `Lazy<E2eeBootstrap>` and calls `.get()` only
inside `clearOnLogout` (`data/service/AndroidSessionCleaner.kt:35`), so building the logout
use case does not construct the voice singletons, whose `init` blocks start app-wide collectors
(call state, process lifecycle, deleted clips).

## Manual EntryPoint for framework-instantiated classes

`MyFirebaseMessagingService` cannot use `@AndroidEntryPoint` cleanly because the OS constructs
`FirebaseMessagingService`. Instead it declares a Hilt `@EntryPoint` interface and pulls
dependencies out of the graph on demand
(`data/remote/firebase/MyFirebaseMessagingService.kt:27`):

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface FcmEntryPoint {
    fun firestoreService(): FirestoreService
    fun telecomCallManager(): TelecomCallManager
    fun messageRepository(): IMessageRepository
    fun e2eeCodec(): E2eeMessageCodec
    fun deviceRegistry(): DeviceRegistry
    fun incomingCallGate(): IncomingCallGate
    fun featureFlags(): FeatureFlags
}
// ...
val entryPoint = EntryPointAccessors.fromApplication(applicationContext, FcmEntryPoint::class.java)
```

The last two members exist because the push handler must apply the same call and notification
switches as the UI: `incomingCallGate()` decides whether an incoming call is admitted, and
`featureFlags().awaitFresh()` refreshes the flags before a notification is built (see
[[Feature Flags and Remote Config]]). This is the standard Hilt escape hatch for a component you do
not construct. See [[Notifications]].

## Related

[[Architecture Overview]] - [[Data Layer]] - [[Domain Layer]] - [[Navigation]] -
[[Notifications]] - [[Voice Messages]] - [[Feature Flags and Remote Config]] -
[[Tech Stack and Versions]]
