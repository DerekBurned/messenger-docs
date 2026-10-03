# Conversations List - Code

Companion to [[Conversations List]]. The actual source for the conversations list, block by
block. Snippets are copied from the repo; long expressions are sometimes joined onto one line.

Files covered:
- `presentation/state/ConversationsUiState.kt`
- `presentation/viewmodel/ChatsViewModel.kt` (formerly `ConversationsViewModel.kt`)
- Use cases in `domain/usecase/conversation/`
- `data/repository/ConversationRepositoryImpl.kt`
- `data/remote/firebase/FirestoreService.kt` (conversation methods; see [[Chat and Messaging - Code]]
  and [[Data Layer]] for the rest)

---

## 1. State

```kotlin
@Immutable
data class ConversationsUiState(
    val isLoading: Boolean = false,
    val conversations: List<Conversation> = emptyList(),
    val error: UiText? = null,
    val presenceMap: Map<String, UserPresence> = emptyMap(),
    val aliases: Map<String, String> = emptyMap(),
    val avatars: Map<String, String?> = emptyMap(),
    val participants: Map<String, User> = emptyMap(),
    val isRefreshing: Boolean = false,
    val currentUserId: String = "",
) : UiState
```

The row data is derived by joining `conversations` with four side maps: `presenceMap` (online
dots), `aliases` (locally renamed contacts), `avatars` (fetched profile photos) and
`participants` (the live cached user rows, which carry the current name and the `deleted` flag).
Keeping them as separate maps means a presence tick does not rebuild the conversation list
itself.

---

## 2. ChatsViewModel

`presentation/viewmodel/ChatsViewModel.kt` - the method-and-StateFlow style (plain `ViewModel`,
no MVI base). `deleteConversation` / `createConversation` expand their `fold` over several lines
in the source:

```kotlin
@HiltViewModel
class ChatsViewModel @Inject constructor(
    private val getConversationsUseCase: GetConversationsUseCase,
    private val deleteConversationUseCase: DeleteConversationUseCase,
    private val deleteConversationForEveryoneUseCase: DeleteConversationForEveryoneUseCase,
    private val clearConversationForMeUseCase: ClearConversationForMeUseCase,
    private val markConversationAsReadUseCase: MarkConversationAsReadUseCase,
    private val markConversationAsUnreadUseCase: MarkConversationAsUnreadUseCase,
    private val createConversationUseCase: CreateConversationUseCase,
    private val syncConversationsUseCase: SyncConversationsUseCase,
    private val observeUserPresenceUseCase: ObserveUserPresenceUseCase,
    private val observeAuthStateUseCase: ObserveAuthStateUseCase,
    private val userRepository: IUserRepository,
    private val firebaseAuth: FirebaseAuth
) : ViewModel() {

    private val _uiState = MutableStateFlow(
        ConversationsUiState(currentUserId = firebaseAuth.currentUser?.uid.orEmpty()),
    )
    val uiState: StateFlow<ConversationsUiState> = _uiState.asStateFlow()

    private var presenceJob: Job? = null
    private val fetchedUserIds = mutableSetOf<String>()

    init {
        loadConversations()
        observeAliases()
        observeParticipants()
    }

    private fun observeParticipants() {
        viewModelScope.launch {
            userRepository.observeCachedUsers().collect { users ->
                _uiState.update { it.copy(participants = users) }
            }
        }
    }

    private fun observeAliases() {
        viewModelScope.launch {
            runCatching { userRepository.refreshContactAliases() }
            userRepository.observeContactAliases().collect { aliases ->
                _uiState.update { it.copy(aliases = aliases) }
            }
        }
    }

    private fun loadConversations() {
        viewModelScope.launch {
            observeAuthStateUseCase()
                .flatMapLatest { user ->
                    if (user == null) {
                        flowOf(emptyList())
                    } else {
                        getConversationsUseCase()
                            .onStart { _uiState.update { it.copy(isLoading = true) } }
                    }
                }
                .catch { e -> _uiState.update { it.copy(isLoading = false, error = e.message?.toUiText()) } }
                .collect { conversations ->
                    _uiState.update { it.copy(isLoading = false, conversations = conversations, error = null) }
                    observeParticipantPresence(conversations)
                    loadParticipantAvatars(conversations)
                }
        }
    }

    private fun loadParticipantAvatars(conversations: List<Conversation>) {
        val currentUserId = firebaseAuth.currentUser?.uid ?: return
        val seeded = buildMap<String, String?> {
            conversations.forEach { conversation ->
                conversation.participantIds.forEachIndexed { index, id ->
                    if (id != currentUserId && this[id].isNullOrBlank()) {
                        put(id, conversation.participantAvatars.getOrNull(index))
                    }
                }
            }
        }
        if (seeded.isNotEmpty()) _uiState.update { it.copy(avatars = seeded + it.avatars) }

        val newUserIds = conversations.flatMap { it.participantIds }
            .filter { it != currentUserId }.distinct().filter { it !in fetchedUserIds }
        if (newUserIds.isEmpty()) return
        fetchedUserIds.addAll(newUserIds)
        viewModelScope.launch {
            newUserIds.forEach { id ->
                val result = runCatching {
                    userRepository.getUserById(id).getOrNull()?.avatarUrl
                }
                val url = result.getOrNull()
                if (url != null) {
                    _uiState.update { it.copy(avatars = it.avatars + (id to url)) }
                } else if (result.isFailure) {
                    fetchedUserIds.remove(id)
                }
            }
        }
    }

    private fun observeParticipantPresence(conversations: List<Conversation>) {
        presenceJob?.cancel()
        val currentUserId = firebaseAuth.currentUser?.uid ?: return
        val otherUserIds = conversations.flatMap { it.participantIds }
            .filter { it != currentUserId }.distinct()
        if (otherUserIds.isEmpty()) return
        presenceJob = viewModelScope.launch {
            observeUserPresenceUseCase.observeMultiple(otherUserIds)
                .catch { }
                .collect { presenceMap -> _uiState.update { it.copy(presenceMap = presenceMap) } }
        }
    }

    fun deleteConversation(conversationId: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            deleteConversationUseCase(conversationId).fold(
                onSuccess = { _uiState.update { it.copy(isLoading = false) } },
                onFailure = { e -> _uiState.update { it.copy(isLoading = false, error = e.message?.toUiText()) } })
        }
    }

    fun deleteForMe(conversationId: String) {
        viewModelScope.launch {
            clearConversationForMeUseCase(conversationId)
                .onFailure { e -> _uiState.update { it.copy(error = e.message?.toUiText()) } }
        }
    }

    fun deleteForEveryone(conversationId: String) {
        viewModelScope.launch {
            deleteConversationForEveryoneUseCase(conversationId)
                .onFailure { e -> _uiState.update { it.copy(error = e.message?.toUiText()) } }
        }
    }

    fun markAsRead(conversationId: String) {
        viewModelScope.launch {
            markConversationAsReadUseCase(conversationId)
                .onFailure { e -> _uiState.update { it.copy(error = e.message?.toUiText()) } }
        }
    }

    fun markAsUnread(conversationId: String) {
        viewModelScope.launch {
            markConversationAsUnreadUseCase(conversationId)
                .onFailure { e -> _uiState.update { it.copy(error = e.message?.toUiText()) } }
        }
    }

    fun createConversation(participantId: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            createConversationUseCase(participantId).fold(
                onSuccess = { _uiState.update { it.copy(isLoading = false) } },
                onFailure = { e -> _uiState.update { it.copy(isLoading = false, error = e.message?.toUiText()) } })
        }
    }

    fun refresh() {
        viewModelScope.launch {
            _uiState.update { it.copy(isRefreshing = true) }
            syncConversationsUseCase()
            _uiState.update { it.copy(isRefreshing = false) }
        }
    }

    fun clearError() { _uiState.update { it.copy(error = null) } }
}
```

