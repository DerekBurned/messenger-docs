# Async and Reactivity

Every asynchronous primitive in the app, with the property that forced the choice and a real
call site. Coroutines version is `1.11.0` (`gradle/libs.versions.toml:29`).

## Flow (cold) - a stream that does work per collector

A cold `Flow` starts fresh for each collector and stops when collection stops. Used wherever
the source is a query or a listener that should only be active while someone is watching.

Canonical example - the local message stream, `MessageRepositoryImpl.getMessagesStream`
(`data/repository/MessageRepositoryImpl.kt:34`):

```kotlin
override fun getMessagesStream(conversationId: String): Flow<List<Message>> {
    return queryFlow {
        messageBox.query(
            ObxMessage_.conversationId.equal(conversationId)
                .and(ObxMessage_.deleted.equal(false)),
        )
            .order(ObxMessage_.timestamp)
            .build()
    }
        .map { rows -> rows.map { it.toDomain() } }
        .flowOn(Dispatchers.Default)
}
```

Goal: expose the conversation's messages as a live list that re-emits whenever the local
database changes, without the UI polling. It returns a `Flow` (cold) because the underlying
ObjectBox subscription should exist only while a screen is collecting; when the user leaves the
chat, collection ends and the native subscription is torn down (see `queryFlow` below).

Why `flowOn(Dispatchers.Default)`: the `map { it.toDomain() }` converts every row to a domain
model - pure CPU work. `flowOn` moves the upstream (the query emission and the mapping) onto
the `Default` (CPU) dispatcher so the mapping never runs on the main thread, while the
collector still receives on whatever context it collects in. `Default` and not `IO` because
this is computation over already-fetched rows, not blocking IO.

## `callbackFlow` - bridging a listener API into a Flow

`callbackFlow` is the tool for adapting a register/unregister callback into a cold Flow. Three
listener families are bridged this way.

### ObjectBox query subscription - `ObxFlow.asFlow`

`data/local/obx/ObxFlow.kt:8`:

```kotlin
fun <T> Query<T>.asFlow(): Flow<List<T>> = callbackFlow {
    val query = this@asFlow
    val subscription = subscribe().observer { data -> trySend(data) }
    awaitClose {
        subscription.cancel()
        query.close()
    }
}
```

Goal: turn an ObjectBox `Query` into a Flow that emits the full result set now and again on
every change to matching rows. `callbackFlow` is required (not `flow { }`) because the data
arrives through an ObjectBox `observer` callback, not by pulling.

Two cleanups in `awaitClose`, both load-bearing:
- `subscription.cancel()` stops ObjectBox from calling back after the collector is gone.
- `query.close()` releases the native `Query` handle. The `val query = this@asFlow` line
  exists precisely so this resolves to `io.objectbox.query.Query.close()` and not to the
  `ProducerScope.close()` that `callbackFlow` also brings into scope. Without capturing the
  receiver, an unqualified `close()` binds to the channel and the native query handle leaks
  on every unsubscribe. `awaitClose` runs when the collector cancels, so both cleanups are
  tied to the collector's lifetime.

`trySend` (not `send`) because the observer callback is not a suspend context; `trySend` is
non-blocking and, with the default buffered channel, drops nothing under normal load.

`queryFlow` (`ObxFlow.kt:17`) is the same bridge with one difference: it takes a builder and
calls `build()` inside the `callbackFlow`, so every collection gets its own fresh `Query`. With
`asFlow` the `Query` is created once and closed in `awaitClose`, so collecting the same Flow a
second time would subscribe to a closed handle. Repositories use `queryFlow` for streams that a
screen may collect more than once (`getMessagesStream`, the recents lists, deleted-clip ids).

### Firestore snapshot listener - `FirestoreService.getRecentMessagesStream`

`data/remote/firebase/FirestoreService.kt:229`. A `callbackFlow` wraps
`addSnapshotListener(MetadataChanges.INCLUDE)`:

```kotlin
val listener = messagesRef.addSnapshotListener(MetadataChanges.INCLUDE) { snapshot, error ->
    if (error != null) { close(error); return@addSnapshotListener }
    if (snapshot != null) {
        val rows = snapshot.documents.mapNotNull { doc ->
            doc.toObject(RemoteMessageDto::class.java)?.copy(id = doc.id)
                ?.let { it to doc.metadata.hasPendingWrites() }
        }.reversed()
        trySend(rows)
    }
}
awaitClose { listener.remove() }
```

