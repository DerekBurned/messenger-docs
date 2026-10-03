# Calls (Voice and Video) - Code

Companion to [[Calls (Voice and Video)]]. The source for the call feature. Two files are very
large - `CallForegroundService.kt` (~990 lines) and `AgoraCallService.kt` (~290 lines) - so those
are shown as their structurally essential parts with a note; every other file is verbatim and
complete. Full "why" is in the main note.

Files covered:
- `data/remote/call/ActiveCallHolder.kt`, `data/remote/call/CallForegroundService.kt`,
  `data/remote/call/AgoraCallService.kt`, `data/remote/call/FirebaseCallSignalingService.kt`
- `presentation/state/CallUiState.kt`, `presentation/viewmodel/CallViewModel.kt`
- `presentation/components/call/{ActiveCallBar, CallBarPresenter}.kt`
- `data/repository/CallHistoryRepository.kt`

---

## 1. ActiveCallHolder - the one source of truth

`data/remote/call/ActiveCallHolder.kt` in full. The single global `StateFlow`; only the service
writes (`internal`), everything else reads.

```kotlin
object ActiveCallHolder {

    data class ActiveCall(
        val callId: String,
        val callerId: String,
        val calleeId: String,
        val channelName: String,
        val partnerName: String,
        val partnerPhone: String,
        val isIncoming: Boolean,
        val wasIncoming: Boolean = isIncoming,
        val isActive: Boolean = false,
        val remotePresent: Boolean = false,
        val callEnded: Boolean = false,
        val remoteRinging: Boolean = false,
        val muted: Boolean = false,
        val speakerOn: Boolean = false,
        val isVideoCall: Boolean = false,
        val localVideoOn: Boolean = false,
        val remoteVideoOn: Boolean = false,
        val frontCamera: Boolean = true,
        val seconds: Int = 0,
        val connectionState: CallConnectionState = CallConnectionState.CONNECTING,
        val error: String? = null,
    )

    private val _state = MutableStateFlow<ActiveCall?>(null)
    val state: StateFlow<ActiveCall?> = _state.asStateFlow()

    internal fun set(call: ActiveCall) { _state.value = call }
    internal fun update(transform: (ActiveCall) -> ActiveCall) { _state.update { it?.let(transform) } }
    internal fun clear() { _state.value = null }
    fun snapshot(): ActiveCall? = _state.value
}
```

---

## 2. CallBarPresenter and CallUiState

`presentation/components/call/CallBarPresenter.kt` in full:

```kotlin
enum class CallBarMode { BAR, HIDDEN, BUBBLE }

object CallBarPresenter {
    private val _mode = MutableStateFlow(CallBarMode.BAR)
    val mode: StateFlow<CallBarMode> = _mode.asStateFlow()

    @Volatile var bubbleOffset: Offset? = null

    fun showBar() { _mode.value = CallBarMode.BAR }
    fun hide() { _mode.value = CallBarMode.HIDDEN }
    fun minimizeToBubble() { _mode.value = CallBarMode.BUBBLE }
    fun reset() { _mode.value = CallBarMode.BAR; bubbleOffset = null }
}
```

`presentation/state/CallUiState.kt` in full - a flat mirror of `ActiveCall` for the UI:

```kotlin
data class CallUiState(
    val partnerName: String = "",
    val partnerPhone: String = "",
    val partnerAvatarUrl: String? = null,
    val channelName: String = "",
    val isIncoming: Boolean = false,
    val isActive: Boolean = false,
    val remotePresent: Boolean = false,
    val callEnded: Boolean = false,
    val remoteRinging: Boolean = false,
    val seconds: Int = 0,
    val speakerOn: Boolean = false,
    val muted: Boolean = false,
    val isVideoCall: Boolean = false,
    val localVideoOn: Boolean = false,
    val remoteVideoOn: Boolean = false,
    val frontCamera: Boolean = true,
    val connectionState: CallConnectionState = CallConnectionState.CONNECTING,
    val error: UiText? = null,
) : UiState
```

---

## 3. FirebaseCallSignalingService

`data/remote/call/FirebaseCallSignalingService.kt` in full - all call signaling over Realtime
Database (three trees: `calls`, `callAcks`, `callMembers`):

