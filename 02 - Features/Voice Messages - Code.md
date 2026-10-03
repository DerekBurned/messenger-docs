# Voice Messages - Code

Companion to [[Voice Messages]]. The source of the voice pipeline, block by block. Snippets are
copied from the repo with `Log` lines trimmed; the "why" lives in the main note.

Files covered:
- `domain/model/voiceMessage/Clip.kt`
- `data/media/VoiceRecorder.kt`, `data/media/VoiceDraftStore.kt`
- `presentation/components/chat/RecordMicButton.kt`
- `data/repository/ClipRepositoryImpl.kt`
- `data/media/ClipPlaybackController.kt`
- `presentation/screens/chat/VoiceBubble.kt`
- `domain/usecase/message/DeleteMessageUseCase.kt`

---

## 1. The clip model

`domain/model/voiceMessage/Clip.kt`:

```kotlin
@Immutable
sealed class Clip {
    abstract val url: String
    abstract val durationMs: Long
    abstract val sizeBytes: Long
    abstract val fileKey: String?

    abstract val fileExtension: String

    abstract val previewLabel: String

    abstract val maxDurationMs: Long?

    val exceedsDurationLimit: Boolean
        get() = maxDurationMs?.let { durationMs > it } == true

    abstract fun withUpload(url: String, fileKey: String?): Clip

    data class Voice(
        override val url: String = "",
        override val durationMs: Long = 0L,
        override val sizeBytes: Long = 0L,
        override val fileKey: String? = null,
        val waveform: List<Int> = emptyList(),
    ) : Clip() {
        override val fileExtension: String get() = FILE_EXTENSION

        override val previewLabel: String get() = PREVIEW_LABEL

        override val maxDurationMs: Long get() = MAX_DURATION_MS

        override fun withUpload(url: String, fileKey: String?): Clip =
            copy(url = url, fileKey = fileKey)

        companion object {
            const val MAX_DURATION_MS = 60 * 60 * 1000L
            const val PREVIEW_LABEL = "Voice message"
            const val FILE_EXTENSION = "m4a"
        }
    }
}
```

`Clip.Video` has the same shape without a waveform. Every label, extension and limit question is
answered by the subtype, so no caller writes its own `when (clip)`.

---

## 2. Recording

Pure helpers at the top of `data/media/VoiceRecorder.kt`:

```kotlin
fun VoiceRecordState.visibleTo(conversationId: String): VoiceRecordState = when (this) {
    is VoiceRecordState.Recording -> if (this.conversationId == conversationId) this else VoiceRecordState.Idle
    VoiceRecordState.Idle -> this
}

internal fun interruptsRecording(change: Int): Boolean =
    change == AudioManager.AUDIOFOCUS_LOSS || change == AudioManager.AUDIOFOCUS_LOSS_TRANSIENT

internal fun recordedDurationMs(startedAt: Long, stoppedAt: Long): Long =
    (stoppedAt - startedAt).coerceIn(0L, Clip.Voice.MAX_DURATION_MS)
```

The recorder interrupts itself on calls and when the app leaves the foreground:

```kotlin
init {
    scope.launch {
        ActiveCallHolder.state
            .map { it != null }
            .distinctUntilChanged()
            .filter { inCall -> inCall }
            .collect { interrupt() }
    }
    scope.launch {
        ProcessLifecycleOwner.get().lifecycle.currentStateFlow
            .map { it.isAtLeast(Lifecycle.State.STARTED) }
            .distinctUntilChanged()
            .filter { visible -> !visible }
            .collect { interrupt() }
    }
}
```

Finishing versus interrupting (`VoiceRecorder.kt:191`):

```kotlin
fun stop(conversationId: String): VoiceRecording? {
    if (owner != conversationId) return null
    return finish()
}

fun interrupt(conversationId: String? = null) {
    if (recorder == null) return
    if (conversationId != null && owner != conversationId) return
    finish()?.let(drafts::save)
}

private fun finish(): VoiceRecording? {
    val active = recorder
    val file = outputFile
    val conversationId = owner
    val durationMs = recordedDurationMs(startedAt, SystemClock.elapsedRealtime())
    val waveform = downsampleWaveform(samples.toList())
    release(active)
    if (active == null || file == null || conversationId == null) return null
    if (!file.exists() || file.length() == 0L || durationMs < MIN_DURATION_MS) {
        file.delete()
        return null
    }
    return VoiceRecording(
        conversationId = conversationId,
        file = file,
        durationMs = durationMs,
        waveform = waveform,
    )
}
```

`stop` and `interrupt` share `finish`; the only difference is where the result goes - to
`sendVoice`, or to the draft store. `owner` makes a recording belong to one chat.

---

## 3. Drafts

`data/media/VoiceDraftStore.kt`:

