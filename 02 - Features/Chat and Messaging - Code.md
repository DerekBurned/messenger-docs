# Chat and Messaging - Code

Companion to [[Chat and Messaging]]. The full source for the chat feature, block by block. This
is the largest feature; snippets are copied verbatim from the repo. Conceptual "why" lives in the
main note.

Files covered:
- MVI: `presentation/state/chat/ChatUiState.kt` (with `PendingAttachment.kt` beside it),
  `presentation/intent/ChatIntent.kt`, `presentation/effect/ChatEffect.kt`
- `presentation/viewmodel/ChatViewModel.kt`
- Use cases `domain/usecase/message/{GetMessageUseCase, SendMessageUseCase}.kt` (others noted)
- `data/repository/MessageRepositoryImpl.kt`
- `data/repository/MediaRepositoryImpl.kt`
- `data/remote/firebase/FirestoreService.kt` (message methods)
- `data/local/obx/ObxFlow.kt`

---

## 1. MVI contract

`PendingAttachment.kt` and `ChatUiState.kt` (blank lines between fields dropped; the voice
fields are covered in [[Voice Messages - Code]]):

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
    val voiceDraft: VoiceRecording? = null,
    val micDenied: Boolean = false,
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
    data class MicPermissionDenied(val permanently: Boolean) : ChatIntent
    data object DismissMicPrompt : ChatIntent
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

`presentation/viewmodel/ChatViewModel.kt` - the full MVI orchestration:

```kotlin
@HiltViewModel
class ChatViewModel @Inject constructor(
    private val sendMessageUseCase: SendMessageUseCase,
    private val getMessagesUseCase: GetMessagesUseCase,
    private val syncMessagesUseCase: SyncMessagesUseCase,
    private val loadOlderMessagesUseCase: LoadOlderMessagesUseCase,
    private val markMessageAsReadUseCase: MarkMessageAsReadUseCase,
    private val markConversationAsReadUseCase: MarkConversationAsReadUseCase,
    private val deleteMessageUseCase: DeleteMessageUseCase,
    private val observeUserPresenceUseCase: ObserveUserPresenceUseCase,
    private val observeTypingUseCase: ObserveTypingUseCase,
    private val typingService: ITypingService,
    private val mediaRepository: IMediaRepository,
    private val userRepository: IUserRepository,
    firebaseAuthService: FirebaseAuthService,
    savedStateHandle: SavedStateHandle,
) : MviViewModel<ChatUiState, ChatIntent, ChatEffect>(initialState = ChatUiState()) {

    val conversationId: String = savedStateHandle.get<String>("conversationId") ?: ""
    private val partnerId: String = savedStateHandle.get<String>("partnerId") ?: ""
    private val partnerName: String = savedStateHandle.get<String>("partnerName") ?: ""
    private val seededAvatarUrl: String? =
        savedStateHandle.get<String>("partnerAvatarUrl")?.takeIf { it.isNotBlank() }

    private var typingJob: Job? = null
    private var typingClearJob: Job? = null
    private var highlightClearJob: Job? = null
    private val markedAsReadIds = mutableSetOf<String>()

    private companion object {
        const val TAG = "ChatViewModel"
        const val PAGE_SIZE = 100L
        const val TYPING_DEBOUNCE_MS = 500L
        const val TYPING_TTL_MS = 5000L
        const val LAST_SEEN_TICK_MS = 60_000L
        const val HIGHLIGHT_MS = 2500L
        const val MAX_JUMP_PAGES = 50
        const val JUMP_PAGE_TIMEOUT_MS = 2000L
    }

    init {
        val currentUserId = firebaseAuthService.getCurrentUserId().orEmpty()
        setState {
            copy(
                currentUserId = currentUserId,
                partnerUsername = resolveDisplayName(partnerName, null),
                partnerAvatarUrl = seededAvatarUrl,
                partnerLastSeenDisplay = DateUtils.formatLastSeen(partnerPresence.lastSeen),
            )
        }
        observePartnerAlias()
        observePartnerAvatar()
        observeTransfers()
        if (conversationId.isNotBlank()) {
            loadMessages()
            syncMessages()
            markConversationRead()
            if (partnerId.isNotBlank()) observePartnerPresence()
            observeTyping()
        }
    }

    private fun markConversationRead() {
        viewModelScope.launch { markConversationAsReadUseCase(conversationId) }
    }

    private fun observePartnerAlias() {
        if (partnerId.isBlank()) return
        userRepository.observeContactAliases()
            .onEach { aliases -> setState { copy(partnerUsername = resolveDisplayName(partnerName, aliases[partnerId])) } }
            .launchIn(viewModelScope)
    }

    private fun observePartnerAvatar() {
        if (partnerId.isBlank()) return
        userRepository.observeUser(partnerId)
            .onEach { user -> setState { copy(partnerAvatarUrl = user?.avatarUrl?.takeIf { it.isNotBlank() } ?: partnerAvatarUrl) } }
            .launchIn(viewModelScope)
        viewModelScope.launch { userRepository.getUserById(partnerId) }
    }

    override fun handleIntent(intent: ChatIntent) {
        when (intent) {
            is ChatIntent.TextChanged -> onTextChanged(intent.text)
            is ChatIntent.SendMessage -> sendMessage(intent.text)
            is ChatIntent.MessagesSeen -> onMessagesSeen(intent.messages)
            ChatIntent.LoadOlderMessages -> loadOlder()
            is ChatIntent.DeleteMessage -> deleteMessage(intent.message, intent.forEveryone)
            is ChatIntent.SetReplyTo -> setState { copy(replyingTo = intent.message) }
            ChatIntent.ClearReply -> setState { copy(replyingTo = null) }
            is ChatIntent.JumpToMessage -> jumpToMessage(intent.messageId)
            is ChatIntent.Forward -> setState { copy(forwardingMessage = intent.message) }
            ChatIntent.ClearForward -> setState { copy(forwardingMessage = null) }
            ChatIntent.ClearError -> setState { copy(error = null) }
            is ChatIntent.DownloadMedia -> mediaRepository.downloadMedia(intent.item)
            is ChatIntent.CancelUpload -> mediaRepository.cancelUpload(intent.messageId, intent.itemId)
            is ChatIntent.CancelDownload -> mediaRepository.cancelDownload(intent.itemId)
            is ChatIntent.ToggleAttachment -> toggleAttachment(intent.uri, intent.kind)
            is ChatIntent.AddAttachment -> addAttachment(intent.uri, intent.kind)
            ChatIntent.ClearAttachments -> setState { copy(pendingAttachments = emptyList()) }
            is ChatIntent.RetryMedia -> mediaRepository.retry(intent.messageId)
        }
    }

    private fun loadOlder() {
        if (currentState.isLoadingOlder || !currentState.hasMoreOlder) return
        viewModelScope.launch {
            setState { copy(isLoadingOlder = true) }
            loadOlderPage()
            setState { copy(isLoadingOlder = false) }
        }
    }

    private suspend fun loadOlderPage(): Int {
        val result = loadOlderMessagesUseCase(conversationId)
        val fetched = result.getOrDefault(0)
        setState { copy(hasMoreOlder = if (result.isSuccess) fetched >= PAGE_SIZE else hasMoreOlder) }
        return fetched
    }

    private fun jumpToMessage(messageId: String) {
        viewModelScope.launch {
            var guard = 0
            while (currentState.messages.none { it.id == messageId } &&
                currentState.hasMoreOlder && guard < MAX_JUMP_PAGES) {
                guard++
                val sizeBefore = currentState.messages.size
                loadOlderPage()
                withTimeoutOrNull(JUMP_PAGE_TIMEOUT_MS) {
                    state.first { s -> s.messages.size > sizeBefore || s.messages.any { it.id == messageId } }
                }
                if (currentState.messages.size == sizeBefore) break
            }
            if (currentState.messages.any { it.id == messageId }) {
                setState { copy(highlightedMessageId = messageId) }
                highlightClearJob?.cancel()
                highlightClearJob = viewModelScope.launch {
                    delay(HIGHLIGHT_MS)
                    setState { copy(highlightedMessageId = null) }
                }
            } else {
                emitEffect(ChatEffect.ShowError("Original message not found".toUiText()))
            }
        }
    }

    private fun syncMessages() {
        viewModelScope.launch {
            try { syncMessagesUseCase(conversationId).collect { } }
            catch (e: Exception) { Log.w(TAG, "Message sync stream errored", e) }
        }
    }

    private fun loadMessages() {
        viewModelScope.launch {
            getMessagesUseCase(conversationId).collect { resource ->
                when (resource) {
                    is Resource.Loading -> setState { copy(isLoading = true) }
                    is Resource.Success -> {
                        setState {
                            val resolvedAnchor = if (!unreadAnchorResolved && currentUserId.isNotBlank() && resource.data.isNotEmpty()) {
                                val anchorId = resource.data.firstOrNull {
                                    it.senderId != currentUserId && it.status != MessageStatus.READ
                                }?.id
                                copy(firstUnreadMessageId = anchorId, unreadAnchorResolved = true)
                            } else this
                            val media = resource.data.filterIsInstance<Message.Media>().flatMap { it.items }
                            resolvedAnchor.copy(isLoading = false, messages = resource.data,
                                conversationMedia = media, error = null)
                        }
                    }
                    is Resource.Error -> setState { copy(isLoading = false, error = resource.message.toUiText()) }
                    is Resource.Failure -> setState { copy(isLoading = false, error = resource.exception.message?.toUiText()) }
                }
            }
        }
    }

    private fun onMessagesSeen(messages: List<Message>) {
        val me = currentState.currentUserId
        if (me.isBlank()) return
        var newlyRead = false
        messages.forEach { message ->
            if (markedAsReadIds.add(message.id)) { markAsRead(message); newlyRead = true }
        }
        if (newlyRead) markConversationRead()
    }

    private fun observePartnerPresence() {
        viewModelScope.launch {
            val presence = observeUserPresenceUseCase(partnerId).catch { e -> Log.w(TAG, "Presence observer errored", e) }
            combine(presence, tickerFlow(LAST_SEEN_TICK_MS)) { p, _ -> p }
                .collect { p ->
                    setState { copy(partnerPresence = p, partnerLastSeenDisplay = DateUtils.formatLastSeen(p.lastSeen)) }
                }
        }
    }

    private fun tickerFlow(periodMs: Long) = flow {
        emit(Unit)
        while (true) { delay(periodMs); emit(Unit) }
    }

    private fun observeTransfers() {
        mediaRepository.transfers.onEach { setState { copy(transfers = it) } }.launchIn(viewModelScope)
    }

    private fun observeTyping() {
        viewModelScope.launch {
            observeTypingUseCase(conversationId)
                .catch { e -> Log.w(TAG, "Typing observer errored", e) }
                .collect { typingUserIds ->
                    setState { copy(typingUsernames = typingUserIds.toList(), isPartnerTyping = typingUserIds.isNotEmpty()) }
                }
        }
    }

    private fun onTextChanged(text: String) {
        typingJob?.cancel()
        typingClearJob?.cancel()
        if (text.isBlank()) {
            viewModelScope.launch { try { typingService.clearTyping(conversationId) } catch (e: Exception) { Log.w(TAG, "Typing clear failed", e) } }
            return
        }
        typingJob = viewModelScope.launch {
            delay(TYPING_DEBOUNCE_MS)
            try { typingService.setTyping(conversationId) } catch (e: Exception) { Log.w(TAG, "Typing set failed", e) }
        }
        typingClearJob = viewModelScope.launch {
            delay(TYPING_TTL_MS)
            try { typingService.clearTyping(conversationId) } catch (e: Exception) { Log.w(TAG, "Typing clear (TTL) failed", e) }
        }
    }

    private fun toggleAttachment(uri: Uri, kind: String) {
        setState {
            val existing = pendingAttachments.firstOrNull { it.uri == uri }
            val updated = if (existing != null) pendingAttachments - existing
                          else pendingAttachments + PendingAttachment(uri, kind)
            copy(pendingAttachments = updated)
        }
    }

    private fun addAttachment(uri: Uri, kind: String) {
        setState {
            if (pendingAttachments.any { it.uri == uri }) return@setState this
            copy(pendingAttachments = pendingAttachments + PendingAttachment(uri, kind))
        }
    }

    private fun sendMessage(text: String) {
        val attachments = currentState.pendingAttachments
        if (attachments.isNotEmpty()) { sendMediaMessage(text, attachments); return }
        val replyTo = currentState.replyingTo
        viewModelScope.launch { typingService.clearTyping(conversationId) }
        viewModelScope.launch {
            setState { copy(isSending = true, replyingTo = null) }
            sendMessageUseCase(conversationId, text, replyTo).collect { resource ->
                when (resource) {
                    is Resource.Loading -> {}
                    is Resource.Success -> setState { copy(isSending = false, error = null) }
                    is Resource.Error -> setState { copy(isSending = false, error = resource.message.toUiText()) }
                    is Resource.Failure -> setState { copy(isSending = false, error = resource.exception.message?.toUiText()) }
                }
            }
        }
    }

    private fun sendMediaMessage(caption: String, attachments: List<PendingAttachment>) {
        viewModelScope.launch { typingService.clearTyping(conversationId) }
        mediaRepository.sendMediaMessage(
            conversationId = conversationId,
            uris = attachments.map { it.uri.toString() },
            caption = caption.trim(),
        )
        setState { copy(pendingAttachments = emptyList(), replyingTo = null) }
    }

    private fun markAsRead(message: Message) {
        viewModelScope.launch { markMessageAsReadUseCase(message) }
    }

    private fun deleteMessage(message: Message, forEveryone: Boolean) {
        viewModelScope.launch {
            deleteMessageUseCase(message, forEveryone).collect { resource ->
                when (resource) {
                    is Resource.Error -> emitEffect(ChatEffect.ShowError(resource.message.toUiText()))
                    is Resource.Failure -> emitEffect(ChatEffect.ShowError(
                        resource.exception.message?.toUiText() ?: UiText.StringResource(R.string.chat_error_delete_failed)))
                    else -> {}
                }
            }
        }
    }

    override fun onCleared() {
        super.onCleared()
        typingJob?.cancel()
        typingClearJob?.cancel()
    }
}
```

Notable mechanics (full "why" in [[Chat and Messaging]]):

- `init` reads nav args from `SavedStateHandle`, seeds derived display state, then wires all the
  streams (messages, sync, presence, typing, transfers, alias/avatar). Everything runs on
  `viewModelScope`.
- `loadMessages` collects the `Resource`-wrapped ObjectBox stream; on first non-empty success it
  resolves the unread anchor once and flattens `Message.Media` items into `conversationMedia`.
- `observePartnerPresence` `combine`s presence with a 60s `tickerFlow` so "last seen" stays fresh.
- `onTextChanged` debounces typing (500ms) with a 5s TTL clear.
- `jumpToMessage` pages backward with a per-page `withTimeoutOrNull`, then highlights for 2.5s.
- `onCleared` cancels the manual typing jobs.

---

## 3. Message use cases

`GetMessageUseCase.kt` (class `GetMessagesUseCase`):

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