```kotlin
class FirebaseCallSignalingService @Inject constructor(
    private val db: FirebaseDatabase,
) : ICallSignalingService {

    private val callsRef get() = db.reference.child("calls")
    private val callAcksRef get() = db.reference.child("callAcks")
    private val callMembersRef get() = db.reference.child("callMembers")

    override suspend fun sendCallSignal(signal: CallSignal) {
        callsRef.child(signal.calleeId).child(signal.callId).setValue(signal.toDto()).await()
    }

    override fun observeIncomingCall(myUserId: String): Flow<CallSignal?> = callbackFlow {
        val inbox = callsRef.child(myUserId)
        val listener = object : ValueEventListener {
            override fun onDataChange(snapshot: DataSnapshot) {
                val now = System.currentTimeMillis()
                val signal = snapshot.children
                    .mapNotNull { it.getValue(CallSignalDto::class.java)?.toCallSignal() }
                    .filter { it.status == CallStatus.RINGING && now - it.timestamp < RINGING_TTL_MS }
                    .maxByOrNull { it.timestamp }
                trySend(signal)
            }
            override fun onCancelled(error: DatabaseError) { trySend(null) }
        }
        inbox.addValueEventListener(listener)
        awaitClose { inbox.removeEventListener(listener) }
    }

    override suspend fun updateCallStatus(calleeId: String, callId: String, status: CallStatus) {
        callsRef.child(calleeId).child(callId).child("status").setValue(status.name).await()
    }

    override fun observeCallStatus(calleeId: String, callId: String): Flow<CallStatus?> = callbackFlow {
        val ref = callsRef.child(calleeId).child(callId).child("status")
        val listener = object : ValueEventListener {
            override fun onDataChange(snapshot: DataSnapshot) {
                trySend(snapshot.getValue(String::class.java)?.let { runCatching { CallStatus.valueOf(it) }.getOrNull() })
            }
            override fun onCancelled(error: DatabaseError) { trySend(null) }
        }
        ref.addValueEventListener(listener)
        awaitClose { ref.removeEventListener(listener) }
    }

    override suspend fun clearCall(calleeId: String, callId: String) {
        callsRef.child(calleeId).child(callId).removeValue().await()
    }

    override suspend fun ackRinging(callerId: String, callId: String, calleeId: String) {
        callAcksRef.child(callerId).child(callId)
            .setValue(mapOf("calleeId" to calleeId, "ringingAt" to System.currentTimeMillis())).await()
    }

    override fun observeRingingAck(callerId: String, callId: String): Flow<Boolean> = callbackFlow {
        val ref = callAcksRef.child(callerId).child(callId)
        val listener = object : ValueEventListener {
            override fun onDataChange(snapshot: DataSnapshot) { trySend(snapshot.exists()) }
            override fun onCancelled(error: DatabaseError) { trySend(false) }
        }
        ref.addValueEventListener(listener)
        awaitClose { ref.removeEventListener(listener) }
    }

    override suspend fun clearRingingAck(callerId: String, callId: String) {
        callAcksRef.child(callerId).child(callId).removeValue().await()
    }

    override suspend fun joinCallPresence(callId: String, uid: String) {
        val ref = callMembersRef.child(callId).child(uid)
        ref.onDisconnect().removeValue()
        ref.setValue(true).await()
    }

    override fun leaveCallPresence(callId: String, uid: String) {
        val ref = callMembersRef.child(callId).child(uid)
        ref.onDisconnect().cancel()
        ref.removeValue()
    }

    override fun observeCallMembers(callId: String): Flow<Set<String>> = callbackFlow {
        val ref = callMembersRef.child(callId)
        val listener = object : ValueEventListener {
            override fun onDataChange(snapshot: DataSnapshot) { trySend(snapshot.children.mapNotNull { it.key }.toSet()) }
            override fun onCancelled(error: DatabaseError) { trySend(emptySet()) }
        }
        ref.addValueEventListener(listener)
        awaitClose { ref.removeEventListener(listener) }
    }

    private companion object { const val RINGING_TTL_MS = 60_000L }
}
```

All reads are `callbackFlow` + `ValueEventListener` (family in [[Async and Reactivity]]).
`observeIncomingCall` keeps only `RINGING` signals within a 60s TTL and takes the newest.
`joinCallPresence` uses `onDisconnect().removeValue()` so a crashed peer disappears from
`callMembers` server-side - that is what the foreground service's presence timeout watches.

