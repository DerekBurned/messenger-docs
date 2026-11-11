# Chat and Messaging - Code

Companion to [[Chat and Messaging]]. The full source for the chat feature, block by block. This
is the largest feature; snippets are copied verbatim from the repo. Conceptual "why" lives in the
main note.

Files covered:
- MVI: `presentation/screens/chat/ChatUiState.kt` (with `PendingAttachment.kt` beside it),
  `presentation/screens/chat/ChatIntent.kt`, `presentation/screens/chat/ChatEffect.kt`
- `presentation/screens/chat/ChatViewModel.kt`
- Use cases `domain/usecase/message/{GetMessagesUseCase, SendMessageUseCase}.kt` (others noted)
- `data/repository/MessageRepositoryImpl.kt`
- `data/repository/MediaRepositoryImpl.kt`
- `data/remote/firebase/FirestoreService.kt` (message methods)
- `data/local/obx/ObxFlow.kt`

---

## 1. MVI contract

`PendingAttachment.kt` and `ChatUiState.kt` (blank lines between fields dropped; the voice and
video fields are covered in [[Voice Messages - Code]] and [[Video Messages - Code]]):

```kotlin
@Immutable
data class PendingAttachment(val uri: Uri, val kind: String)

@Immutable
data class ChatUiState(
    val isLoading: Boolean = false,
    val messages: List<Message> = emptyList(),
    val error: UiText? = null,
    val isSending: Boolean = false,
    val partnerPresence: UserPresence = UserPresence(),
    val partnerUsername: String = "",
    val partnerAvatarUrl: String? = null,
    val partnerDeleted: Boolean = false,
    val typingUsernames: List<String> = emptyList(),
    val isPartnerTyping: Boolean = false,
    val replyingTo: Message? = null,
    val forwardingMessage: Message? = null,
    val currentUserId: String = "",
    val partnerLastSeenDisplay: String = "",
    val isLoadingOlder: Boolean = false,
    val hasMoreOlder: Boolean = true,
    val firstUnreadMessageId: String? = null,
    val unreadAnchorResolved: Boolean = false,
    val highlightedMessageId: String? = null,
    val transfers: Map<String, MediaTransfer> = emptyMap(),
    val pendingAttachments: List<PendingAttachment> = emptyList(),
    val conversationMedia: List<MediaItem> = emptyList(),
    val recording: VoiceRecordState = VoiceRecordState.Idle,
    val videoRecording: VideoRecordState = VideoRecordState.Idle,
    val clipMode: ClipKind = ClipKind.VOICE,
    val clipDraft: ClipRecording? = null,
    val micDenied: Boolean = false,
    val micDeniedForVideo: Boolean = false,
) : UiState
```

`ChatIntent.kt`:

```kotlin
sealed interface ChatIntent : UiIntent {
    data class TextChanged(val text: String) : ChatIntent
    data class SendMessage(val text: String) : ChatIntent
    data class MessagesSeen(val messages: List<Message>) : ChatIntent
    data object LoadOlderMessages : ChatIntent
    data class DeleteMessage(val message: Message, val forEveryone: Boolean) : ChatIntent
    data class SetReplyTo(val message: Message) : ChatIntent
    data class JumpToMessage(val messageId: String) : ChatIntent
    data class Forward(val message: Message) : ChatIntent
    data object ClearReply : ChatIntent
    data object ClearForward : ChatIntent
    data object ClearError : ChatIntent
    data class DownloadMedia(val item: MediaItem) : ChatIntent
    data class ViewerDownload(val item: MediaItem) : ChatIntent
    data class CancelUpload(val messageId: String, val itemId: String) : ChatIntent
    data class CancelDownload(val itemId: String) : ChatIntent
    data class ToggleAttachment(val uri: Uri, val kind: String) : ChatIntent
    data class AddAttachment(val uri: Uri, val kind: String) : ChatIntent
    data object ClearAttachments : ChatIntent
    data class RetryMedia(val messageId: String) : ChatIntent
    data object StartRecording : ChatIntent
    data object LockRecording : ChatIntent
    data object CancelRecording : ChatIntent
    data object FinishRecording : ChatIntent
    data object InterruptRecording : ChatIntent
    data object SendVoiceDraft : ChatIntent
    data object DiscardVoiceDraft : ChatIntent
    data class MicPermissionDenied(val permanently: Boolean, val video: Boolean = false) : ChatIntent
    data object DismissMicPrompt : ChatIntent
    data object ToggleClipMode : ChatIntent
    data class PlayClip(val messageId: String) : ChatIntent
    data class SeekClip(val fraction: Float) : ChatIntent
    data class RetryClip(val messageId: String) : ChatIntent
    data class CancelClipSend(val messageId: String) : ChatIntent
}
```

