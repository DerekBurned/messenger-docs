# Presence and Typing - Code

Companion to [[Presence and Typing]]. The full source for presence and typing, block by block.
Snippets are verbatim from the repo.

Files covered:
- `data/remote/firebase/RealtimeDbService.kt`
- `data/presence/FirebasePresenceService.kt`, `data/service/FirebaseTypingService.kt`
- `data/presence/PresenceManager.kt`, `data/presence/AppLifecycleObserver.kt`
- `domain/usecase/presence/ObserveUserPresenceUseCase.kt`,
  `domain/usecase/typing/ObserveTypingUseCase.kt`

---

## 1. RealtimeDbService

`data/remote/firebase/RealtimeDbService.kt` in full - the raw RTDB reads/writes:

```kotlin
@Singleton
class RealtimeDbService @Inject constructor(
    private val database: FirebaseDatabase
) {
    private val presenceRef = database.getReference("presence")
    private val typingRef = database.getReference("typing")
    private val connectedRef = database.getReference(".info/connected")

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

    suspend fun setPresence(userId: String, state: String, lastSeen: Long) {
        val data = mapOf("state" to state, "lastSeen" to lastSeen, "lastChanged" to ServerValue.TIMESTAMP)
        presenceRef.child(userId).setValue(data).await()
    }

    fun setupOnDisconnect(userId: String) {
        val offlineData = mapOf("state" to "offline", "lastSeen" to ServerValue.TIMESTAMP,
            "lastChanged" to ServerValue.TIMESTAMP)
        presenceRef.child(userId).onDisconnect().setValue(offlineData)
    }

    fun cancelOnDisconnect(userId: String) { presenceRef.child(userId).onDisconnect().cancel() }

    fun observePresence(userId: String): Flow<Map<String, Any?>> = callbackFlow {
        val ref = presenceRef.child(userId)
        val listener = object : ValueEventListener {
            override fun onDataChange(snapshot: DataSnapshot) {
                trySend(snapshot.value as? Map<String, Any?> ?: emptyMap())
            }
            override fun onCancelled(error: DatabaseError) { close(error.toException()) }
        }
        ref.addValueEventListener(listener)
        awaitClose { ref.removeEventListener(listener) }
    }

    fun removePresence(userId: String) { presenceRef.child(userId).removeValue() }

    suspend fun setTyping(conversationId: String, userId: String) {
        val ref = typingRef.child(conversationId).child(userId)
        ref.onDisconnect().removeValue()
        ref.setValue(mapOf("timestamp" to ServerValue.TIMESTAMP)).await()
    }

    suspend fun clearTyping(conversationId: String, userId: String) {
        val ref = typingRef.child(conversationId).child(userId)
        ref.onDisconnect().cancel()
        ref.removeValue().await()
    }

    fun clearTypingNow(conversationId: String, userId: String) {
        val ref = typingRef.child(conversationId).child(userId)
        ref.onDisconnect().cancel()
        ref.removeValue()
    }

    fun observeTyping(conversationId: String): Flow<Map<String, Any?>> = callbackFlow {
        val ref = typingRef.child(conversationId)
        val listener = object : ValueEventListener {
            override fun onDataChange(snapshot: DataSnapshot) {
                val typingUsers = mutableMapOf<String, Any?>()
                for (child in snapshot.children) child.key?.let { typingUsers[it] = child.value }
                trySend(typingUsers)
            }
            override fun onCancelled(error: DatabaseError) { close(error.toException()) }
        }
        ref.addValueEventListener(listener)
        awaitClose { ref.removeEventListener(listener) }
    }
}
```

Three `callbackFlow` + `ValueEventListener` reads (connection, presence, typing) that each remove
their listener on `awaitClose`. `setupOnDisconnect` is the reliability trick: the server writes the
offline record when the socket drops, so a crash still marks the user offline. Typing uses the
same trick - `setTyping` arms `onDisconnect().removeValue()` before writing, so a killed app cannot
leave a stuck typing node; `clearTyping` cancels that handler, and `clearTypingNow` is a
fire-and-forget variant (no `await`) for `ChatViewModel.onCleared`, where `viewModelScope` is
already gone. Data lives under `presence/{uid}` and `typing/{conversationId}/{uid}`.

