# Domain Layer

`domain/` is the app-agnostic core: use cases, the interfaces the data layer implements, and
the plain models that cross layers. It has no Android or Firebase imports.

## Use cases

A use case is a single-responsibility class with constructor-injected dependencies and one
public `operator fun invoke`, so call sites read like a function call. They live under
`domain/usecase/` grouped by area (`auth`, `conversation`, `message`, `presence`, `sync`,
`typing`, `user`).

Note: file name and class name do not always match. `GetMessageUseCase.kt` contains the class
`GetMessagesUseCase`; `ObserveConversationsUseCase.kt` / `GetConversationUseCase.kt` back the
`GetConversationsUseCase` the ViewModel injects. Always trust the class name.

Two representative shapes:

Streaming use case - wrap a repository stream and add loading/error framing.
`GetMessagesUseCase` (`domain/usecase/message/GetMessageUseCase.kt:13`):

```kotlin
operator fun invoke(conversationId: String): Flow<Resource<List<Message>>> {
    if (conversationId.isBlank()) return flow { emit(Resource.Error("Invalid conversation ID")) }
    return messageRepository.getMessagesStream(conversationId)
        .map<List<Message>, Resource<List<Message>>> { Resource.Success(it) }
        .onStart { emit(Resource.Loading) }
        .catch { emit(Resource.Error(it.message ?: "Failed to load messages")) }
}
```

Goal: hand the UI a stream that already speaks `Resource` (Loading / Success / Error) so the
ViewModel does not have to try/catch the collect. It returns a cold `Flow` because it just
decorates the repository's cold ObjectBox stream (see [[Async and Reactivity]]); `onStart`
seeds a `Loading` before the first DB emission, `catch` turns a stream failure into a terminal
`Resource.Error` rather than a crash.

Suspend use case producing a result stream - `SendMessageUseCase`
(`domain/usecase/message/SendMessageUseCase.kt:17`) validates input (non-blank, <= 5000 chars),
resolves the current user via `GetCurrentUserUseCase`, builds a `Message.Text` (new `UUID`,
`MessageStatus.SENDING`, reply snippet from `Message.replySnippet()`), calls
`messageRepository.sendMessage`, and emits `Resource.Success/Error` from a `flow { }`.

Use cases added in 0.1.13.1:
- `ObserveSessionStateUseCase` (`auth/`) - combines Firebase auth state, the auth-flow gate and
  the remote profile into a `SessionState` (see [[Navigation]]).
- `DeleteAccountUseCase` (`auth/`) - calls the server deletion, then wipes the device only on
  success (see [[Settings]]).
- `BackfillRecentHistoryUseCase` (`sync/`) - pulls recent messages for the most recent
  conversations after sign-in (see [[Authentication]]).
- `DeleteMessageUseCase` (`message/`) now also takes `IClipRepository` and removes a voice clip's
  uploaded file after a successful delete-for-everyone.
- `MarkConversationAsUnreadUseCase` (`conversation/`) backs the "mark unread" swipe action.

Use cases compose other use cases (`SendMessageUseCase` depends on `GetCurrentUserUseCase`),
which keeps orchestration in the domain rather than the ViewModel. They are provided by
`di/UseCaseModule.kt` (constructor injection, so most need no explicit `@Provides`).

## Repository interfaces

`domain/repository/` defines the contracts the data layer fulfils:
`IAuthRepository`, `IClipRepository`, `IConversationRepository`, `IMessageRepository`,
`IMediaRepository`, `ISyncRepository`, `IUserRepository`. `IClipRepository`
(`domain/repository/IClipRepository.kt`) is the voice-clip contract: `sendClip(...,
replyTo)`, `ensureLocal`, `markPlayed`, `nextVoiceClip`, `evictCachedCopy`,
`observeDeletedClipIds`, `deleteRemoteFile`, `retry`, `cancelSend` (see [[Voice Messages]]).
`IUserRepository` gained `observeRemoteUser` (for the session gate) and `observeCachedUsers` (for
live peer rows). A ViewModel or use case depends on the interface; Hilt
binds the `*Impl` (see [[Dependency Injection (Hilt)]] and [[Data Layer]]). This is the seam
that keeps ObjectBox and Firebase out of the domain.

## Service interfaces