`ChatEffect.kt`:

```kotlin
sealed interface ChatEffect : UiEffect {
    data class ShowError(val message: UiText) : ChatEffect
}
```

---

## 2. ChatViewModel

`presentation/screens/chat/ChatViewModel.kt` (655 lines) - the MVI orchestration. The ViewModel changed a
great deal since 0.1.13.1 (feature-flag gating, clips, a different typing scheme), so it is walked
through here with refs rather than copied. `path:line` refs are relative to
`app/src/main/java/com/example/messenger/`.

- Class and wiring (`:62`-`:85`): `@HiltViewModel`, extends `MviViewModel<ChatUiState, ChatIntent,
  ChatEffect>`; 20 injected collaborators including the three message use cases, `IMediaRepository`,
  `IClipRepository`, the two recorders, `VoiceDraftStore`, `ClipPlaybackController` and
  `FeatureFlags`. Constants (`:101`-`:111`): `PAGE_SIZE = 100`, `TYPING_HEARTBEAT_MS = 3000`,
  `TYPING_STOP_DELAY_MS = 2000`, `LAST_SEEN_TICK_MS = 60_000`, `HIGHLIGHT_MS = 2500`,
  `MAX_JUMP_PAGES = 50`, `JUMP_PAGE_TIMEOUT_MS = 2000`.
- `init` (`:113`): reads `conversationId`, `partnerId`, `partnerName` and the seeded avatar from
  `SavedStateHandle` (`:87`-`:91`), seeds the display state, starts `observePartnerAlias`,
  `observePartnerAvatar` (sets `partnerDeleted`), `observeTransfers`, `observeRecording` and
  `observeVoiceDraft`, records the partner as recently opened, and - when a conversation id is present
  - `loadMessages`, `syncMessages`, `markConversationRead`, `observePartnerPresence` and `observeTyping`.
- `handleIntent` (`:190`): first `requiredFlag(intent)` (`:230`) and `featureFlags.refuses(flag)`
  (`:191`-`:192`), then a `when` over every `ChatIntent` (`:193`-`:227`). See the feature-flag table in
  [[Chat and Messaging]].
- Messages: `loadMessages` (`:429`) collects the `Resource` stream; on the first non-empty success it
  resolves `firstUnreadMessageId` once and flattens `Message.Media` items into `conversationMedia`
  (`:437`-`:457`). `syncMessages` (`:419`) collects the Firestore refresh and swallows errors.
  `loadOlder` / `loadOlderPage` (`:366`-`:385`) fetch a page and set `hasMoreOlder = fetched >= PAGE_SIZE`.
  `jumpToMessage` (`:387`) loops `loadOlderPage` under a per-page `withTimeoutOrNull` and a guard of
  `MAX_JUMP_PAGES`, then highlights for 2.5 s or emits "Original message not found".
- Reading and presence: `onMessagesSeen` (`:467`) marks each message once via `markedAsReadIds`;
  `observePartnerPresence` (`:485`) combines presence with a 60 s `tickerFlow` (`:501`);
  `observeTyping` (`:515`) fills `typingUsernames` / `isPartnerTyping`.
- Typing (outgoing): `onTextChanged` (`:530`) starts a heartbeat loop calling `typingService.setTyping`
  every 3 s while the user types and restarts a 2 s idle job that calls `stopTyping` (`:555`), which cancels
  both and clears the node; blank input clears at once.
- Sending: `toggleAttachment` (`:568`) / `addAttachment` (`:581`) edit `pendingAttachments`;
  `sendMessage` (`:589`) routes to `sendMediaMessage` (`:610`) when attachments exist, otherwise stops
  typing, sets `isSending` and collects `sendMessageUseCase`; `deleteMessage` (`:627`) emits
  `ChatEffect.ShowError` on failure.
- Clips: `startRecording` (`:257`), `lockRecording` (`:273`), `cancelRecording` (`:277`),
  `interruptRecording` (`:282`), `finishRecording` (`:287`), `sendRecording` (`:304`) and `playClip`
  (`:326`); `observeRecording` (`:338`) and `observeVoiceDraft` (`:347`). Covered in
  [[Voice Messages - Code]] and [[Video Messages - Code]].