`MetadataChanges.INCLUDE` makes the listener also fire for local, not-yet-acknowledged writes,
and `doc.metadata.hasPendingWrites()` tags each row as "pending" so optimistic sends render
before the server confirms them. `close(error)` propagates a listen failure as a Flow
exception; `awaitClose { listener.remove() }` detaches the Firestore listener when collection
ends. The raw DTO flow is then `map`-ped through the E2EE codec
(`data/remote/firebase/FirestoreService.kt:259`) - see [[End-to-End Encryption]].

`getAllConversations` (`data/remote/firebase/FirestoreService.kt:504`) uses the same shape and
additionally surfaces removed document ids and `snapshot.metadata.isFromCache` in a
`ConversationSync` value.

### Realtime Database ValueEventListener - presence/typing/connection

`data/remote/firebase/RealtimeDbService.kt` bridges three RTDB references with `callbackFlow` +
`ValueEventListener`. Connection state (`RealtimeDbService.kt:23`):

```kotlin
fun observeConnectionState(): Flow<Boolean> = callbackFlow {
    val listener = object : ValueEventListener {
        override fun onDataChange(snapshot: DataSnapshot) {
            trySend(snapshot.getValue(Boolean::class.java) ?: false)
        }
        override fun onCancelled(error: DatabaseError) { close(error.toException()) }
    }
    connectedRef.addValueEventListener(listener)
    awaitClose { connectedRef.removeEventListener(listener) }
}
```

`observePresence` (`RealtimeDbService.kt:59`) and `observeTyping` (`RealtimeDbService.kt:97`)
follow the identical pattern. In every case `awaitClose` removes the listener so a
backgrounded observer stops receiving RTDB pushes.

## StateFlow (hot, conflated, always-has-value)

Used where there must always be a current value to read and new subscribers should get the
latest immediately.

- ViewModel UI state - `MutableStateFlow(initialState)` exposed via `asStateFlow()`
  (`presentation/base/mvi/MviViewModel.kt:19`). The UI needs a value on first frame; a bare `Flow`
  would leave the screen blank until the first emission. See [[Presentation Layer (MVI)]].
- Global call state - `ActiveCallHolder` (`data/remote/call/ActiveCallHolder.kt:37`) is an
  `object` with a `MutableStateFlow<ActiveCall?>`. The call foreground service is the single
  writer; the UI (`presentation/navigation/graph/MainDisplay.kt:152`, `ActiveCallBar`,
  `CallViewModel`) is a reader. A `StateFlow` fits because "the current call" is state with a
  defined current value (possibly `null`), read from many places. See
  [[Calls (Voice and Video)]].
- Call bar presentation mode - `CallBarPresenter.mode`
  (`presentation/components/call/CallBarPresenter.kt:11`) is a `MutableStateFlow<CallBarMode>`
  (BAR / HIDDEN / BUBBLE / VIDEO).
- Media transfer progress - `MediaTransferManager.transfers` re-exposed as
  `IMediaRepository.transfers: StateFlow<Map<String, MediaTransfer>>`
  (`data/repository/MediaRepositoryImpl.kt:60`).
- Voice recording and playback - `VoiceRecorder.state` (`data/media/VoiceRecorder.kt:109`) and
  `ClipPlaybackController.state` (`data/media/ClipPlaybackController.kt:111`) are app-wide
  singletons, so the chat composer, the voice bubbles and the mini player across tabs all read
  one current value. See [[Voice Messages]].
- Feature flags - `RemoteConfigFeatureFlags.state` (`data/remote/firebase/RemoteConfigFeatureFlags.kt:34`)
  is a `StateFlow<FeatureFlagState>` that `publish()` (`:79`) reassigns after each Remote Config
  fetch or activation. A `StateFlow` fits because every reader (the Compose tree through
  `LocalFeatureFlags`, ViewModels through `isEnabled`) needs a current value immediately, before
  any network answer. See [[Feature Flags and Remote Config]].

`_state.update { }` (`presentation/base/mvi/MviViewModel.kt:36`; `ChatsViewModel` does the same
on its `_uiState`, `presentation/screens/chats/ChatsViewModel.kt:70`) is the atomic compare-and-set
update that makes concurrent reducers safe.

## SharedFlow vs Channel for one-shot events

Two mechanisms for "events, not state", chosen per site:

- `SharedFlow(replay = 0, extraBufferCapacity = 16, onBufferOverflow = DROP_OLDEST)` for MVI
  effects (`presentation/base/mvi/MviViewModel.kt:22`). Zero replay so a rotated screen does not
  re-fire the effect; buffered + non-suspending emit so `emitEffect` never blocks.
- `Channel(Channel.BUFFERED).receiveAsFlow()` for the call-screen exit signal in `CallViewModel`
  (`presentation/screens/call/CallViewModel.kt:54`). A `Channel` is single-consumer with
  guaranteed delivery, which matches "navigate away exactly once".
- `Channel(Channel.CONFLATED)` for presence transitions in `PresenceManager`
  (`data/presence/PresenceManager.kt:24`). CONFLATED keeps only the latest requested state, so
  a rapid online->away->online burst collapses to the final value before it is applied.

## Cold vs hot bridging operators

- `flow { }` builders for one-shot suspend work wrapped as a stream: `SendMessageUseCase`
  (`domain/usecase/message/SendMessageUseCase.kt:21`) emits `Resource.Loading/Success/Error`;
  `MessageRepositoryImpl.observeRecentMessages` (`data/repository/MessageRepositoryImpl.kt:140`)
  wraps a collect in a `flow { }.flowOn(Dispatchers.IO)` (IO here because it drives a network
  snapshot stream and writes rows).
- `combine` to merge streams: `ChatViewModel.observePartnerPresence`
  (`presentation/screens/chat/ChatViewModel.kt:485`) combines the presence flow with a 60s
  `tickerFlow` so the "last seen N minutes ago" label refreshes on a timer even when presence
  itself has not changed; `FirebasePresenceService.observeMultiplePresence`
  (`data/presence/FirebasePresenceService.kt:30`) combines a list of per-user presence flows
  into a single `Map`.
- `onEach { }.launchIn(scope)` for fire-and-forget collection into state:
  `ChatViewModel.observePartnerAlias` (`presentation/screens/chat/ChatViewModel.kt:151`) and
  `observeTransfers` (`presentation/screens/chat/ChatViewModel.kt:509`).
- `collectLatest` where a new emission should cancel work started for the previous one:
  `CallForegroundService.observeRemoteStatus`
  (`data/remote/call/CallForegroundService.kt:359`), `CallViewModel.observeActiveCall`
  (`presentation/screens/call/CallViewModel.kt:106`), `SyncCoordinator`
  (`data/sync/SyncCoordinator.kt:40`).
- `onStart` / `catch` to bracket a stream with a loading state and error handling:
  `ChatsViewModel.loadConversations`
  (`presentation/screens/chats/ChatsViewModel.kt:84`), `GetMessagesUseCase`
  (`domain/usecase/message/GetMessagesUseCase.kt:16`).
- `withTimeoutOrNull` to bound a wait: `MediaRepositoryImpl.uploadAndSend`
  (`data/repository/MediaRepositoryImpl.kt:180`) caps a multi-file upload at 35s;
  `ChatViewModel.jumpToMessage` (`presentation/screens/chat/ChatViewModel.kt:387`) bounds each
  page load while searching backward. Voice clips scale the wait with the file instead:
  `clipUploadTimeout` (`data/repository/ClipRepositoryImpl.kt:401`) is 30s plus one second per
  8000 bytes, and the follow-up Firestore write is confirmed with
  `FirestoreService.sendMessage(..., confirmWithin = 20.seconds)`, which wraps `write.await()` in
  `withTimeoutOrNull` (`data/remote/firebase/FirestoreService.kt:130`).
- `RemoteConfigFeatureFlags.awaitFresh` (`data/remote/firebase/RemoteConfigFeatureFlags.kt:60`) puts
  two `withTimeoutOrNull` bounds (1 s for initialisation, 1.5 s for fetch + activate) around Firebase
  `Task`s turned into suspend calls with `await()`, so the incoming-call and notification paths wait
  for a fresh flag value but never longer than a moment. `IncomingCallGate.admit`
  (`data/remote/call/IncomingCallGate.kt:22`) adds a 3 s bound on declining a refused call.
- A shared `Deferred` per key to deduplicate concurrent work: `InFlight`
  (`data/repository/ClipRepositoryImpl.kt:404`) keeps one `async(start = CoroutineStart.LAZY)`
  per message id, so two bubbles asking for the same clip download it once and both await the
  same result. The entry removes itself in `invokeOnCompletion`, so a failure is not cached.
