# Notifications - Code

Companion to [[Notifications]]. The source for the notification feature, block by block.
Snippets are verbatim from the repo.

Files covered:
- `data/remote/firebase/MyFirebaseMessagingService.kt`
- `data/remote/firebase/FirebaseMessagingManager.kt`, `data/remote/firebase/FcmTokenSyncer.kt`
- `presentation/notification/{ChatNotifier, NotificationChannels, NotificationReplyReceiver,
  CurrentConversationHolder}.kt`

---

## 1. FCM token: manager + syncer

`FirebaseMessagingManager.kt`:

```kotlin
@Singleton
class FirebaseMessagingManager @Inject constructor(
    private val firebaseMessaging: FirebaseMessaging
) {
    suspend fun getFcmToken(): Result<String> = try {
        Result.success(firebaseMessaging.token.await())
    } catch (e: Exception) { Result.failure(e) }
}
```

`FcmTokenSyncer.kt` - keeps the Firestore copy current across auth changes, started from
`MessengerApplication.onCreate`:

```kotlin
@Singleton
class FcmTokenSyncer @Inject constructor(
    private val auth: FirebaseAuth,
    private val messagingManager: FirebaseMessagingManager,
    private val firestoreService: FirestoreService,
) {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private var lastSyncedUid: String? = null

    fun start() {
        auth.addAuthStateListener { firebaseAuth ->
            val uid = firebaseAuth.currentUser?.uid
            if (uid == null) { lastSyncedUid = null; return@addAuthStateListener }
            if (uid == lastSyncedUid) return@addAuthStateListener
            syncFor(uid)
        }
    }

    private fun syncFor(uid: String) {
        scope.launch {
            messagingManager.getFcmToken()
                .onSuccess { token ->
                    if (firestoreService.updateFcmToken(uid, token).isSuccess) lastSyncedUid = uid
                }
                .onFailure { e -> Log.w(TAG, "Failed to fetch FCM token", e) }
        }
    }
    private companion object { const val TAG = "FcmTokenSyncer" }
}
```

`lastSyncedUid` de-dupes repeated syncs for the same user. The token also gets written at login
(`AuthViewModel.syncFcmTokenForCurrentUser`, [[Authentication - Code]]) and on rotation
(`MyFirebaseMessagingService.onNewToken`).

---

## 2. MyFirebaseMessagingService

`data/remote/firebase/MyFirebaseMessagingService.kt` - the FCM receiver (logging trimmed,
named arguments and fully qualified names condensed). It is OS-constructed, so it reaches Hilt via
a manual `@EntryPoint`.

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface FcmEntryPoint {
    fun firestoreService(): FirestoreService
    fun telecomCallManager(): TelecomCallManager
    fun messageRepository(): IMessageRepository
    fun e2eeCodec(): E2eeMessageCodec
}

class MyFirebaseMessagingService : FirebaseMessagingService() {
    private val TAG = "MyFirebaseMsgService"

    override fun onNewToken(token: String) {
        super.onNewToken(token)
        val currentUid = FirebaseAuth.getInstance().currentUser?.uid ?: return
        val firestoreService = EntryPointAccessors.fromApplication(applicationContext, FcmEntryPoint::class.java).firestoreService()
        CoroutineScope(Dispatchers.IO).launch {
            firestoreService.updateFcmToken(currentUid, token).onFailure { Log.w(TAG, "token sync failed", it) }
        }
    }

    override fun onMessageReceived(remoteMessage: RemoteMessage) {
        super.onMessageReceived(remoteMessage)
        val data = remoteMessage.data
        if (FirebaseAuth.getInstance().currentUser == null) return
        when (data["type"]) {
            TYPE_MESSAGE -> handleMessagePayload(data)
            TYPE_CALL -> handleCallPayload(data)
            else -> remoteMessage.notification?.let { ChatNotifier.notifyFallback(this, it.title, it.body) }
        }
    }