- `onCleared` (`:646`): cancels the typing jobs, calls `typingService.clearTypingNow` and interrupts both
  recorders so the microphone and camera are released.

A flavour of the gating, the only part worth quoting - the intent-to-flag map starts like this
(`:230`-`:234`):

```kotlin
private fun requiredFlag(intent: ChatIntent): FeatureFlag? = when (intent) {
    is ChatIntent.SetReplyTo -> FeatureFlag.MESSAGE_REPLY
    is ChatIntent.JumpToMessage -> FeatureFlag.MESSAGE_REPLY_JUMP
```

---

## 3. Message use cases

`GetMessagesUseCase.kt` (class `GetMessagesUseCase`):

```kotlin
class GetMessagesUseCase @Inject constructor(
    private val messageRepository: IMessageRepository
) {
    operator fun invoke(conversationId: String): Flow<Resource<List<Message>>> {
        if (conversationId.isBlank()) return flow { emit(Resource.Error("Invalid conversation ID")) }
        return messageRepository.getMessagesStream(conversationId)
            .map<List<Message>, Resource<List<Message>>> { Resource.Success(it) }
            .onStart { emit(Resource.Loading) }
            .catch { exception -> emit(Resource.Error(exception.message ?: "Failed to load messages")) }
    }
}
```

`SendMessageUseCase.kt`:

```kotlin
class SendMessageUseCase @Inject constructor(
    private val messageRepository: IMessageRepository,
    private val getCurrentUserUseCase: GetCurrentUserUseCase
) {
    suspend operator fun invoke(
        conversationId: String, text: String, replyTo: Message? = null,
    ): Flow<Resource<Unit>> = flow {
        try {
            if (text.isBlank()) emit(Resource.Error("Message is empty"))
            if (text.length > 5000) emit(Resource.Error("Message is too long"))
            val currentUser = getCurrentUserUseCase()
            if (currentUser == null) { emit(Resource.Error("User not logged in")); return@flow }
            val message = Message.Text(
                id = UUID.randomUUID().toString(),
                conversationId = conversationId,
                senderId = currentUser.id,
                text = text.trim(),
                timestamp = System.currentTimeMillis(),
                status = MessageStatus.SENDING,
                replyToMessageId = replyTo?.id,
                replyToText = replyTo?.replySnippet(),
                replyToSenderId = replyTo?.senderId,
            )
            messageRepository.sendMessage(message).fold(
                onSuccess = { emit(Resource.Success(Unit)) },
                onFailure = { e -> emit(Resource.Error(e.message ?: "Failed to send message")) })
        } catch (e: Exception) { emit(Resource.Error(e.message ?: "Internal error occurred")) }
    }
}
```

The snippet rule lives on the domain model, so text sends and voice replies share it
(`domain/model/Message.kt:72`):

```kotlin
fun replySnippet(): String? = when (this) {
    is Text -> text.take(REPLY_SNIPPET_MAX)
    is Media -> caption.take(REPLY_SNIPPET_MAX).ifBlank { "Media" }
    is Clip -> clip.previewLabel
    is Call -> null
}
```

The others are the same thin shape: `LoadOlderMessagesUseCase`, `MarkMessageAsReadUseCase`,
`MarkConversationAsReadUseCase`, `SyncMessagesUseCase`, and `DeleteMessageUseCase` (which chooses
`deleteMessageForEveryone` vs `deleteMessageForMe` on the `forEveryone` flag and wraps the result
in a `Resource` flow).

---

## 4. MessageRepositoryImpl

`data/repository/MessageRepositoryImpl.kt` in full:

```kotlin
class MessageRepositoryImpl @Inject constructor(
    private val messageBox: Box<ObxMessage>,
    private val conversationBox: Box<ObxConversation>,
    private val syncQueueBox: Box<ObxSyncQueueItem>,
    private val firestoreService: FirestoreService,
    private val authService: FirebaseAuthService,
) : IMessageRepository {

    override fun getMessagesStream(conversationId: String): Flow<List<Message>> {
        return queryFlow {
            messageBox.query(
                ObxMessage_.conversationId.equal(conversationId)
                    .and(ObxMessage_.deleted.equal(false)),
            ).order(ObxMessage_.timestamp).build()
        }
            .map { rows -> rows.map { it.toDomain() } }
            .flowOn(Dispatchers.Default)
    }

    private fun cutoffFor(conversationId: String): Long =
        conversationBox.query(ObxConversation_.uid.equal(conversationId)).build()
            .use { it.findFirst() }?.clearedAtForMe ?: 0L

    override suspend fun sendMessage(message: Message): Result<Unit> {
        return try {
            messageBox.put(message.toObx())
            updateConversationPreview(message)
            val result = firestoreService.sendMessage(message.withStatus(MessageStatus.SENT))
            if (result.isFailure) {
                updateMessage(message.id) { it.status = MessageStatus.FAILED.name }
                enqueueRetry(message.id)
            }
            Result.success(Unit)
        } catch (e: Exception) {
            runCatching { updateMessage(message.id) { it.status = MessageStatus.FAILED.name } }
            runCatching { enqueueRetry(message.id) }
            Result.failure(e)
        }
    }

    override suspend fun persistIncomingMessage(message: Message) {
        val alreadyStored = messageBox.query(ObxMessage_.uid.equal(message.id)).build()
            .use { it.findFirst() } != null
        if (alreadyStored) return
        messageBox.put(message.toObx())
        updateConversationPreview(message)
    }

    private fun enqueueRetry(messageId: String, action: String = SyncAction.SEND) {
        syncQueueBox.put(ObxSyncQueueItem(
            entityType = SyncEntityType.MESSAGE, entityId = messageId,
            action = action, timestamp = System.currentTimeMillis()))
    }

    override suspend fun deleteMessageForEveryone(message: Message): Result<Unit> {
        val currentUserId = authService.getCurrentUserId()
        if (currentUserId == null || message.senderId != currentUserId) {
            return Result.failure(SecurityException("Only own messages can be deleted for everyone"))
        }
        return try {
            updateMessage(message.id) { it.deleted = true }
            val result = firestoreService.deleteMessage(message)
            if (result.isFailure) enqueueRetry(message.id, SyncAction.DELETE)
            Result.success(Unit)
        } catch (e: Exception) {
            runCatching { updateMessage(message.id) { it.deleted = true } }
            runCatching { enqueueRetry(message.id, SyncAction.DELETE) }
            Result.success(Unit)
        }
    }

    override suspend fun deleteMessageForMe(message: Message): Result<Unit> {
        return try {
            updateMessage(message.id) { it.deleted = true }
            val result = firestoreService.deleteMessageForMe(message)
            if (result.isFailure) enqueueRetry(message.id, SyncAction.DELETE_FOR_ME)
            Result.success(Unit)
        } catch (e: Exception) {
            runCatching { updateMessage(message.id) { it.deleted = true } }
            runCatching { enqueueRetry(message.id, SyncAction.DELETE_FOR_ME) }
            Result.success(Unit)
        }
    }

    override suspend fun markMessageAsRead(message: Message): Result<Unit> {
        return try {
            val result = firestoreService.markMessageAsRead(message)
            if (result.isSuccess) updateMessage(message.id) { it.isRead = true; it.status = MessageStatus.READ.name }
            result
        } catch (e: Exception) { Result.failure(e) }
    }

    override suspend fun observeRecentMessages(conversationId: String, limit: Long): Flow<Result<Unit>> = flow {
        try {
            firestoreService.getRecentMessagesStream(conversationId, limit, cutoffFor(conversationId)).collect { messages ->
                putPreservingReadState(conversationId, messages)
                emit(Result.success(Unit))
            }
        } catch (e: Exception) { emit(Result.failure(e)) }
    }.flowOn(Dispatchers.IO)

    override suspend fun loadOlderMessages(conversationId: String, limit: Long): Result<Int> {
        val oldestId = messageBox.query(ObxMessage_.conversationId.equal(conversationId))
            .order(ObxMessage_.timestamp).build().use { it.findFirst()?.uid } ?: return Result.success(0)
        return firestoreService.fetchOlderMessages(conversationId, oldestId, limit, cutoffFor(conversationId)).map { older ->
            putPreservingReadState(conversationId, older); older.size
        }
    }

    private fun updateMessage(id: String, mutate: (ObxMessage) -> Unit) {
        val existing = messageBox.query(ObxMessage_.uid.equal(id)).build().use { it.findFirst() } ?: return
        mutate(existing); messageBox.put(existing)
    }

    private fun putPreservingReadState(conversationId: String, messages: List<Message>) {
        if (messages.isEmpty()) return
        val localByUid = messageBox.query(ObxMessage_.conversationId.equal(conversationId))
            .build().use { it.find() }.associateBy { it.uid }
        val merged = messages.map { domain ->
            domain.toObx().also { incoming ->
                localByUid[incoming.uid]?.let { local ->
                    incoming.boxId = local.boxId
                    if (local.isRead) incoming.isRead = true
                    if (local.status == MessageStatus.READ.name) incoming.status = MessageStatus.READ.name
                    if (local.deleted) incoming.deleted = true
                }
            }
        }
        messageBox.put(merged)
    }
    // updateConversationPreview / previewLabel / withStatus omitted for brevity - they build the
    // conversation list preview text from the message (see the main note's previewLabel).
}
```

