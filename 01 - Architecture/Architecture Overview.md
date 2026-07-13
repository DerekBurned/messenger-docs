# Architecture Overview

## Shape of the app

Single Gradle module (`:app`, `app/build.gradle.kts`), one Kotlin source set, package root
`com.example.messenger`. Inside that root the code is split into the classic three layers plus
DI and util:

```
com.example.messenger
  presentation/   Compose UI, ViewModels, MVI State/Intent/Effect, navigation, notifications
  domain/         use cases, repository interfaces, service interfaces, domain models
  data/           repository impls, ObjectBox, Firebase services, crypto, media, sync, calls
  di/             Hilt modules
  util/           Resource, DateUtils, validation, similarity, constants
  MessengerApplication.kt   the @HiltAndroidApp entry point
```

File counts at time of writing: `presentation` 137, `data` 67, `domain` 55, `di` 10,
`util` 10 Kotlin files.

## Dependency rule

Dependencies point inward. `presentation` depends on `domain`; `data` depends on `domain`;
`domain` depends on nothing app-specific. The seam is a set of interfaces in
`domain/repository` and `domain/service` (for example `IMessageRepository`,
`IConversationRepository`, `ICallService`, `IPresenceService`). Implementations live in
`data` and are bound to those interfaces by Hilt - see [[Dependency Injection (Hilt)]]. A
ViewModel never sees an `*Impl` type; it depends on the interface (or on a use case that
does).

The domain models (`domain/model/`) are the currency crossing layers. `data` converts
between three representations:
- ObjectBox entities (`data/local/obx/ObxEntities.kt`, suffix `Obx*`) for local storage,
- Firestore DTOs (`data/remote/dto/`, for example `RemoteMessageDto`) for the wire,
- domain models (`domain/model/`, for example `Message`, `Conversation`, `User`) for use
  cases and UI.

Mappers (`data/local/obx/ObxMappers.kt`, `data/remote/dto/*` `toDomain`/`toRemoteDto`) do the
translation, so neither ObjectBox annotations nor Firestore field shapes leak upward.

## The three layers in one paragraph each

Presentation - Compose screens observe a `StateFlow<UiState>` from a ViewModel and send
`UiIntent`s back; one-shot `UiEffect`s (navigate, show error) are delivered through a
`SharedFlow`. Full detail in [[Presentation Layer (MVI)]] and [[Navigation]].

Domain - single-responsibility use cases (`domain/usecase/...`) expose `operator fun invoke`
and orchestrate repositories. They are plain classes, constructor-injected, with no Android
dependencies. Full detail in [[Domain Layer]].

Data - repositories implement the domain interfaces and are the only place that touches
ObjectBox, Firestore, Realtime Database, Storage, or the crypto codec. They are the merge
point of the offline-first model: local ObjectBox is the source of truth the UI reads, the
network is a writer/refresher behind it. Full detail in [[Data Layer]].

## End-to-end data flow: sending and seeing a text message

1. UI dispatches `ChatIntent.SendMessage(text)` to `ChatViewModel`
   (`presentation/viewmodel/ChatViewModel.kt:140`).
2. The ViewModel calls `SendMessageUseCase`
   (`domain/usecase/message/SendMessageUseCase.kt:17`), which builds a `Message.Text` with a
   `UUID` id and `MessageStatus.SENDING`, then calls the repository.
3. `MessageRepositoryImpl.sendMessage` (`data/repository/MessageRepositoryImpl.kt:52`) writes
   the message to ObjectBox first (`messageBox.put`), updates the conversation preview, then
   attempts the Firestore write via `FirestoreService.sendMessage`
   (`data/remote/firebase/FirestoreService.kt:95`). On network failure it flips the row to
   `FAILED` and enqueues a retry in the sync queue.
4. The message is already on screen, because the UI reads a cold ObjectBox stream:
   `getMessagesStream` (`data/repository/MessageRepositoryImpl.kt:34`) returns a
   `Query.asFlow()` that re-emits the whole list on every local write. The `put` in step 3
   triggers a re-emission; the ViewModel's `GetMessagesUseCase` collects it and calls
   `setState`.
5. Separately, `observeRecentMessages` / `getRecentMessagesStream`
   (`data/remote/firebase/FirestoreService.kt:204`) keeps a Firestore snapshot listener open
   and writes incoming remote rows into ObjectBox, which again re-emits to the same UI stream.

The important property: the read path and the write path are decoupled through ObjectBox. The
UI never waits on the network to show a message. See [[Async and Reactivity]] for why each
stream is cold/hot and which dispatcher it runs on, and [[Chat and Messaging]] for the full
feature.

## App startup wiring

`MessengerApplication` (`MessengerApplication.kt:34`) is `@HiltAndroidApp` and
`Configuration.Provider` (it supplies the `HiltWorkerFactory` so `@HiltWorker` workers get
their dependencies). Its `onCreate` (`MessengerApplication.kt:61`) is the composition root for
long-lived, app-scoped machinery, all `@Inject`ed as singletons:

- Coil `SingletonImageLoader` with a 10%-of-RAM memory cache and a 30 MB disk cache, plus the
  `VideoFrameDecoder` for video thumbnails.
- `NotificationChannels.ensureCreated` - registers channels up front (see [[Notifications]]).
- `AppLifecycleObserver` on the `ProcessLifecycleOwner` - drives foreground/background
  presence (see [[Presence and Typing]]).
- `SyncCoordinator.start()` and `SyncManager.schedulePeriodicSync()` - the offline sync
  machinery (see [[Data Layer]]).
- `FcmTokenSyncer.start()`, `IncomingCallCoordinator.start()`,
  `TelecomCallManager.registerPhoneAccount()` - push and call plumbing.
- `E2eeBootstrap.publishKeys()` - publishes this device's public identity key (see
  [[End-to-End Encryption]]).

Because these are wired in `Application.onCreate`, they live for the whole process, which is
why several of them own their own `CoroutineScope` rather than borrowing a `viewModelScope` -
see the scope discussion in [[Async and Reactivity]].

## Related

[[Presentation Layer (MVI)]] - [[Domain Layer]] - [[Data Layer]] -
[[Dependency Injection (Hilt)]] - [[Async and Reactivity]] - [[Navigation]] -
[[Tech Stack and Versions]]