```kotlin
fun save(recording: VoiceRecording) {
    val audio = audioFileFor(recording.conversationId)
    if (!mediaCache.move(recording.file, audio)) {
        recording.file.delete()
        return
    }
    runCatching {
        metaFileFor(recording.conversationId)
            .writeText(VoiceDraftMeta(recording.durationMs, recording.waveform).encode())
    }
    drafts.update { it + (recording.conversationId to recording.copy(file = audio)) }
}

fun take(conversationId: String): VoiceRecording? {
    val draft = drafts.value[conversationId] ?: return null
    drafts.update { it - conversationId }
    metaFileFor(conversationId).delete()
    return draft
}
```

`load(conversationId)` rebuilds the in-memory map from `voice_drafts/<id>.m4a` and
`<id>.json` after process death and deletes either file if the other is missing or corrupt.

---

## 4. The gesture

`presentation/components/chat/RecordMicButton.kt`:

```kotlin
internal enum class RecordGestureEnd { SEND, DRAFT, NONE }

internal fun recordGestureEnd(locked: Boolean, slidToCancel: Boolean, released: Boolean): RecordGestureEnd = when {
    locked || slidToCancel -> RecordGestureEnd.NONE
    released -> RecordGestureEnd.SEND
    else -> RecordGestureEnd.DRAFT
}
```

Inside `awaitEachGesture`, `released = !change.isConsumed` is read before `change.consume()`:
Compose delivers a system cancellation as an already-consumed "up", so an unconsumed up is a
real finger release and anything else ends in `DRAFT`.

---

## 5. Sending

`ClipRepositoryImpl.uploadAndSend` (`data/repository/ClipRepositoryImpl.kt:236`):

```kotlin
val encryptActive = codec.isEncryptable(message.conversationId)
val path = storagePath(message.conversationId, messageId, message.clip)
val uploaded = withTimeoutOrNull(clipUploadTimeout(file.length())) {
    fileTransfer.upload(messageId, file, path, encryptActive).getOrNull()
}
if (uploaded == null) {
    manager.cancel(messageId)
    manager.clear(messageId)
    markFailed(messageId)
    return
}
manager.clear(messageId)
val stored = updateClip(messageId) { it.copy(clip = it.clip.withUpload(uploaded.url, uploaded.fileKey)) }
if (stored == null) {
    manager.deleteRemote(uploaded.url)
    return
}
val result = firestoreService.sendMessage(
    stored.copy(status = MessageStatus.SENT),
    confirmWithin = CLIP_WRITE_TIMEOUT,
)
if (result.isFailure) {
    markFailed(messageId)
    return
}
updateClip(messageId) { current ->
    if (current.status == MessageStatus.SENDING || current.status == MessageStatus.FAILED) {
        current.copy(status = MessageStatus.SENT)
    } else {
        current
    }
}
mediaCache.move(file, mediaCache.clipFileFor(messageId, message.clip.fileExtension))
```

`updateClip` is the guard against resurrecting a deleted clip:

```kotlin
private fun updateClip(messageId: String, transform: (Message.Clip) -> Message.Clip): Message.Clip? =
    messageBox.store.callInTx {
        val row = findRow(messageId)
        val current = row?.toDomain() as? Message.Clip
        if (row == null || current == null || current.deleted) return@callInTx null
        transform(current).also { updated ->
            messageBox.put(updated.toObx().also { it.boxId = row.boxId })
        }
    }
```

Each send is one registered job, so retry and cancel can find it and a second start is refused:

```kotlin
private fun launchSend(messageId: String) {
    val job = scope.launch(start = CoroutineStart.LAZY) {
        try {
            uploadAndSend(messageId)
        } catch (e: CancellationException) {
            throw e
        } catch (e: Exception) {
            markFailed(messageId)
        }
    }
    if (sendJobs.putIfAbsent(messageId, job) != null) {
        job.cancel()
        return
    }
    job.invokeOnCompletion { sendJobs.remove(messageId, job) }
    job.start()
}
```

---

## 6. Pure decisions in the repository

```kotlin
internal fun strandedClipIds(unsent: List<Message.Clip>, active: Set<String>, startedAt: Long): List<String> =
    unsent.filter { it.status == MessageStatus.SENDING && it.id !in active && it.timestamp < startedAt }
        .map { it.id }

internal fun nextVoiceClipAfter(current: Message.Clip, candidates: List<Message>): Message.Clip? =
    candidates.asSequence()
        .filterIsInstance<Message.Clip>()
        .filter { it.clip is Clip.Voice && !it.deleted && it.conversationId == current.conversationId && it.id != current.id }
        .filter { it.timestamp > current.timestamp || (it.timestamp == current.timestamp && it.id > current.id) }
        .minWithOrNull(compareBy<Message.Clip>({ it.timestamp }, { it.id }))

internal enum class OutboxFate { KEEP, PROMOTE, DROP }

internal fun outboxFate(row: Message.Clip?): OutboxFate = when {
    row == null || row.deleted -> OutboxFate.DROP
    row.status == MessageStatus.SENDING || row.status == MessageStatus.FAILED -> OutboxFate.KEEP
    else -> OutboxFate.PROMOTE
}

internal fun clipUploadTimeout(sizeBytes: Long): Duration =
    UPLOAD_BASE_TIMEOUT + (sizeBytes.coerceAtLeast(0L) * 1000L / UPLOAD_FLOOR_BYTES_PER_SECOND).milliseconds
```