---

## 4. AgoraCallService (essential parts)

`data/remote/call/AgoraCallService.kt` (`ICallService`) wraps one `RtcEngine` and maps Agora's
`IRtcEngineEventHandler` onto the domain `CallEventListener`. Abridged to structure:

```kotlin
@Singleton
class AgoraCallService @Inject constructor(@ApplicationContext private val ctx: Context) : ICallService {
    private var engine: RtcEngine? = null
    private var listener: CallEventListener? = null
    private var localVideoOn = false

    private val handler = object : IRtcEngineEventHandler() {
        override fun onUserJoined(uid: Int, elapsed: Int) { listener?.onRemoteUserJoined(uid) }
        override fun onUserOffline(uid: Int, reason: Int) {
            if (reason == Constants.USER_OFFLINE_QUIT) listener?.onRemoteUserLeft(uid)
            else listener?.onRemoteConnectionLost(uid)
        }
        override fun onConnectionStateChanged(state: Int, reason: Int) {
            listener?.onConnectionStateChanged(state.toCallConnectionState())
        }
        override fun onTokenPrivilegeWillExpire(token: String?) { listener?.onTokenPrivilegeWillExpire() }
        // onRemoteAudioStateChanged / onRemoteVideoStateChanged map to onRemoteConnection*/onRemoteVideoStateChanged
    }

    init {
        if (BuildConfig.AGORA_APP_ID.isBlank()) {
            Log.e(TAG, "BuildConfig.AGORA_APP_ID is missing. Set AGORA_APP_ID in local.properties or as a Gradle property.")
        } else try {
            val config = RtcEngineConfig().apply { mContext = ctx; mAppId = BuildConfig.AGORA_APP_ID; mEventHandler = handler }
            engine = RtcEngine.create(config)?.also { it.enableAudio() }
        } catch (t: Throwable) {
            Log.e(TAG, "Failed to create Agora RtcEngine", t)
        }
    }

    override fun joinChannel(channelName: String, uid: Int, token: String?) {
        val options = ChannelMediaOptions().apply {
            channelProfile = Constants.CHANNEL_PROFILE_COMMUNICATION
            clientRoleType = Constants.CLIENT_ROLE_BROADCASTER
            publishMicrophoneTrack = true
            publishCameraTrack = localVideoOn
            autoSubscribeVideo = true
        }
        engine?.joinChannel(token.orEmpty(), channelName, uid, options)
    }

    override fun enableLocalVideo(enable: Boolean) { /* ensureVideoModule + startPreview/stopPreview + updateChannelMediaOptions */ }
    override fun bindLocalVideo(view: SurfaceView) { engine?.setupLocalVideo(VideoCanvas(view, RENDER_MODE_HIDDEN, 0)) }
    override fun bindRemoteVideo(view: SurfaceView, uid: Int) { engine?.setupRemoteVideo(VideoCanvas(view, RENDER_MODE_HIDDEN, uid)) }
    override fun renewToken(token: String) { engine?.renewToken(token) }
    override fun leaveChannel() { /* stopPreview if video, then engine.leaveChannel() */ }
    override fun muteLocalAudio(mute: Boolean) { engine?.muteLocalAudioStream(mute) }
    override fun setSpeakerphone(enable: Boolean) { engine?.setEnableSpeakerphone(enable) }
    override fun switchCamera() { engine?.switchCamera() }
    override fun setEventListener(listener: CallEventListener) { this.listener = listener }
    override fun clearEventListener(listener: CallEventListener) { if (this.listener === listener) this.listener = null }
}
```

The full file additionally contains the `Int -> CallConnectionState` mapping and verbose
human-readable log helpers for every Agora constant. `onUserOffline` distinguishes an intentional
quit from a network drop, which is the seed of the reconnect-vs-end decision in the service.

---

## 5. CallForegroundService (essential parts)

`data/remote/call/CallForegroundService.kt` is the state machine. It is ~990 lines; the structure
that matters:

```kotlin
@AndroidEntryPoint
class CallForegroundService : Service() {
    @Inject lateinit var callService: ICallService
    @Inject lateinit var signalingService: ICallSignalingService
    @Inject lateinit var missedCallRecorder: MissedCallRecorder
    @Inject lateinit var tokenProvider: AgoraTokenProvider

    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private var tickerJob: Job? = null
    private var statusObserverJob: Job? = null
    private var ringTimeoutJob: Job? = null
    private var requestingTimeoutJob: Job? = null
    private var outgoingTimeoutJob: Job? = null
    private var ringingAckJob: Job? = null
    private var reconnectTimeoutJob: Job? = null
    private var presenceObserverJob: Job? = null
    private var presenceTimeoutJob: Job? = null
    // + audio focus / ringback / recorded-flags state

    override fun onCreate() { super.onCreate(); callService.setEventListener(eventListener) }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        when (intent?.action) {
            ACTION_START_OUTGOING -> handleStartOutgoing(intent)
            ACTION_START_INCOMING -> handleStartIncoming(intent)
            ACTION_ACCEPT -> handleAccept()
            ACTION_DECLINE -> handleDecline()
            ACTION_END -> handleEnd()
            ACTION_TOGGLE_MUTE -> handleToggleMute()
            ACTION_TOGGLE_SPEAKER -> handleToggleSpeaker()
            ACTION_SET_SPEAKER -> handleSetSpeaker(intent.getBooleanExtra(EXTRA_SPEAKER, false))
            ACTION_TOGGLE_CAMERA -> handleToggleCamera()
            ACTION_SWITCH_CAMERA -> handleSwitchCamera()
        }
        return START_NOT_STICKY
    }

    // Remote signaling drives state transitions:
    private fun observeRemoteStatus(calleeId: String, callId: String, isCaller: Boolean) {
        statusObserverJob?.cancel()
        statusObserverJob = scope.launch {
            signalingService.observeCallStatus(calleeId, callId).collectLatest { status ->
                when (status) {
                    CallStatus.ACTIVE -> { /* caller enters active, start timer + presence */ }
                    CallStatus.ENDED -> if (ActiveCallHolder.snapshot()?.isActive == true) recordEndedCallThenStop() else recordMissedCallThenStop()
                    CallStatus.DECLINED -> stopSelfClean()
                    CallStatus.RINGING, null -> Unit
                }
            }
        }
    }

    // Agora events also drive state:
    private val eventListener = object : CallEventListener {
        override fun onRemoteUserJoined(uid: Int) { ActiveCallHolder.update { it.copy(isActive = true, isIncoming = false, remotePresent = true) }; enterActiveState(); reconcileConnection() }
        override fun onRemoteUserLeft(uid: Int) { recordEndedCallThenStop() }
        override fun onRemoteConnectionLost(uid: Int) { ActiveCallHolder.update { it.copy(remotePresent = false) }; reconcileConnection() }
        override fun onRemoteConnectionRestored(uid: Int) { ActiveCallHolder.update { it.copy(remotePresent = true) }; reconcileConnection() }
        override fun onConnectionStateChanged(state: CallConnectionState) { ActiveCallHolder.update { it.copy(connectionState = state) }; reconcileConnection() }
        override fun onTokenPrivilegeWillExpire() { scope.launch { tokenProvider.fetchToken(channel)?.let { callService.renewToken(it) } } }
    }

    private fun stopSelfClean() {
        // cancel EVERY job, stop ringback, leave channel, release audio focus,
        // ActiveCallHolder.clear(), ActiveConnectionHolder.disconnect(callId, cause), stopForeground, stopSelf()
    }

    override fun onDestroy() {
        stopTicker(); stopRingback(); releaseAudioFocus()
        ActiveConnectionHolder.disconnectAll(DisconnectCause.LOCAL)  // no stray system call
        ActiveCallHolder.clear()
        callService.clearEventListener(eventListener)
        scope.coroutineContext[Job]?.cancel()   // nothing outlives the service
        super.onDestroy()
    }
}
```

Everything runs on `scope = SupervisorJob() + Dispatchers.IO`; the named timeout jobs (ring 60s,
requesting 30s, outgoing 65s, reconnect 20s) make each call edge case deterministic;
`stopSelfClean` and `onDestroy` guarantee no job - and no leaked Telecom connection - outlives the
service (see the race-proof `ActiveConnectionHolder` in [[Calls (Voice and Video)]]). The ongoing
in-call notification is `buildOngoingNotification` (`:803`), which uses
`NotificationCompat.CallStyle.forOngoingCall(person, hangUpIntent)` on API 31+ (a persistent
system in-call notification with an End action) and a plain builder below that. The full handler
bodies (`handleStartOutgoing`, `handleStartIncoming`, `handleAccept`, `terminate`, notification
builders, audio focus, ringback, presence tracking) are in the source and narrated in
[[Calls (Voice and Video)]].