`domain/service/` abstracts device- and vendor-specific services so the domain and
presentation layers do not import Agora or Firebase directly:
- `ICallService` - the RTC engine (join/leave channel, mute, camera, video binding). Bound to
  `AgoraCallService`.
- `ICallSignalingService` - call signaling over Realtime Database. Bound to
  `FirebaseCallSignalingService`.
- `IPresenceService`, `ITypingService` - presence and typing. Bound to
  `FirebasePresenceService` / `FirebaseTypingService`.
- `ISessionCleaner` - logout-time local wipe (`clearOnLogout`), plus `clearOnDelete` which also
  removes the E2EE identity key after account deletion.
- `IAuthFlowGate` - `isBusy: Flow<Boolean>` and `withAuthFlow { }`, marking a sign-in as in
  flight so the session gate waits. Bound to `InMemoryAuthFlowGate`.

`CallEventListener` and `CallConnectionState` also live under `domain/service/`, so the Agora
callback surface is expressed in domain terms and `AgoraCallService`
(`data/remote/call/AgoraCallService.kt`) maps Agora constants onto them.

## Domain models

`domain/model/` holds the cross-layer types. The central one is `Message`
(`domain/model/Message.kt:6`), a sealed hierarchy with `Message.Text`, `Message.Media`,
`Message.Clip` (`:57`) and `Message.Call` variants (referenced throughout, for example the
`when (message)` in `MessageRepositoryImpl.previewLabel`,
`data/repository/MessageRepositoryImpl.kt:209`). `Message.replySnippet()` (`:72`) is the one rule
for reply quotes. A clip's payload is `Clip` (`domain/model/voiceMessage/Clip.kt:6`), a sealed
class with `Voice` (waveform, 1-hour cap `MAX_DURATION_MS`, label "Voice message", `m4a`) and
`Video` (placeholder bubble only); each subtype carries its own `previewLabel`, `fileExtension`
and `maxDurationMs`, so previews, pushes and UI share one label source. Session types:
`SessionState` (`Resolving` / `SignedOut(reason)` / `Active(user)`) with `SessionEndReason`
(`PROFILE_MISSING`, `ACCOUNT_DELETED`), and `RemoteUserSnapshot(user, authoritative)`. `User`
gained `deleted` / `deletedAt` for tombstoned accounts. Others: `Conversation`, `User`, `DomainUser`,
`Profile`, `MediaItem`, `MediaTransfer`, `MessageStatus` (SENDING/SENT/READ/FAILED),
`CallHistoryEntry`, `CallSignal`, `CallStatus`, `PresenceState` (ONLINE/AWAY/OFFLINE),
`UserPresence`, `PhoneNumber`, `PhoneVisibility`, `SyncStatus`, and the sync constants under
`domain/model/sync/` (`SyncAction`, `SyncEntityType`).

Because these are ordinary Kotlin types, they double as the state the UI renders (a
`ChatUiState` holds `List<Message>` directly) and the values Firestore/ObjectBox mappers
produce. Message-type discriminators used by the call/history paths live as constants on
`Message` (`TYPE_TEXT`, `TYPE_MEDIA`, `TYPE_MISSED_CALL`, `TYPE_UNREACHED_CALL`,
`TYPE_ENDED_CALL`, `TYPE_VOICE_CLIP`, `TYPE_VIDEO_CLIP`; `Message.kt:81`-`:87`; the call types are
used in `data/repository/CallHistoryRepository.kt:79`).

## Why a domain layer at all

- Testability - use cases are plain classes with faked repositories; no Robolectric needed.
- Swappability - the `I*Repository` / `I*Service` seams let the data layer change
  (Room -> ObjectBox already happened; Agora or Firebase could be swapped) without touching
  presentation.
- One place for orchestration - multi-step flows (validate, resolve user, send, retry) live in
  a use case, not scattered across ViewModels.

Trade-off accepted: many small classes and some file/class-name drift. The project keeps the
pattern lightweight (no separate `Result`-mapping ceremony beyond `Resource` and Kotlin's
`Result`).

## Related

[[Data Layer]] - [[Presentation Layer (MVI)]] - [[Dependency Injection (Hilt)]] -
[[Architecture Overview]] - [[Voice Messages]]