Key mechanics:

- `loadConversations` collects the offline-first stream with `onStart` (spinner) and `catch`
  (degrade to error state, do not crash the collect). Each emission fans out to
  `observeParticipantPresence` and `loadParticipantAvatars`. The stream sits inside
  `flatMapLatest` over auth state: this ViewModel is Activity-scoped and outlives a logout, so
  signing out must cancel the Firestore listener (otherwise it would keep writing the old
  account's conversations into the freshly reset ObjectBox) and empty the list.
- `observeParticipants` keeps the live user rows in state; the list prefers them over the frozen
  `participantNames`/`participantAvatars` arrays, which is how renames and deleted accounts
  ("Unknown user") reach each row.
- `observeParticipantPresence` cancels `presenceJob` before resubscribing, so when the set of
  participants changes it does not leak an observer for the old set. `observeMultiple` is a
  `combine` of per-user presence flows (see [[Presence and Typing]]).
- `loadParticipantAvatars` seeds avatars already embedded in the conversation, then fetches the
  rest once each (tracked by `fetchedUserIds`; a failed fetch is removed so it is tried again).
- The three delete methods map one-to-one to the three use cases below.

---

## 3. Use cases

```kotlin
class GetConversationsUseCase @Inject constructor(
    private val conversationRepository: IConversationRepository
) {
    operator fun invoke(): Flow<List<Conversation>> = conversationRepository.getAllConversations()
}

class CreateConversationUseCase @Inject constructor(
    private val conversationRepository: IConversationRepository,
    private val getCurrentUserUseCase: GetCurrentUserUseCase
) {
    suspend operator fun invoke(participantId: String): Result<Conversation> {
        val currentUser = getCurrentUserUseCase() ?: return Result.failure(Exception("User not logged in"))
        if (participantId.isBlank()) return Result.failure(Exception("Invalid participant ID"))
        if (participantId == currentUser.id) return Result.failure(Exception("Cannot create conversation with yourself"))
        return conversationRepository.createConversation(listOf(currentUser.id, participantId))
    }
}

class ClearConversationForMeUseCase @Inject constructor(
    private val conversationRepository: IConversationRepository
) {
    suspend operator fun invoke(conversationId: String): Result<Unit> {
        if (conversationId.isBlank()) return Result.failure(Exception("Invalid conversation ID"))
        return conversationRepository.clearConversationForMe(conversationId)
    }
}

class DeleteConversationForEveryoneUseCase @Inject constructor(
    private val conversationRepository: IConversationRepository
) {
    suspend operator fun invoke(conversationId: String): Result<Unit> {
        if (conversationId.isBlank()) return Result.failure(Exception("Invalid conversation ID"))
        return conversationRepository.deleteConversationForEveryone(conversationId)
    }
}
```

`DeleteConversationUseCase` and `SyncConversationsUseCase` are the same thin shape (validate,
delegate to `deleteConversation` / `syncConversations`).

---

## 4. ConversationRepositoryImpl

`data/repository/ConversationRepositoryImpl.kt` in full - the offline-first merge lives here.

```kotlin
class ConversationRepositoryImpl @Inject constructor(
    private val firestoreService: FirestoreService,
    private val conversationBox: Box<ObxConversation>,
    private val messageBox: Box<ObxMessage>,
    private val authService: FirebaseAuthService,
) : IConversationRepository {

    override fun getAllConversations(): Flow<List<Conversation>> = channelFlow {
        val remoteJob = launch {
            val userId = authService.getCurrentUserId()
            if (userId.isNullOrBlank()) return@launch
            try {
                firestoreService.getAllConversations(userId).collect { sync ->
                    sync.conversations.forEach { upsert(it) }
                    if (!sync.fromCache && sync.removedIds.isNotEmpty()) prune(sync.removedIds)
                }
            } catch (e: Exception) { Log.w(TAG, "Remote conversation listener errored", e) }
        }
        val localJob = launch {
            conversationBox.query()
                .orderDesc(ObxConversation_.lastMessageTimestamp)
                .build()
                .asFlow()
                .map { rows -> rows.map { it.toDomain() } }
                .collect { send(it) }
        }
        awaitClose { remoteJob.cancel(); localJob.cancel() }
    }

    override suspend fun createConversation(participantIds: List<String>): Result<Conversation> {
        return try {
            val currentUserId = authService.getCurrentUserId()
                ?: return Result.failure(IllegalStateException("Cannot create conversation: no signed-in user"))
            val existing = firestoreService.findExistingConversation(participantIds, currentUserId)
                .getOrElse { return Result.failure(it) }
            if (existing != null) { upsert(existing); return Result.success(existing) }

            val names = mutableListOf<String>()
            val avatars = mutableListOf<String?>()
            for (id in participantIds) {
                val user = firestoreService.getUserProfile(id).getOrNull()
                names.add(user?.username ?: "Unknown")
                avatars.add(user?.avatarUrl)
            }
            val conversation = Conversation(
                id = "", participantIds = participantIds, participantNames = names,
                participantAvatars = avatars, lastMessage = null,
                lastMessageTimestamp = System.currentTimeMillis(),
            )
            val firestoreId = firestoreService.createConversation(conversation).getOrThrow()
            val saved = conversation.copy(id = firestoreId)
            upsert(saved)
            Result.success(saved)
        } catch (e: Exception) { Result.failure(e) }
    }

    override suspend fun deleteConversation(conversationId: String): Result<Unit> = try {
        conversationBox.query(ObxConversation_.uid.equal(conversationId)).build().use { it.remove() }
        Result.success(Unit)
    } catch (e: Exception) { Result.failure(e) }

    override suspend fun deleteConversationForEveryone(conversationId: String): Result<Unit> = try {
        firestoreService.deleteConversation(conversationId).getOrThrow()
        prune(listOf(conversationId))
        Result.success(Unit)
    } catch (e: Exception) { Result.failure(e) }

    override suspend fun clearConversationForMe(conversationId: String): Result<Unit> = try {
        val userId = authService.getCurrentUserId()
            ?: return Result.failure(IllegalStateException("Cannot clear conversation: no signed-in user"))
        val local = findByUid(conversationId)
            ?: return Result.failure(IllegalStateException("Conversation not found: $conversationId"))
        val cutoff = maxOf(local.lastMessageTimestamp, local.latestMessageTimestamp)
        firestoreService.clearConversationForMe(conversationId, userId, cutoff).getOrThrow()
        prune(listOf(conversationId))
        Result.success(Unit)
    } catch (e: Exception) { Result.failure(e) }

    override suspend fun markConversationAsRead(conversationId: String): Result<Unit> = try {
        val userId = authService.getCurrentUserId()
        findByUid(conversationId)?.let { conv -> conv.unreadCount = 0; conversationBox.put(conv) }
        if (!userId.isNullOrBlank()) {
            messageBox.query(
                ObxMessage_.conversationId.equal(conversationId)
                    .and(ObxMessage_.senderId.notEqual(userId))
                    .and(ObxMessage_.isRead.equal(false)),
            ).build().use { query ->
                val unread = query.find()
                if (unread.isNotEmpty()) { unread.forEach { it.isRead = true }; messageBox.put(unread) }
            }
            firestoreService.resetUnreadCount(conversationId, userId)
        }
        Result.success(Unit)
    } catch (e: Exception) { Result.failure(e) }

    override suspend fun syncConversations() {
        try {
            val userId = authService.getCurrentUserId() ?: return
            val remote = firestoreService.fetchAllConversationsOnce(userId).getOrNull() ?: return
            remote.forEach { upsert(it) }
            val staleIds = ConversationReconciler.staleIds(
                local = conversationBox.all.map { it.uid }.toSet(),
                remote = RemoteConversations.Server(remote.map { it.id }.toSet()),
            )
            prune(staleIds)
        } catch (e: Exception) { Log.w(TAG, "Conversation sync failed (SyncWorker will retry)", e) }
    }

    private fun findByUid(uid: String): ObxConversation? =
        conversationBox.query(ObxConversation_.uid.equal(uid)).build().use { it.findFirst() }

    private fun localUnread(conversationId: String, me: String): Int {
        if (me.isBlank()) return 0
        return messageBox.query(
            ObxMessage_.conversationId.equal(conversationId)
                .and(ObxMessage_.senderId.notEqual(me))
                .and(ObxMessage_.isRead.equal(false))
                .and(ObxMessage_.deleted.equal(false)),
        ).build().use { it.count().toInt() }
    }

    private fun prune(conversationIds: Collection<String>) {
        if (conversationIds.isEmpty()) return
        for (id in conversationIds) {
            conversationBox.query(ObxConversation_.uid.equal(id)).build().use { it.remove() }
            messageBox.query(ObxMessage_.conversationId.equal(id)).build().use { it.remove() }
        }
    }

    private fun upsert(conversation: Conversation) {
        val incoming = conversation.toObx()
        val currentUserId = authService.getCurrentUserId()
        val serverClearedAt = currentUserId?.let { conversation.clearedAt[it] } ?: 0L
        val me = currentUserId ?: ""
        val serverUnread = conversation.unreadCounts[me] ?: 0
        incoming.unreadCount = maxOf(serverUnread, localUnread(conversation.id, me))
        findByUid(conversation.id)?.let { existing ->
            incoming.boxId = existing.boxId
            incoming.latestMessageText = existing.latestMessageText
            incoming.latestMessageSenderId = existing.latestMessageSenderId
            incoming.latestMessageTimestamp = existing.latestMessageTimestamp
        }
        incoming.clearedAtForMe = serverClearedAt
        conversationBox.put(incoming)
    }

    private companion object { const val TAG = "ConversationRepo" }
}
```

Key mechanics:

- `getAllConversations` is a `channelFlow` (not `callbackFlow`) because it launches two child
  coroutines into the same producer scope: a remote job that collects the Firestore listener and
  writes rows into ObjectBox (`upsert`) plus prunes server-removed ids, and a local job that
  emits the ObjectBox `asFlow()` stream to the collector via `send`. `channelFlow` is required
  because emissions come from a different coroutine than the block itself; `awaitClose` cancels
  both jobs. The UI only ever sees the local stream, so it renders instantly and updates as the
  remote writes through. This is the concrete offline-first merge described in [[Data Layer]].
- `upsert` is where local and server state are reconciled per conversation: unread count is the
  max of the server counter and the locally-computed unread (`localUnread`), the local
  `latestMessage*` preview fields are preserved over the incoming row, and `clearedAtForMe` is
  taken from the server `clearedAt[uid]`. Preserving `boxId` makes it an update, not an insert.
- `prune` deletes a conversation and its messages locally; it is driven both by the live
  listener's `removedIds` and by `syncConversations` via `ConversationReconciler.staleIds`.
- `clearConversationForMe` computes the cutoff as the latest message timestamp and delegates to
  Firestore's `clearConversationForMe` (writes `clearedAt.{uid}` + removes uid from `visibleTo`),
  then prunes locally. This is the per-user delete described in [[Conversations List]].

The Firestore side (`findExistingConversation`, `createConversation`, `clearConversationForMe`,
`deleteConversation`, `getAllConversations`, `fetchAllConversationsOnce`, `resetUnreadCount`) is
listed in [[Chat and Messaging - Code]] / [[Data Layer]].

---

## Related

[[Conversations List]] - [[Chat and Messaging - Code]] - [[Presence and Typing]] -
[[Data Layer]] - [[Async and Reactivity]] - [[Presentation Layer (MVI)]]
