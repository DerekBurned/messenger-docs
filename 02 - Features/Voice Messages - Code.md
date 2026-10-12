# Voice Messages - Code

Companion to [[Voice Messages]]. The source of the voice pipeline, block by block. Snippets are
copied from the repo with `Log` lines trimmed; the "why" lives in the main note.

Files covered:
- `domain/model/voiceMessage/Clip.kt`
- `data/media/VoiceRecorder.kt`, `data/media/VoiceDraftStore.kt`
- `presentation/components/chat/RecordMicButton.kt`, `LiveRecordingWaveform.kt`, `VoiceWaveform.kt`
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

Waveform levels (`VoiceRecorder.kt:78`), on a decibel scale and normalised to the peak:

```kotlin
private const val FULL_SCALE_AMPLITUDE = 32_767.0
private const val SILENCE_FLOOR_DB = -50.0

internal fun amplitudeToLevel(amplitude: Int): Int {
    if (amplitude <= 0) return 0
    val db = 20.0 * log10(amplitude / FULL_SCALE_AMPLITUDE)
    return ((db - SILENCE_FLOOR_DB) / -SILENCE_FLOOR_DB * 100.0).roundToInt().coerceIn(0, 100)
}

fun normalizeWaveform(levels: List<Int>): List<Int> {
    val peak = levels.maxOrNull() ?: return levels
    if (peak <= 0 || peak >= 100) return levels.map { it.coerceIn(0, 100) }
    return levels.map { (it.coerceAtLeast(0) * 100f / peak).roundToInt() }
}
```

A quiet phone microphone peaking at amplitude 1000 was level 3 under the old linear mapping and
is level 39 here; `normalizeWaveform` then stretches the whole note so its loudest bucket is 100.
`VoiceWaveform` runs `normalizeWaveform` on stored waveforms too
(`presentation/components/chat/VoiceWaveform.kt:28`), which is what repairs notes already sent
with the old scale. Covered by `WaveformTest`.

Finishing versus interrupting (`VoiceRecorder.kt:201`):

```kotlin
fun stop(conversationId: String): ClipRecording? {
    if (owner != conversationId) return null
    return finish()
}

fun interrupt(conversationId: String? = null) {
    if (recorder == null) return
    if (conversationId != null && owner != conversationId) return
    finish()?.let(drafts::save)
}

private fun finish(): ClipRecording? {
    val active = recorder
    val file = outputFile
    val conversationId = owner
    val durationMs = recordedDurationMs(startedAt, SystemClock.elapsedRealtime())
    val waveform = normalizeWaveform(downsampleWaveform(samples.toList()))
    release(active)
    if (active == null || file == null || conversationId == null) return null
    if (!file.exists() || file.length() == 0L || durationMs < MIN_DURATION_MS) {
        file.delete()
        return null
    }
    return ClipRecording(
        conversationId = conversationId,
        file = file,
        durationMs = durationMs,
        waveform = waveform,
    )
}
```

`stop` and `interrupt` share `finish`; the only difference is where the result goes - to
`sendRecording`, or to the draft store. `owner` makes a recording belong to one chat.
`ClipRecording` defaults `kind` to `ClipKind.VOICE`; the video recorder produces the same type
with `kind = VIDEO` (see [[Video Messages - Code]]).

The live bar (`presentation/components/chat/LiveRecordingWaveform.kt:25`):

```kotlin
internal fun liveWaveformOffset(barCount: Int, pitchPx: Float, widthPx: Float): Float =
    (barCount * pitchPx - widthPx).coerceAtLeast(0f)
```

Bars are drawn at a fixed pitch at `index * pitch - scroll`, and `scroll` is an `Animatable`
animated to this offset over one sample interval, so bars grow from the left and then glide left
instead of being re-spread across the width.

---

## 3. Drafts

`data/media/VoiceDraftStore.kt`:

```kotlin
@Serializable
internal data class VoiceDraftMeta(
    val durationMs: Long,
    val waveform: List<Int> = emptyList(),
    val kind: ClipKind = ClipKind.VOICE,
)

fun save(recording: ClipRecording) {
    val media = mediaFileFor(recording.conversationId, recording.kind)
    ClipKind.entries
        .filter { it != recording.kind }
        .forEach { mediaFileFor(recording.conversationId, it).delete() }
    if (!mediaCache.move(recording.file, media)) {
        recording.file.delete()
        return
    }
    runCatching {
        metaFileFor(recording.conversationId)
            .writeText(VoiceDraftMeta(recording.durationMs, recording.waveform, recording.kind).encode())
    }
    drafts.update { it + (recording.conversationId to recording.copy(file = media)) }
}

fun take(conversationId: String): ClipRecording? {
    val draft = drafts.value[conversationId] ?: return null
    drafts.update { it - conversationId }
    metaFileFor(conversationId).delete()
    return draft
}
```

`load(conversationId)` reads `<id>.json` first, then the media file for that kind
(`voice_drafts/<id>.m4a` or `<id>.mp4`), and deletes both if either is missing or corrupt. A
sidecar written before video support has no `kind` and decodes as `VOICE`
(`VoiceDraftMetaTest`).

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

Before any of that, the first 200 ms decide tap versus hold (a tap toggles voice/video and
returns before `start()`; see section 6 of [[Video Messages - Code]]). Drags are measured in root
coordinates so a composer that moves under the finger cannot eat the slide:

```kotlin
val origin = remember { arrayOf(Offset.Zero) }
...
.onGloballyPositioned { coordinates -> origin[0] = coordinates.positionInRoot() }
...
val downInRoot = down.position + origin[0]
...
val delta = change.position + origin[0] - downInRoot
```

`dragProgress(distancePx, thresholdPx)` (`RecordMicButton.kt:25`) turns the same delta into the
0..1 progress that drives the lock rail's padlock and the fading "Slide to cancel" label.

---

## 5. Sending

`ClipRepositoryImpl.uploadAndSend` (`data/repository/ClipRepositoryImpl.kt:244`):

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

Auto-advance (`ClipPlaybackController.kt:319`):

```kotlin
private fun onCompleted() {
    val finished = loadedMessageId ?: return
    stopProgressUpdates()
    advanceJob?.cancel()
    advanceJob = scope.launch {
        clipRepository.markPlayed(finished)
        if (!shouldAutoAdvance(_state.value)) {
            if (_state.value.messageId == finished) stop()
            return@launch
        }
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
`nextVoiceClip` was querying. `shouldAutoAdvance` (`:88`) is `!finished.isVideo`: a round video
note stops at its end instead of rolling into the next voice note.

---

## 8. The bubble's button

`presentation/screens/chat/VoiceBubble.kt:58` (also reused by the round video bubble):

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
[[Video Messages - Code]] - [[Chat and Messaging - Code]] - [[Async and Reactivity]]