    private fun handleMessagePayload(data: Map<String, String>) {
        val conversationId = data["conversationId"].orEmpty()
        val senderId = data["senderId"].orEmpty()
        val senderName = data["senderName"].orEmpty().ifBlank { getString(R.string.notif_message_fallback_sender) }
        val senderAvatar = data["senderAvatar"].orEmpty()
        val timestamp = data["timestamp"]?.toLongOrNull() ?: System.currentTimeMillis()
        val dto = parseIncomingDto(data)
        if (dto == null) { ChatNotifier.notifyFallback(this, senderName, data["preview"].orEmpty()); return }

        val entryPoint = EntryPointAccessors.fromApplication(applicationContext, FcmEntryPoint::class.java)
        val messageRepository = entryPoint.messageRepository()
        val codec = entryPoint.e2eeCodec()
        val firestoreService = entryPoint.firestoreService()

        CoroutineScope(Dispatchers.IO).launch {
            val resolved = if (dto.enc == 1 && dto.ciphertext.isBlank())
                firestoreService.getMessageOnce(dto.conversationId, dto.id).getOrNull() ?: dto else dto
            val message = runCatching { codec.decode(resolved) }.getOrNull()
            if (message != null && resolved.type == Message.TYPE_TEXT) {
                runCatching { messageRepository.persistIncomingMessage(message) }
                    .onFailure { Log.w(TAG, "persistIncomingMessage failed", it) }
            }
            if (CurrentConversationHolder.isOpen(conversationId)) return@launch
            val clipLabel = clipLabelFor(
                msgType = resolved.type,
                voice = getString(R.string.clip_voice_message),
                video = getString(R.string.clip_video_message),
            )
            val body = when {
                clipLabel != null -> clipLabel
                message is Message.Text && message.text != E2eeMessageCodec.UNDECRYPTABLE_TEXT -> message.text
                message is Message.Media -> message.caption.ifBlank { "Media" }
                dto.enc == 1 -> "New message"
                else -> data["preview"].orEmpty()
            }
            ChatNotifier.notifyIncoming(this@MyFirebaseMessagingService, conversationId, senderId,
                senderName, body, timestamp, senderAvatar)
        }
    }

    private fun handleCallPayload(data: Map<String, String>) {
        val callId = data["callId"].orEmpty()
        val callerId = data["callerId"].orEmpty()
        val callerName = data["callerName"].orEmpty().ifBlank { callerId }
        val channelName = data["channelName"].orEmpty()
        val calleeId = FirebaseAuth.getInstance().currentUser?.uid.orEmpty()
        if (callId.isBlank() || channelName.isBlank() || calleeId.isBlank()) return
        if (ActiveCallHolder.snapshot()?.callId == callId) return
        val intent = CallForegroundService.incomingIntent(this, callId, callerId, calleeId,
            channelName, callerName, "")
        ContextCompat.startForegroundService(this, intent)
        val meta = TelecomCallMeta(callId, callerId, calleeId, channelName, callerName, "", isIncoming = true)
        runCatching {
            EntryPointAccessors.fromApplication(applicationContext, FcmEntryPoint::class.java)
                .telecomCallManager().addIncoming(meta)
        }
    }

    companion object { private const val TYPE_MESSAGE = "message"; private const val TYPE_CALL = "call" }
}
```

The clip label comes from a pure helper in `data/remote/firebase/IncomingMessagePayload.kt`,
covered by `IncomingMessagePayloadTest`:

```kotlin
fun clipLabelFor(msgType: String, voice: String, video: String): String? = when (msgType) {
    Message.TYPE_VOICE_CLIP -> voice
    Message.TYPE_VIDEO_CLIP -> video
    else -> null
}
```

Each push spins an ephemeral `CoroutineScope(Dispatchers.IO)` (no long-lived scope in an
OS-constructed service). A message push is decoded (fetching the full ciphertext from Firestore if
the payload was stripped), the text message is persisted locally for offline delivery, and a
notification is posted unless the conversation is already open. A call push starts the
`CallForegroundService` and registers a Telecom incoming connection. DI-from-EntryPoint is covered
in [[Dependency Injection (Hilt)]].

---

## 3. CurrentConversationHolder

`presentation/notification/CurrentConversationHolder.kt` - the "is this chat on screen" gate:

```kotlin
object CurrentConversationHolder {
    private val ref = AtomicReference<String?>(null)
    fun setOpen(conversationId: String) { ref.set(conversationId) }
    fun clear(conversationId: String) { ref.compareAndSet(conversationId, null) }
    fun isOpen(conversationId: String): Boolean = ref.get() == conversationId
}
```

Set by `ChatScreen` while open ([[Chat and Messaging - Code]]) and read by the FCM handler above.
`compareAndSet` on clear avoids clearing a different conversation that opened in the meantime.

---

## 4. NotificationChannels

`presentation/notification/NotificationChannels.kt` - all channels, created once at startup:

```kotlin
object NotificationChannels {
    const val CHAT_MESSAGES = "chat_messages_channel"
    const val INCOMING_CALL = "call_channel"
    const val ONGOING_CALL = "ongoing_call_channel"
    const val MISSED_CALLS = "missed_calls_channel"
    private const val GROUP_MESSAGES = "messages_group"
    private const val GROUP_CALLS = "calls_group"