The heart of chat: `getMessagesStream` is the cold ObjectBox stream (`asFlow().map.flowOn(Default)`);
`sendMessage` writes locally first then attempts Firestore, flipping to `FAILED` + enqueuing a
retry on error; the two delete methods implement the for-everyone (`deleted` field, sender-only
guard) vs for-me (`deletedFor` array) rules; `putPreservingReadState` merges remote refreshes
without clobbering local read/deleted state. See the flow analysis in [[Async and Reactivity]].

---

## 5. MediaRepositoryImpl

`data/repository/MediaRepositoryImpl.kt` - `@Singleton` owning its own app-scoped coroutine
scope so uploads survive leaving the chat. The full file is in the repo; the load-bearing parts:

```kotlin
@Singleton
class MediaRepositoryImpl @Inject constructor(
    @param:ApplicationContext private val context: Context,
    private val messageBox: Box<ObxMessage>,
    private val conversationBox: Box<ObxConversation>,
    private val syncQueueBox: Box<ObxSyncQueueItem>,
    private val firestoreService: FirestoreService,
    private val manager: MediaTransferManager,
    private val mediaCache: MediaCache,
    private val mediaMetadata: MediaMetadata,
    private val authService: FirebaseAuthService,
    private val codec: E2eeMessageCodec,
) : IMediaRepository {

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private val cancelledItems = ConcurrentHashMap.newKeySet<String>()

    override val transfers: StateFlow<Map<String, MediaTransfer>> = manager.transfers

    override fun sendMediaMessage(conversationId: String, uris: List<String>, caption: String) {
        scope.launch {
            val message = buildLocalMessage(conversationId, uris.take(MAX_GROUP), caption)
            if (message.items.isEmpty()) return@launch
            messageBox.put(message.toObx())
            updateConversationPreview(message)
            uploadAndSend(message.id)
        }
    }

    private suspend fun uploadAndSend(messageId: String) {
        val original = loadMessage(messageId) as? Message.Media ?: return
        val encryptActive = codec.isEncryptable(original.conversationId)
        coroutineScope {
            val jobs = original.items.filterNot { cancelledItems.contains(it.id) }.map { item ->
                launch {
                    val file = mediaCache.fileFor(item.id, item.kind)
                    val path = storagePath(original.conversationId, messageId, item)
                    val fileKey = if (encryptActive) MediaCipher.generateFileKey() else null
                    val uploadFile = if (fileKey != null) File(context.cacheDir, "${item.id}.enc").also {
                        MediaCipher.encrypt(fileKey, file, it) } else file
                    val result = manager.upload(item.id, uploadFile, path)
                    if (fileKey != null) uploadFile.delete()
                    if (result.isSuccess) {
                        val url = result.getOrThrow()
                        val current = loadMessage(messageId) as? Message.Media
                        putMedia(messageId, current?.items.orEmpty().map {
                            if (it.id == item.id) it.copy(url = url, storagePath = path, fileKey = fileKey?.let(::b64)) else it
                        })
                        manager.clear(item.id)
                    }
                }
            }
            val completed = withTimeoutOrNull(SEND_TIMEOUT_MS) { jobs.joinAll(); true } ?: false
            if (!completed) jobs.forEach { it.cancel() }
        }
        val finalMessage = loadMessage(messageId) as? Message.Media ?: return
        val succeeded = finalMessage.items.filter { it.url.isNotBlank() }
        val failed = finalMessage.items.filter { it.url.isBlank() }
        when {
            succeeded.isEmpty() -> markFailed(messageId)
            failed.isEmpty() -> sendOrFail(finalMessage.copy(status = MessageStatus.SENT))
            else -> {
                val sentPart = finalMessage.copy(items = succeeded, status = MessageStatus.SENT)
                messageBox.put(sentPart.toObx()); sendOrFail(sentPart); splitFailed(finalMessage, failed)
            }
        }
    }
    // downloadMedia, cancelUpload, retry, buildLocalMessage, splitFailed, etc. as in the repo.
    private companion object { const val TAG = "MediaRepositoryImpl"; const val MAX_GROUP = 15; const val SEND_TIMEOUT_MS = 35_000L }
}
```