`UPLOAD_BASE_TIMEOUT = 30.seconds`, `UPLOAD_FLOOR_BYTES_PER_SECOND = 8_000L`. All four are
covered by unit tests (`StrandedClipsTest`, `NextVoiceClipTest`, `ClipUploadTimeoutTest`).

Download deduplication:

```kotlin
internal class InFlight<K : Any, V>(private val scope: CoroutineScope) {
    private val running = HashMap<K, Deferred<V>>()

    suspend fun join(key: K, block: suspend () -> V): V {
        val flight = synchronized(running) {
            running.getOrPut(key) {
                scope.async(start = CoroutineStart.LAZY) { block() }.also { started ->
                    started.invokeOnCompletion {
                        synchronized(running) { if (running[key] === started) running.remove(key) }
                    }
                }
            }
        }
        flight.start()
        return flight.await()
    }
}
```

The work runs in the repository scope, not in the caller, so cancelling one waiter does not
cancel the shared download.

---

## 7. Playback decisions

`data/media/ClipPlaybackController.kt`:

```kotlin
internal fun clipToggle(state: ClipPlaybackState, loadedMessageId: String?, target: String): ClipToggle = when {
    state.messageId != target -> ClipToggle.PLAY
    loadedMessageId == target -> if (state.isPlaying) ClipToggle.PAUSE else ClipToggle.RESUME
    state.isLoading -> ClipToggle.IGNORE
    else -> ClipToggle.PLAY
}

internal fun focusReaction(change: Int, resumeOnGain: Boolean): FocusReaction = when (change) {
    AudioManager.AUDIOFOCUS_LOSS -> FocusReaction.PAUSE_AND_RELEASE
    AudioManager.AUDIOFOCUS_LOSS_TRANSIENT,
    AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK,
    -> FocusReaction.PAUSE_AND_HOLD
    AudioManager.AUDIOFOCUS_GAIN -> if (resumeOnGain) FocusReaction.RESUME else FocusReaction.NONE
    else -> FocusReaction.NONE
}
```

Auto-advance (`ClipPlaybackController.kt:272`):

```kotlin
private fun onCompleted() {
    val finished = loadedMessageId ?: return
    stopProgressUpdates()
    advanceJob?.cancel()
    advanceJob = scope.launch {
        clipRepository.markPlayed(finished)
        val next = clipRepository.nextVoiceClip(finished)
        if (_state.value.messageId != finished) return@launch
        if (next == null) {
            stop()
        } else {
            play(ClipQueueItem(next.id, next.conversationId, next.clip.durationMs))
        }
    }
}
```

The `messageId != finished` check drops the advance if the user started another clip while
`nextVoiceClip` was querying.

---

## 8. The bubble's button

`presentation/screens/chat/VoiceBubble.kt:58`:

```kotlin
internal enum class VoiceButton { CANCEL_SEND, DOWNLOADING, RETRY_SEND, LOADING, RETRY_PLAY, PAUSE, PLAY }

internal fun voiceButton(
    transfer: MediaTransfer?,
    status: MessageStatus,
    isMe: Boolean,
    isLoading: Boolean,
    playFailed: Boolean,
    isPlaying: Boolean,
): VoiceButton = when {
    transfer is MediaTransfer.Uploading -> VoiceButton.CANCEL_SEND
    transfer is MediaTransfer.Downloading -> VoiceButton.DOWNLOADING
    isMe && status == MessageStatus.FAILED -> VoiceButton.RETRY_SEND
    isLoading -> VoiceButton.LOADING
    playFailed -> VoiceButton.RETRY_PLAY
    isPlaying -> VoiceButton.PAUSE
    else -> VoiceButton.PLAY
}
```

---

## 9. Deleting a clip for everyone

`domain/usecase/message/DeleteMessageUseCase.kt`:

```kotlin
result.fold(
    onSuccess = {
        if (forEveryone && message is Message.Clip) clipRepository.deleteRemoteFile(message)
        emit(Resource.Success(Unit))
    },
    onFailure = { e ->
        emit(Resource.Error(e.message ?: "Failed to delete the message"))
    }
```

The local purge on every device is driven by `observeDeletedClipIds()` in the repository's
`init`, not by the use case, so receivers clean up too.

---

## Related

[[Voice Messages]] - [[Voice Messages - Explained Simply]] - [[Voice Messages - Basic Example]] -
[[Chat and Messaging - Code]] - [[Async and Reactivity]]