---

## 2. Service adapters

`FirebasePresenceService.kt` - maps raw RTDB maps to the domain `UserPresence` and combines many:

```kotlin
@Singleton
class FirebasePresenceService @Inject constructor(
    private val realtimeDbService: RealtimeDbService,
    private val firebaseAuth: FirebaseAuth
) : IPresenceService {

    override fun observePresence(userId: String): Flow<UserPresence> =
        realtimeDbService.observePresence(userId).map { data ->
            UserPresence(
                state = PresenceState.fromString(data["state"] as? String),
                lastSeen = (data["lastSeen"] as? Number)?.toLong() ?: 0L
            )
        }

    override fun observeMultiplePresence(userIds: List<String>): Flow<Map<String, UserPresence>> {
        if (userIds.isEmpty()) return flowOf(emptyMap())
        val flows = userIds.map { userId -> observePresence(userId).map { presence -> userId to presence } }
        return combine(flows) { results -> results.toMap() }
    }

    override suspend fun setMyPresence(state: PresenceState): Boolean {
        val userId = firebaseAuth.currentUser?.uid ?: return false
        realtimeDbService.setPresence(userId, state.name.lowercase(), System.currentTimeMillis())
        return true
    }

    override suspend fun setupOnDisconnect() {
        val userId = firebaseAuth.currentUser?.uid ?: return
        realtimeDbService.setupOnDisconnect(userId)
    }

    override suspend fun removePresence() {
        val userId = firebaseAuth.currentUser?.uid ?: return
        realtimeDbService.cancelOnDisconnect(userId)
        realtimeDbService.removePresence(userId)
    }
}
```

`observeMultiplePresence` is the `combine` of per-user flows into one `Map` used by the
conversation list ([[Conversations List]]).

`FirebaseTypingService.kt` - filters out the current user:

```kotlin
@Singleton
class FirebaseTypingService @Inject constructor(
    private val realtimeDbService: RealtimeDbService,
    private val firebaseAuth: FirebaseAuth
) : ITypingService {

    override fun observeTyping(conversationId: String): Flow<Set<String>> =
        realtimeDbService.observeTyping(conversationId).map { data ->
            val currentUserId = firebaseAuth.currentUser?.uid
            val now = System.currentTimeMillis()
            data.entries
                .filter { it.key != currentUserId && isFresh(it.value, now) }
                .map { it.key }
                .toSet()
        }

    override suspend fun setTyping(conversationId: String) {
        val userId = firebaseAuth.currentUser?.uid ?: return
        realtimeDbService.setTyping(conversationId, userId)
    }

    override suspend fun clearTyping(conversationId: String) {
        val userId = firebaseAuth.currentUser?.uid ?: return
        realtimeDbService.clearTyping(conversationId, userId)
    }

    override fun clearTypingNow(conversationId: String) {
        val userId = firebaseAuth.currentUser?.uid ?: return
        realtimeDbService.clearTypingNow(conversationId, userId)
    }

    private fun isFresh(value: Any?, now: Long): Boolean {
        val timestamp = (value as? Map<*, *>)?.get("timestamp") as? Number ?: return true
        return now - timestamp.toLong() <= TYPING_STALE_MS
    }

    private companion object { const val TYPING_STALE_MS = 6_000L }
}
```

