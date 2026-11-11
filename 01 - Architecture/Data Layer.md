# Data Layer

`data/` implements every domain interface and is the only layer that touches ObjectBox,
Firebase, Agora, the crypto codec, media transfer, and the OS. It is where the offline-first
model is enforced.

Sub-packages: `local/` (ObjectBox + prefs), `remote/` (auth, firebase, call, dto), `crypto/`
(E2EE), `device/` (this device's registry row for push), `media/`, `presence/`, `service/`,
`repository/`, `sync/`.

## Offline-first principle

The local ObjectBox store is the source of truth the UI reads. The network is a writer and a
refresher behind it. A repository method typically:
1. writes/updates ObjectBox immediately (optimistic), which
2. re-emits the local `Query.asFlow()` stream the UI is collecting, and only then
3. pushes to Firestore, falling back to the sync queue on failure.

`MessageRepositoryImpl.sendMessage` (`data/repository/MessageRepositoryImpl.kt:53`) is the
model: `messageBox.put(...)` then `updateConversationPreview` then
`firestoreService.sendMessage`; on failure it flips the row to `FAILED` and calls
`enqueueRetry`. The read side never blocks on the network - see the flow analysis in
[[Async and Reactivity]].

## ObjectBox persistence

Version `5.4.2` (`gradle/libs.versions.toml:16`), annotation processing via kapt
(`app/build.gradle.kts:11`; the ObjectBox plugin itself is `:12`). Entities are declared one per file in `data/local/obx/`
(`ObxMessage.kt`, `ObxConversation.kt`, `ObxUser.kt`, `ObxRecentContact.kt`, `ObxSyncQueueItem.kt`,
...) and generated into `MyObjectBox`.

Entity conventions (`data/local/obx/ObxMessage.kt:14`): `@Id var boxId: Long` is the internal
key; the business key is a separate `@Unique(onConflict = ConflictStrategy.REPLACE) var uid`,
so an upsert of the same remote id replaces the row rather than duplicating it. Query-hot
columns carry `@Index` - `ObxMessage.conversationId` and `ObxMessage.type`
(`ObxMessage.kt:16`, `:24`); the `type` index exists so the call-history query
(`type.oneOf(...)`) is not a full-table scan. Complex fields are stored via `@Convert`
property converters (for example `MediaItemListConverter` for `mediaItemsJson`,
`PhoneNumberConverter`, `StringListConverter`), so a `List<MediaItem>` persists as JSON in one
column. Voice clips use the same trick: `ObxMessage.clipJson` holds a `ClipPayload` through
`ClipConverter` (`ObxMessage.kt:36`), and `clipPlayed` (`:38`) drives the unplayed dot. Deleted
accounts are kept as rows with `ObxUser.deleted` / `deletedAt` (`ObxUser.kt:23`).

The eight boxes are provided as singletons in `di/ObjectBoxModule.kt`: `provideBoxStore`
(`di/ObjectBoxModule.kt:30`) builds `MyObjectBox` and, in debug only, recovers from a
`DbSchemaException` by wiping and rebuilding the store - a schema downgrade on a dev device
auto-heals instead of crashing on launch. Each `Box<Obx*>` is a separate `@Provides @Singleton`
so repositories inject exactly the boxes they need.

`ObxRecentContact` (`data/local/obx/ObxRecentContact.kt:9`) backs the search-suggestion lists: one row
per contact keyed on a `@Unique userId`, carrying `username`, `avatarUrl`, and two timestamps
`lastSearchedAt` / `lastOpenedAt`. Recording a search or a chat-open upserts the row (updating one
timestamp), so the same person is never duplicated across the two suggestion sections.

Streaming from ObjectBox uses the `Query<T>.asFlow()` extension
(`data/local/obx/ObxFlow.kt:8`); its `callbackFlow` + `awaitClose` cleanup is dissected in
[[Async and Reactivity]].

Mappers live in `data/local/obx/mappers/` (`ObxMessageMappers.kt`, `ObxConversationMappers.kt`,
`ObxUserMappers.kt`; `toObx()` / `toDomain()`), keeping ObjectBox annotations out of the domain
models.

## Repositories

`data/repository/` (all `@Singleton`, bound in `di/RepositoryModule.kt`):
- `MessageRepositoryImpl` (`IMessageRepository`) - local stream + send/delete/read + Firestore
  refresh + older-page loading (`loadOlderMessages`, `:151`) + one-off shared-media pages for the
  chat profile (`fetchSharedMediaPage`, `:171`, backed by `FirestoreService.fetchMessagesOfTypes`,
  `data/remote/firebase/FirestoreService.kt:317`) + read-state-preserving merge
  (`putPreservingReadState`, `data/repository/MessageRepositoryImpl.kt:190`). Delete rules explained in
  [[Chat and Messaging]].
- `MediaRepositoryImpl` (`IMediaRepository`) - owns its own app-scoped coroutine scope for
  uploads/downloads; see [[Chat and Messaging]] and the scope rationale in
  [[Async and Reactivity]].
- `ClipRepositoryImpl` (`IClipRepository`, `data/repository/ClipRepositoryImpl.kt:48`) - voice
  clips: send through the outbox, confirmed Firestore write, deduplicated download
  (`InFlight`), recovery after process death, cleanup of deleted clips. See [[Voice Messages]].
- `ConversationRepositoryImpl` (`IConversationRepository`) - conversation list stream,
  create/find, delete-for-me vs delete-for-everyone via `visibleTo`/`clearedAt`.
- `UserRepositoryImpl` (`IUserRepository`) - user lookup/search, contact aliases, avatar
  observation.
- `AuthRepositoryImpl` (`IAuthRepository`) - wraps `FirebaseAuthService`.
- `SyncRepositoryImpl` (`ISyncRepository`) - drains the sync queue.
- `CallHistoryRepository` - call-log entries derived from call-type messages
  (`data/repository/CallHistoryRepository.kt:20`); see [[Calls (Voice and Video)]].
- `RecentContactsRepository` (`data/repository/RecentContactsRepository.kt`) - a plain
  box-backed `@Singleton` (same shape as `CallHistoryRepository`, no interface) reading/writing
  `ObxRecentContact`. Exposes `recordSearched` / `recordOpened` (IO-dispatched upserts) and two
  reactive `queryFlow`-backed reads `observeRecentSearched` / `observeRecentOpened`, mapped to
  `List<User>`. Powers the search overlay's "Recently searched" / "Recently opened" sections.
- `ConversationReconciler` - reconciles remote conversation snapshots against local rows.

`data/local/repository/LocalRepositoryImpl` (`ILocalRepository`) centralizes direct box access
used by several repositories.

## Firebase services

BOM `34.10.0` (`gradle/libs.versions.toml:13`) manages Firebase versions. Services wrap the
SDKs and return domain types / `Result`.

- `FirestoreService` (`data/remote/firebase/FirestoreService.kt`) - users, conversations, and
  the `conversations/{id}/messages` subcollection. Reads are snapshot-listener `callbackFlow`s
  (`getRecentMessagesStream:229`, `getAllConversations:504`); writes are suspend `Result`
  methods using `kotlinx-coroutines-play-services` `await()`. It integrates the E2EE codec on
  encode/decode. Delete-for-everyone sets the `deleted` field (`:204`); delete-for-me adds the
  uid to a `deletedFor` array (`:177`); clearing a conversation writes `clearedAt.{uid}` and
  removes the uid from `visibleTo` (`:455`).
- `RealtimeDbService` (`data/remote/firebase/RealtimeDbService.kt`) - presence, typing, and
  `.info/connected`. Presence uses `onDisconnect()` so the server marks a user offline if the
  socket drops (`:52`). Reads are `ValueEventListener` `callbackFlow`s.
- `FirebaseStorageService` - media upload/download URLs (used by `MediaTransferManager`).
- `FirebaseMessagingManager` / `MyFirebaseMessagingService` / `FcmTokenSyncer` - FCM; see
  [[Notifications]].
- `FirebaseCallSignalingService` - call signaling over RTDB; it also writes the `callChannels/<channel>`
  record (caller, callee, server timestamp) in the same multi-path update as the signal
  (`callSignalUpdate`, `data/remote/call/FirebaseCallSignalingService.kt:159`), which the
  `getRtcToken` function reads to decide who may receive a call token; see
  [[Calls (Voice and Video)]].
- `RemoteConfigFeatureFlags` and `CrashlyticsUserSync` - the Remote Config client behind the feature
  flags and the Crashlytics user-id hook; see [[Feature Flags and Remote Config]].
- `DeviceRegistry` (`data/device/DeviceRegistry.kt`) - writes this device's row (with its FCM token)
  under `users/<uid>/devices` and clears the token on sign-out.

Presence and typing are adapted to the domain interfaces by `FirebasePresenceService` (in
`data/presence/`, next to `PresenceManager` and `AppLifecycleObserver`) and
`FirebaseTypingService` (in `data/service/`, which also holds `AndroidSessionCleaner` and
`InMemoryAuthFlowGate`); both adapt `RealtimeDbService` and filter out the current user. See
[[Presence and Typing]].

## DTOs and mappers

`data/remote/dto/` holds the Firestore wire types: `RemoteMessageDto` (in `dto/message/RemoteMessageDto.kt`,
next to `RemoteMediaItemDto` and `RemoteClipDto`),
`ConversationDTO`, `UserDTO`, `CallSignalDto`. Extension mappers `toDomain()` / `toRemoteDto()`
convert to and from domain models, and carry the encryption fields (`enc`, `ciphertext`,
`nonce`, `senderEpoch`, `recipientEpoch`, `recipientId`) that the codec reads/writes. The DTO
also carries `deletedFor` and the `deleted` flag used by the deletion rules.

## Offline sync queue

When a write cannot reach Firestore, the repository enqueues an `ObxSyncQueueItem`
(`entityType`, `entityId`, `action`, `timestamp`, `retryCount`) - see `enqueueRetry`
(`data/repository/MessageRepositoryImpl.kt:80`). The queue is drained by:
- `SyncCoordinator` (`data/sync/SyncCoordinator.kt`) - `@Singleton` started in
  `MessengerApplication.onCreate`; on its own `Default` scope it triggers a one-time sync on
  startup and again whenever the network returns (`collectLatest` over `INetworkObserver`).
- `SyncManager` (`data/sync/SyncManager.kt`) - schedules a periodic (15-minute,
  network-constrained, exponential-backoff, `KEEP`) `SyncWorker` and can trigger one-time
  syncs.
- `SyncWorker` (`data/sync/SyncWorker.kt`) - a `@HiltWorker` `CoroutineWorker` that delegates
  to `SyncAllDataUseCase` and returns `Result.retry()` on failure so WorkManager backs off.
- `NetworkObserver` / `INetworkObserver` - a Flow of connectivity used by the coordinator.

`@HiltWorker` needs `androidx.hilt:hilt-compiler` (`app/build.gradle.kts:167`) and the
`HiltWorkerFactory` wired via `MessengerApplication` `Configuration.Provider`
(`MessengerApplication.kt:150`); otherwise WorkManager cannot construct the worker with its
`SyncAllDataUseCase` dependency.

## Media transfer and cache

`data/media/`:
- `MediaTransferManager` - performs uploads/downloads via `FirebaseStorageService`, exposing a
  `StateFlow<Map<String, MediaTransfer>>` of progress that `MediaRepositoryImpl.transfers`
  re-exposes upward.
- `MediaCache` - on-disk cache of originals/downloads keyed by item id and kind
  (`cacheDir/chat_media`, `MediaCache.DIR`). For voice clips it also owns two `filesDir`
  folders that must survive cache eviction: `clip_outbox` (`OUTBOX_DIR`, unsent clips until the
  send is confirmed, then promoted into the cache) and `voice_drafts` (`DRAFTS_DIR`).
  `deleteStaleTemporaries` (`data/media/MediaCache.kt:62`) removes leftover `recording_`, `.dl`,
  `.part` and `.enc` files on startup.
- `MediaFileTransfer` - the per-file upload/download with encryption; downloads go through a
  unique temp file and an atomic move, so a crash never leaves a half-written file under the
  final name.
- `VoiceRecorder`, `VoiceDraftStore`, `ClipPlaybackController` - the voice recording, draft
  persistence (`<conversationId>.m4a` + JSON sidecar) and ExoPlayer playback singletons. See
  [[Voice Messages]].
- `MediaMetadata` - extracts width/height/duration and a BlurHash for a picked file.
- `BlurHash` / `BlurHashCache` - compact blur placeholders shown before an image loads.
- `MediaStoreGallery` - reads the device gallery for the picker.

`MediaCipher` (`data/crypto/`) encrypts/decrypts media files with a per-file key when the
conversation is encryptable; the key travels inside the encrypted message payload. See
[[Chat and Messaging]] and [[End-to-End Encryption]].

## Related

[[Domain Layer]] - [[Async and Reactivity]] - [[Dependency Injection (Hilt)]] -
[[Chat and Messaging]] - [[Notifications]] - [[Presence and Typing]] -
[[End-to-End Encryption]] - [[Voice Messages]] - [[Architecture Overview]]