- A `CompletableDeferred` to turn a listener callback into one awaited result:
  `VideoNoteRecorder.Session.result` (`data/media/VideoNoteRecorder.kt:109`) is completed exactly
  once by CameraX's asynchronous `VideoRecordEvent.Finalize`. `stop` awaits it under
  `withTimeoutOrNull(5_000)` in a `try`/`finally` (`:187`): if the waiting coroutine is cancelled
  (the ViewModel was cleared) or the timeout fires, the `finally` keeps the clip as a draft
  instead of dropping it. See [[Video Messages]].
- `withContext(NonCancellable)` to protect shared state from cancellation:
  `ProcessCameraProvider.awaitInstance` is wrapped in it (`VideoNoteRecorder.kt:211`) because
  cancelling the awaiting coroutine would cancel CameraX's process-wide init future and break the
  camera until the process dies. The coroutine checks afterwards whether its session is still
  current and drops the result if not.

## Scopes and dispatchers - which one, and why

- `viewModelScope` - all ViewModel work (`ChatViewModel`, `ChatsViewModel`,
  `AuthViewModel`, `CallViewModel`). Cancelled automatically in `onCleared`, so nothing
  outlives the screen. Manual `Job` handles (for example `typingHeartbeatJob` /
  `typingStopJob`, `presentation/screens/chat/ChatViewModel.kt:94`) are cancelled explicitly to
  debounce.
- A `@Singleton` owning its own `CoroutineScope(SupervisorJob() + Dispatchers.IO)` - used where
  work must outlive any single screen. `MediaRepositoryImpl`
  (`data/repository/MediaRepositoryImpl.kt:57`): an upload started from the chat screen must
  finish even after the user navigates away, so it cannot use `viewModelScope`. `SupervisorJob`
  means one failed upload does not cancel the others; `Dispatchers.IO` because uploads are
  blocking IO. The scope is never cancelled - it lives for the app, matching the `@Singleton`
  lifetime. `ClipRepositoryImpl` (`data/repository/ClipRepositoryImpl.kt:59`) uses the same
  shape plus a `CoroutineExceptionHandler` that logs, so a failing voice-clip task can never crash
  the process; each send runs as a registered `Job` in `sendJobs` so retry and cancel can find it.
- `CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)` - `VoiceRecorder`
  (`data/media/VoiceRecorder.kt:106`) and `ClipPlaybackController`
  (`data/media/ClipPlaybackController.kt:108`). Both drive a platform object (`MediaRecorder`,
  ExoPlayer) and an audio-focus listener; confining every call and state update to the main
  thread serialises start/stop/interrupt and player callbacks without locks, and ExoPlayer must
  be used from the thread it was built on. `VideoNoteRecorder` (`data/media/VideoNoteRecorder.kt:112`)
  uses the same scope for the same reason; CameraX events are delivered on the main executor
  (`ContextCompat.getMainExecutor`), so every session transition happens on one thread.
- `CoroutineScope(SupervisorJob() + Dispatchers.IO)` inside a `Service` - `CallForegroundService`
  (`data/remote/call/CallForegroundService.kt:58`) runs its call state machine (signaling
  writes, ringtones, timeouts) on this scope and cancels it in `onDestroy`
  (`data/remote/call/CallForegroundService.kt:941`) via `scope.coroutineContext[Job]?.cancel()`,
  so no timeout job survives the service. See [[Calls (Voice and Video)]].
- `CoroutineScope(SupervisorJob() + Dispatchers.Default)` - `SyncCoordinator`
  (`data/sync/SyncCoordinator.kt:23`) watches the network flow on `Default` and triggers
  WorkManager; light coordination, not blocking IO, hence `Default`.
- Ephemeral `CoroutineScope(Dispatchers.IO)` per callback - `MyFirebaseMessagingService`
  (`data/remote/firebase/MyFirebaseMessagingService.kt:54`, `:99`). An FCM callback has no
  ViewModel or long-lived scope handy, so each message spins a short-lived IO scope to persist
  and notify. See [[Notifications]].

## Related

[[Presentation Layer (MVI)]] - [[Data Layer]] - [[Calls (Voice and Video)]] -
[[Presence and Typing]] - [[End-to-End Encryption]] - [[Architecture Overview]] -
[[Voice Messages]] - [[Video Messages]] - [[Feature Flags and Remote Config]]