    fun ensureCreated(context: Context) {
        val nm = context.getSystemService<NotificationManager>() ?: return
        nm.createNotificationChannelGroup(NotificationChannelGroup(GROUP_MESSAGES, context.getString(R.string.notif_group_messages)))
        nm.createNotificationChannelGroup(NotificationChannelGroup(GROUP_CALLS, context.getString(R.string.notif_group_calls)))

        if (nm.getNotificationChannel(CHAT_MESSAGES) == null) {
            nm.createNotificationChannel(NotificationChannel(CHAT_MESSAGES,
                context.getString(R.string.notif_channel_messages_name), NotificationManager.IMPORTANCE_HIGH).apply {
                group = GROUP_MESSAGES; enableLights(true); enableVibration(true); setShowBadge(true)
                lockscreenVisibility = Notification.VISIBILITY_PRIVATE
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) setAllowBubbles(true)
            })
        }
        if (nm.getNotificationChannel(INCOMING_CALL) == null) {
            val ringtoneUri = RingtoneManager.getDefaultUri(RingtoneManager.TYPE_RINGTONE)
            val audioAttrs = AudioAttributes.Builder()
                .setUsage(AudioAttributes.USAGE_NOTIFICATION_RINGTONE)
                .setContentType(AudioAttributes.CONTENT_TYPE_SONIFICATION).build()
            nm.createNotificationChannel(NotificationChannel(INCOMING_CALL,
                context.getString(R.string.notif_channel_incoming_call_name), NotificationManager.IMPORTANCE_HIGH).apply {
                group = GROUP_CALLS; setBypassDnd(true); enableLights(true); enableVibration(true); setSound(ringtoneUri, audioAttrs)
            })
        }
        // ONGOING_CALL (IMPORTANCE_LOW, silent) and MISSED_CALLS (IMPORTANCE_HIGH) created the same way.
    }
}
```

`CHAT_MESSAGES` enables bubbles (`setAllowBubbles`), `INCOMING_CALL` bypasses Do-Not-Disturb and
plays the ringtone, `ONGOING_CALL` is silent/low. Called from `MessengerApplication.onCreate`.

---

## 5. NotificationReplyReceiver

`presentation/notification/NotificationReplyReceiver.kt` - handles the inline actions:

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface NotificationActionEntryPoint {
    fun sendMessageUseCase(): SendMessageUseCase
    fun markConversationAsReadUseCase(): MarkConversationAsReadUseCase
}

class NotificationReplyReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val conversationId = intent.getStringExtra(EXTRA_CONVERSATION_ID).orEmpty()
        if (conversationId.isBlank()) return
        val appContext = context.applicationContext
        if (intent.action == ACTION_OPEN_BUBBLE) {
            ChatNotifier.openAsBubble(appContext, conversationId,
                intent.getStringExtra(EXTRA_PARTNER_ID).orEmpty(),
                intent.getStringExtra(EXTRA_PARTNER_NAME).orEmpty())
            return
        }
        val entry = EntryPointAccessors.fromApplication(appContext, NotificationActionEntryPoint::class.java)
        val pending = goAsync()
        val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
        scope.launch {
            try {
                when (intent.action) {
                    ACTION_REPLY -> handleReply(appContext, intent, conversationId, entry)
                    ACTION_MARK_READ -> handleMarkRead(appContext, conversationId, entry)
                }
            } catch (t: Throwable) { Log.w(TAG, "notification action failed", t) }
            finally { pending.finish() }
        }
    }

    private suspend fun handleReply(context: Context, intent: Intent, conversationId: String, entry: NotificationActionEntryPoint) {
        val text = RemoteInput.getResultsFromIntent(intent)?.getCharSequence(KEY_TEXT_REPLY)?.toString().orEmpty().trim()
        if (text.isBlank()) return
        entry.sendMessageUseCase().invoke(conversationId, text).collect { }
        ChatNotifier.appendOutgoing(context, conversationId, text, System.currentTimeMillis())
    }

    private suspend fun handleMarkRead(context: Context, conversationId: String, entry: NotificationActionEntryPoint) {
        entry.markConversationAsReadUseCase().invoke(conversationId)
        ChatNotifier.clear(context, conversationId)
    }
}
```

`goAsync()` + a short-lived scope keeps the broadcast alive while the reply send / mark-read
suspends; `pending.finish()` releases it in `finally`. A reply reuses `SendMessageUseCase` (the
same path as the chat screen) and then appends the outgoing line to the notification thread so it
stays in sync.