---

## 6. CallViewModel

`presentation/viewmodel/CallViewModel.kt` - the UI mirror. Key parts:

```kotlin
@HiltViewModel
class CallViewModel @Inject constructor(
    application: Application,
    private val auth: FirebaseAuth,
    private val conversationRepository: IConversationRepository,
    private val userRepository: IUserRepository,
    private val telecomCallManager: TelecomCallManager,
    private val callService: ICallService,
    savedStateHandle: SavedStateHandle,
) : AndroidViewModel(application) {

    private val _uiState = MutableStateFlow(CallUiState())
    val uiState: StateFlow<CallUiState> = _uiState.asStateFlow()

    private val _callExit = Channel<CallExit>(Channel.BUFFERED)
    val callExit = _callExit.receiveAsFlow()

    private fun observeActiveCall() {
        viewModelScope.launch {
            ActiveCallHolder.state.collectLatest { active ->
                if (active == null) {
                    if (hadActiveCall) { /* play "ended" state, then routeExit via _callExit */ }
                    return@collectLatest
                }
                hadActiveCall = true; lastCall = active
                if (active.isActive) wasAnswered = true
                observePartnerAvatar(if (active.isIncoming) active.callerId else active.calleeId)
                _uiState.value = CallUiState(
                    partnerName = active.partnerName, partnerPhone = active.partnerPhone,
                    partnerAvatarUrl = partnerAvatarUrl, channelName = active.channelName,
                    isIncoming = active.isIncoming, isActive = active.isActive,
                    remotePresent = active.remotePresent, remoteRinging = active.remoteRinging,
                    seconds = active.seconds, muted = active.muted, speakerOn = active.speakerOn,
                    isVideoCall = active.isVideoCall, localVideoOn = active.localVideoOn,
                    remoteVideoOn = active.remoteVideoOn, frontCamera = active.frontCamera,
                    connectionState = active.connectionState, error = active.error?.toUiText(),
                )
            }
        }
    }

    fun startOutgoing() { /* build TelecomCallMeta + CallForegroundService.outgoingIntent, startForegroundService, telecom.placeOutgoing */ }
    fun acceptCall()  = sendAction(CallForegroundService.ACTION_ACCEPT)
    fun declineCall() = sendAction(CallForegroundService.ACTION_DECLINE)
    fun endCall()     = sendAction(CallForegroundService.ACTION_END)
    fun toggleMute()  = sendAction(CallForegroundService.ACTION_TOGGLE_MUTE)
    fun toggleSpeaker() = sendAction(CallForegroundService.ACTION_TOGGLE_SPEAKER)
    fun toggleCamera() = sendAction(CallForegroundService.ACTION_TOGGLE_CAMERA)
    fun switchCamera() = sendAction(CallForegroundService.ACTION_SWITCH_CAMERA)
    fun bindLocalVideo(view: SurfaceView) = callService.bindLocalVideo(view)
    fun bindRemoteVideo(view: SurfaceView) { /* callService.bindRemoteVideo(view, CallUids.fromUserId(partnerId)) */ }

    private fun sendAction(action: String) {
        val intent = Intent(context, CallForegroundService::class.java).setAction(action)
        runCatching { context.startService(intent) }
    }
}
```

The VM holds no call logic - it collects `ActiveCallHolder.state` (`collectLatest`), copies each
field into `CallUiState`, and turns user actions into `startService` intents. The one-shot exit is
a `Channel<CallExit>` (`receiveAsFlow`) yielding `Pop` or `OpenChat`. `observePartnerAvatar`
cancels the previous observer before resubscribing.

---

## 7. ActiveCallBar - the swipe/bubble/snap-back gesture

`presentation/components/call/ActiveCallBar.kt` - the gesture handler (the load-bearing block):