`observeTyping` filters stale nodes by their `timestamp` (6s window, twice the sender's 3s heartbeat) in
addition to dropping the current user, so a stuck entry self-heals on the reader even without a
disconnect.

---

## 3. PresenceManager and AppLifecycleObserver

`data/presence/PresenceManager.kt` - turns lifecycle transitions into RTDB writes through a
conflated channel:

```kotlin
@Singleton
class PresenceManager @Inject constructor(
    private val presenceService: IPresenceService
) {
    companion object {
        private const val TAG = "PresenceManager"
        private const val AWAY_TO_OFFLINE_DELAY_MS = 5 * 60 * 1000L
    }

    private val transitions = Channel<PresenceState>(Channel.CONFLATED)
    private var collector: Job? = null
    private var offlineTimerJob: Job? = null
    private var lastAppliedState: PresenceState? = null

    fun goOnline(scope: CoroutineScope) {
        ensureCollector(scope)
        offlineTimerJob?.cancel(); offlineTimerJob = null
        transitions.trySend(PresenceState.ONLINE)
    }

    fun goAway(scope: CoroutineScope) {
        ensureCollector(scope)
        transitions.trySend(PresenceState.AWAY)
        offlineTimerJob?.cancel()
        offlineTimerJob = scope.launch {
            delay(AWAY_TO_OFFLINE_DELAY_MS)
            transitions.trySend(PresenceState.OFFLINE)
        }
    }

    fun goIdleAway(scope: CoroutineScope) {
        ensureCollector(scope)
        offlineTimerJob?.cancel(); offlineTimerJob = null
        transitions.trySend(PresenceState.AWAY)
    }

    fun goOffline(scope: CoroutineScope) {
        ensureCollector(scope)
        offlineTimerJob?.cancel(); offlineTimerJob = null
        transitions.trySend(PresenceState.OFFLINE)
    }

    fun disconnect(scope: CoroutineScope) {
        offlineTimerJob?.cancel(); offlineTimerJob = null
        lastAppliedState = null
        scope.launch {
            try { presenceService.removePresence() } catch (e: Exception) { Log.e(TAG, "Failed to remove presence", e) }
        }
    }

    private fun ensureCollector(scope: CoroutineScope) {
        if (collector?.isActive == true) return
        collector = scope.launch {
            transitions.consumeAsFlow().collect { state ->
                if (state == lastAppliedState) return@collect
                try {
                    val applied = presenceService.setMyPresence(state)
                    if (applied) {
                        if (state == PresenceState.ONLINE) presenceService.setupOnDisconnect()
                        lastAppliedState = state
                    }
                } catch (e: Exception) { Log.e(TAG, "Failed to set $state presence", e) }
            }
        }
    }
}
```

`Channel.CONFLATED` collapses a rapid burst of transitions to the latest before it is applied
(avoids a storm of RTDB writes). `lastAppliedState` de-dupes no-op writes. The manager never owns
a scope; callers pass one in.

`data/presence/AppLifecycleObserver.kt` - the process-lifecycle bridge registered in
`MessengerApplication.onCreate`:

```kotlin
@Singleton
class AppLifecycleObserver @Inject constructor(
    private val presenceManager: PresenceManager
) : DefaultLifecycleObserver {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    override fun onStart(owner: LifecycleOwner) { presenceManager.goOnline(scope) }
    override fun onPause(owner: LifecycleOwner) { presenceManager.goAway(scope) }
    override fun onResume(owner: LifecycleOwner) { presenceManager.goOnline(scope) }
    override fun onStop(owner: LifecycleOwner) { presenceManager.goOffline(scope) }
}
```

Foreground -> `goOnline`, background -> `goOffline` (with `goAway` on pause). It owns the scope it
hands to the manager; that scope lives for the process (it is a `@Singleton`).

---

## 4. Use cases

```kotlin
class ObserveUserPresenceUseCase @Inject constructor(
    private val presenceService: IPresenceService
) {
    operator fun invoke(userId: String): Flow<UserPresence> = presenceService.observePresence(userId)
    fun observeMultiple(userIds: List<String>): Flow<Map<String, UserPresence>> =
        presenceService.observeMultiplePresence(userIds)
}

class ObserveTypingUseCase @Inject constructor(
    private val typingService: ITypingService
) {
    operator fun invoke(conversationId: String): Flow<Set<String>> = typingService.observeTyping(conversationId)
}
```

The outgoing typing side (debounce + TTL) lives in `ChatViewModel.onTextChanged` - see
[[Chat and Messaging - Code]].

---

## Related

[[Presence and Typing]] - [[Conversations List - Code]] - [[Chat and Messaging - Code]] -
[[Authentication - Code]] - [[Async and Reactivity]] - [[Data Layer]]