---

## 6. ChatNotifier (key methods)

`presentation/notification/ChatNotifier.kt` is a global `object` keeping a per-conversation
history so notifications read as a `MessagingStyle` conversation. The load-bearing methods:

```kotlin
fun notifyIncoming(context: Context, conversationId: String, senderId: String, senderName: String,
                   text: String, timestamp: Long, senderAvatar: String = "") {
    val cachedIcon = synchronized(lock) { states[conversationId]?.takeIf { it.partnerAvatar == senderAvatar }?.partnerIcon }
    val icon = cachedIcon ?: loadAvatarIcon(context, senderAvatar) ?: monogramIcon(senderName)
    synchronized(lock) {
        val state = states.getOrPut(conversationId) { ConvState(senderId, senderName, senderAvatar) }
        if (senderAvatar.isNotBlank()) state.partnerAvatar = senderAvatar
        state.partnerIcon = icon
        state.messages.add(Entry(text, timestamp, senderId, senderName, fromMe = false))
        trim(state)
        ConversationShortcuts.push(context, conversationId, state.partnerId, state.partnerName, icon)
        post(context, conversationId, state, silent = false)
        postSummary(context)
    }
}

private fun post(context: Context, conversationId: String, state: ConvState, silent: Boolean, expandAsBubble: Boolean = false) {
    val nm = context.getSystemService<NotificationManager>() ?: return
    val me = Person.Builder().setName(context.getString(R.string.notif_message_you)).build()
    val style = NotificationCompat.MessagingStyle(me)
    state.messages.forEach { entry ->
        val person = if (entry.fromMe) null else Person.Builder().setKey(entry.senderId)
            .setName(entry.senderName).setIcon(state.partnerIcon).build()
        style.addMessage(entry.text, entry.timestamp, person)
    }
    val builder = NotificationCompat.Builder(context, NotificationChannels.CHAT_MESSAGES)
        .setSmallIcon(R.drawable.ic_stat_notification)
        .setStyle(style).setCategory(NotificationCompat.CATEGORY_MESSAGE)
        .setPriority(NotificationCompat.PRIORITY_HIGH).setAutoCancel(true).setSilent(silent)
        .setGroup(GROUP_KEY).setShortcutId(conversationId).setLocusId(LocusIdCompat(conversationId))
        .setContentIntent(contentIntent(context, conversationId, state))
        .addAction(replyAction(context, conversationId, state))
        .addAction(openInWindowAction(context, conversationId, state))
        .addAction(markReadAction(context, conversationId))
    if (expandAsBubble) builder.setBubbleMetadata(bubbleMetadata(context, conversationId, state))
    nm.notify(conversationId.hashCode(), builder.build())
}

private fun bubbleMetadata(context: Context, conversationId: String, state: ConvState): NotificationCompat.BubbleMetadata {
    val intent = Intent(context, BubbleActivity::class.java).apply {
        putExtra(MainActivity.EXTRA_OPEN_CONVERSATION_ID, conversationId)
        putExtra(MainActivity.EXTRA_PARTNER_ID, state.partnerId)
        putExtra(MainActivity.EXTRA_PARTNER_NAME, state.partnerName)
    }
    val pendingIntent = PendingIntent.getActivity(context, (conversationId + ":bubble").hashCode(), intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE)
    val icon = state.partnerIcon ?: IconCompat.createWithResource(context, R.drawable.ic_stat_notification)
    return NotificationCompat.BubbleMetadata.Builder(pendingIntent, icon)
        .setDesiredHeight(BUBBLE_HEIGHT_DP).setAutoExpandBubble(true).setSuppressNotification(true).build()
}
```

`post` builds a `MessagingStyle` with a `Person` per sender, tied to the conversation shortcut +
`LocusId` (needed for bubbles), and attaches reply / open-in-window / mark-read actions. The
avatar `IconCompat` is cached, then loaded via Coil (5s timeout), else a generated monogram
(`monogramIcon`). When 2+ conversations are active, `postSummary` posts an `InboxStyle` group
summary. `appendOutgoing` (used by the reply receiver) adds the user's own reply silently.
`openAsBubble` posts with `BubbleMetadata` pointing at `BubbleActivity`. Full detail in
[[Notifications]].

---

## Related

[[Notifications]] - [[Chat and Messaging - Code]] - [[Calls (Voice and Video) - Code]] -
[[End-to-End Encryption]] - [[Dependency Injection (Hilt)]] - [[Authentication - Code]]