```kotlin
Box(
    modifier = modifier.fillMaxWidth()
        .graphicsLayer { translationY = dragY.value }
        .pointerInput(Unit) {
            detectVerticalDragGestures(
                onVerticalDrag = { change, delta ->
                    change.consume()
                    scope.launch { dragY.snapTo(dragY.value + delta) }
                },
                onDragEnd = {
                    val offset = dragY.value
                    when {
                        offset <= -hideThresholdPx -> scope.launch {
                            dragY.animateTo(-hideTargetPx, tween(160)); CallBarPresenter.hide()
                        }
                        offset >= bubbleThresholdPx -> {
                            CallBarPresenter.bubbleOffset = null
                            CallBarPresenter.minimizeToBubble()
                        }
                        else -> scope.launch { dragY.animateTo(0f, tween(180)) }
                    }
                },
                onDragCancel = { scope.launch { dragY.animateTo(0f, tween(180)) } },
            )
        },
) { CallBarRow(call = call, context = context, onClick = onClick) }
```

Swipe up past threshold -> `hide()`; swipe down past threshold -> `minimizeToBubble()`;
otherwise `animateTo(0f)` snaps back. The bar reads `ActiveCallHolder.state` and
`CallBarPresenter.mode`, and shows a live waveform + timer when connected or a status label
otherwise (`callBarStatus`). Full behaviour in [[Calls (Voice and Video)]].

---

## 8. CallHistoryRepository

`data/repository/CallHistoryRepository.kt` in full - the call log derived from call-type messages:

```kotlin
@Singleton
class CallHistoryRepository @Inject constructor(
    private val messageBox: Box<ObxMessage>,
    private val conversationBox: Box<ObxConversation>,
) {
    fun getCallHistory(currentUserId: String): Flow<List<CallHistoryEntry>> {
        val query = messageBox.query(
            ObxMessage_.type.oneOf(CALL_TYPES, QueryBuilder.StringOrder.CASE_SENSITIVE),
        ).orderDesc(ObxMessage_.timestamp).build()
        return query.asFlow().map { rows ->
            val liveRows = rows.filterNot { it.deleted }
            val conversationIds = liveRows.map { it.conversationId }.distinct().toTypedArray()
            val conversationsById = if (conversationIds.isEmpty()) emptyMap() else
                conversationBox.query(ObxConversation_.uid.oneOf(conversationIds)).build()
                    .use { it.find() }.associateBy { it.uid }
            liveRows.mapNotNull { it.toEntry(currentUserId, conversationsById[it.conversationId]) }
        }
    }

    private fun ObxMessage.toEntry(currentUserId: String, conversation: ObxConversation?): CallHistoryEntry? {
        conversation ?: return null
        val partnerIndex = conversation.participantIds.indexOfFirst { it != currentUserId }
        val partnerId = conversation.participantIds.getOrNull(partnerIndex)
            ?: conversation.participantIds.firstOrNull { it != currentUserId } ?: return null
        return CallHistoryEntry(
            id = uid, conversationId = conversationId, partnerId = partnerId,
            partnerName = conversation.participantNames.getOrNull(partnerIndex).orEmpty(),
            partnerAvatarUrl = conversation.participantAvatars.getOrNull(partnerIndex),
            timestamp = timestamp, callType = type.toCallType(),
            video = callVideo, durationSeconds = callDurationSeconds,
            outgoing = senderId == currentUserId,
        )
    }

    private fun String.toCallType(): CallType = when (this) {
        Message.TYPE_MISSED_CALL -> CallType.MISSED
        Message.TYPE_UNREACHED_CALL -> CallType.UNREACHED
        else -> CallType.ENDED
    }

    private companion object {
        val CALL_TYPES = arrayOf(Message.TYPE_MISSED_CALL, Message.TYPE_UNREACHED_CALL, Message.TYPE_ENDED_CALL)
    }
}
```

The query filters on the indexed `ObxMessage.type` (`oneOf(CALL_TYPES)`), then batches the needed
conversations in one `uid.oneOf(...)` query (avoids an N+1 per row) and maps each to a
`CallHistoryEntry`. Returning `asFlow().map` keeps the history live.

---

## Related

[[Calls (Voice and Video)]] - [[Notifications - Code]] - [[Async and Reactivity]] -
[[Navigation]] - [[Data Layer]] - [[Presentation Layer (MVI)]]