`scope` is `SupervisorJob() + Dispatchers.IO` and is never cancelled (app-scoped), because an
upload started from a chat screen must outlive it. `uploadAndSend` uploads items as parallel
`launch`es inside a `coroutineScope`, bounded by `withTimeoutOrNull(35s)`, then partitions
succeeded/failed and splits failures into their own bubble. When encryptable, each file is
`MediaCipher.encrypt`-ed and the per-file key is stored in the message payload (see
[[End-to-End Encryption]]). Scope rationale in [[Async and Reactivity]].

---

## 6. FirestoreService message methods

`data/remote/firebase/FirestoreService.kt` - the wire side, with the delete rules and the
optimistic-send listener:

```kotlin
suspend fun sendMessage(message: Message): Result<Unit> {
    return try {
        val dto = codec.encode(message).let {
            if (message.status == MessageStatus.SENDING) it.copy(status = MessageStatus.SENT.name) else it
        }
        val conversationRef = conversationsCollection.document(message.conversationId)
        conversationRef.collection("messages").document(dto.id).set(dto)
        conversationRef.update(mapOf(
            "lastMessage" to if (dto.enc == 1) encryptedPreview(message) else previewLabel(message),
            "lastMessageSenderId" to message.senderId,
            "lastMessageTimestamp" to message.timestamp))
        Result.success(Unit)
    } catch (e: Exception) { Result.failure(e) }
}

suspend fun deleteMessage(message: Message): Result<Unit> {           // delete for everyone
    return try {
        conversationsCollection.document(message.conversationId)
            .collection("messages").document(message.id)
            .update("deleted", true).await()
        Result.success(Unit)
    } catch (e: Exception) { Result.failure(e) }
}

suspend fun deleteMessageForMe(message: Message): Result<Unit> {      // delete for me
    return try {
        val uid = authService.getCurrentUserId() ?: return Result.failure(Exception("Not signed in"))
        conversationsCollection.document(message.conversationId)
            .collection("messages").document(message.id)
            .update("deletedFor", FieldValue.arrayUnion(uid)).await()
        Result.success(Unit)
    } catch (e: Exception) { Result.failure(e) }
}

fun getRecentMessagesStream(conversationId: String, limit: Long = 100, cutoff: Long = 0): Flow<List<Message>> {
    val raw: Flow<List<Pair<RemoteMessageDto, Boolean>>> = callbackFlow {
        val messagesRef = conversationsCollection.document(conversationId)
            .collection("messages").whereGreaterThan("timestamp", cutoff)
            .orderBy("timestamp", Query.Direction.DESCENDING).limit(limit)
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
    }
    return raw.map { rows -> rows.map { (dto, pending) -> applyDeletedFor(dto, codec.decode(dto, pending)) } }
}
```

`sendMessage` runs the message through the E2EE codec (see [[End-to-End Encryption]]) and writes
both the message doc and the conversation preview. `deleteMessage` sets `deleted=true` (everyone);
`deleteMessageForMe` array-unions the uid into `deletedFor`. `getRecentMessagesStream` is the
`callbackFlow` snapshot listener with `MetadataChanges.INCLUDE` + `hasPendingWrites()` for
optimistic rendering, then decodes/decrypts each row (analysis in [[Async and Reactivity]]).

---

## 7. ObxFlow

The ObjectBox-to-Flow bridge every local stream above depends on:

```kotlin
fun <T> Query<T>.asFlow(): Flow<List<T>> = callbackFlow {
    val query = this@asFlow
    val subscription = subscribe().observer { data -> trySend(data) }
    awaitClose { subscription.cancel(); query.close() }
}
```

Why the `val query = this@asFlow` capture and both cleanups matter is dissected in
[[Async and Reactivity]].

---

## Related

[[Chat and Messaging]] - [[Conversations List - Code]] - [[Data Layer]] -
[[Async and Reactivity]] - [[End-to-End Encryption]] - [[Notifications]] -
[[Presentation Layer (MVI)]]
