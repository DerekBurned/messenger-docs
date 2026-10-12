# Video Messages - Code

Companion to [[Video Messages]]. The source of round video notes, block by block. Snippets are
copied from the repo (branch `feat/voice_messages`, 2026-10-12); the "why" lives in the main
note. `path:line` refs are relative to `app/src/main/java/com/example/messenger/`.

Files covered:
- `data/media/VideoNoteRecorder.kt`
- `data/media/ClipPlaybackController.kt` (video surface)
- `presentation/screens/chat/VideoNoteBubble.kt`
- `presentation/components/chat/RecordMicButton.kt` (tap vs hold)

---

## 1. Ending a recording is a pure decision

`data/media/VideoNoteRecorder.kt:70`:

```kotlin
internal enum class VideoEnd { SEND, DRAFT, DISCARD }

private const val MIN_VIDEO_DURATION_MS = 1_000L

internal fun nextVideoEnd(current: VideoEnd?, requested: VideoEnd): VideoEnd =
    if (current == null || requested == VideoEnd.DISCARD) requested else current

internal fun videoEndWithoutWaiter(current: VideoEnd?): VideoEnd? =
    if (current == VideoEnd.SEND) VideoEnd.DRAFT else current

private val USABLE_FINALIZE_ERRORS = setOf(
    VideoRecordEvent.Finalize.ERROR_NONE,
    VideoRecordEvent.Finalize.ERROR_DURATION_LIMIT_REACHED,
    VideoRecordEvent.Finalize.ERROR_FILE_SIZE_LIMIT_REACHED,
    VideoRecordEvent.Finalize.ERROR_SOURCE_INACTIVE,
)

internal fun videoFinalizeOutcome(requested: VideoEnd?, error: Int, fileBytes: Long, durationMs: Long): VideoEnd = when {
    requested == VideoEnd.DISCARD -> VideoEnd.DISCARD
    fileBytes <= 0L || durationMs < MIN_VIDEO_DURATION_MS || error !in USABLE_FINALIZE_ERRORS -> VideoEnd.DISCARD
    else -> requested ?: VideoEnd.DRAFT
}
```

- Several end requests can arrive before CameraX's asynchronous `Finalize`: a release (`SEND`),
  then the app going to the background (`DRAFT`), then logout (`DISCARD`). `nextVideoEnd` keeps
  the first one, except that a cancel always overrides.
- `SOURCE_INACTIVE` (the camera closed under us, e.g. backgrounding) and the two limit errors
  still leave a valid file, so they count as usable. With no explicit request (the finger is still
  down when the 2-minute cap hits) the outcome is a draft, never a send.
- All three are covered by `VideoFinalizeOutcomeTest`.

## 2. Binding the camera without breaking CameraX

`VideoNoteRecorder.kt:209`:

```kotlin
@SuppressLint("MissingPermission")
private suspend fun bindAndRecord(active: Session) {
    try {
        val provider = withContext(NonCancellable) { ProcessCameraProvider.awaitInstance(context) }
        if (session !== active) return
        val selector = if (provider.hasCamera(CameraSelector.DEFAULT_FRONT_CAMERA)) {
            CameraSelector.DEFAULT_FRONT_CAMERA
        } else {
            CameraSelector.DEFAULT_BACK_CAMERA
        }
        val recorder = Recorder.Builder()
            .setQualitySelector(
                QualitySelector.from(Quality.SD, FallbackStrategy.lowerQualityOrHigherThan(Quality.SD)),
            )
            .setTargetVideoEncodingBitRate(VIDEO_BIT_RATE)
            .build()
        val capture = VideoCapture.Builder(recorder)
            .setMirrorMode(MirrorMode.MIRROR_MODE_ON_FRONT_ONLY)
            .build()
        val preview = Preview.Builder().build()
        preview.setSurfaceProvider(surfaceProvider)
        provider.bindToLifecycle(ProcessLifecycleOwner.get(), selector, preview, capture)
        active.provider = provider
        active.preview = preview
        active.capture = capture
        val output = FileOutputOptions.Builder(active.file)
            .setDurationLimitMillis(Clip.Video.MAX_DURATION_MS)
            .setFileSizeLimit(MAX_FILE_BYTES)
            .build()
        active.recording = recorder.prepareRecording(context, output)
            .withAudioEnabled()
            .start(ContextCompat.getMainExecutor(context)) { event -> onEvent(active, event) }
    } catch (e: CancellationException) {
        if (!currentCoroutineContext().isActive) throw e
        Log.w(TAG, "camera provider was cancelled", e)
        complete(active, VideoRecordEvent.Finalize.ERROR_UNKNOWN)
    } catch (e: Exception) {
        Log.w(TAG, "video note recording failed to start", e)
        complete(active, VideoRecordEvent.Finalize.ERROR_UNKNOWN)
    }
}
```

- `awaitInstance` is `getInstance(context).await()`. Cancelling that `await()` cancels the future,
  and in CameraX 1.4.2 the cancel reaches the process-wide init future. `NonCancellable` keeps a
  quick release from poisoning CameraX for the rest of the process; the `session !== active`
  check then drops the result if the user already let go.
- A `CancellationException` while this coroutine is still active means the future was cancelled
  by someone else; it is treated as a start failure so the UI never stays in `Recording`.
- `bindToLifecycle(ProcessLifecycleOwner.get(), ...)` ties the camera to app visibility instead
  of an Activity. The preview's surface provider comes from the overlay through
  `attachPreview` / `detachPreview`, in whichever order the binding and the overlay happen.

## 3. A send nobody waits for becomes a draft

`VideoNoteRecorder.kt:183` and `:303`:

```kotlin
suspend fun stop(conversationId: String): ClipRecording? {
    val active = session ?: return null
    if (active.conversationId != conversationId) return null
    end(active, VideoEnd.SEND)
    var claimed: ClipRecording? = null
    try {
        claimed = withTimeoutOrNull(FINALIZE_TIMEOUT_MS) { active.result.await() }
        return claimed
    } finally {
        if (claimed == null) keepUnclaimedAsDraft(active) else active.unclaimed = null
    }
}

private fun keepUnclaimedAsDraft(active: Session) {
    val unclaimed = active.unclaimed
    if (unclaimed != null) {
        active.unclaimed = null
        drafts.save(unclaimed)
    } else if (!active.result.isCompleted) {
        active.requested = videoEndWithoutWaiter(active.requested)
    }
}
```

`ChatViewModel.finishRecording` calls `stop` inside `viewModelScope`. If the user leaves the chat
before `Finalize`, that scope is cancelled; if `Finalize` takes longer than 5 s the timeout fires.
Either way `claimed` stays null and the `finally` keeps the clip: a result that already arrived
(`unclaimed`) is saved as a draft, and a pending `SEND` becomes `DRAFT` for when `Finalize`
arrives. `result` is a `CompletableDeferred<ClipRecording?>` completed exactly once by `complete`.

## 4. One player, several possible surfaces

`data/media/ClipPlaybackController.kt:73` and `:224`:

```kotlin
internal class VideoViewStack<T : Any> {
    private val views = mutableListOf<T>()

    val current: T? get() = views.lastOrNull()

    fun attach(view: T) {
        views.remove(view)
        views.add(view)
    }

    fun detach(view: T) {
        views.remove(view)
    }
}

fun attachVideo(view: TextureView) {
    videoViews.attach(view)
    player?.setVideoTextureView(view)
}

fun detachVideo(view: TextureView) {
    val wasShown = videoViews.current === view
    videoViews.detach(view)
    if (!wasShown) return
    player?.clearVideoTextureView(view)
    videoViews.current?.let { player?.setVideoTextureView(it) }
}
```

ExoPlayer renders into one view at a time. During recomposition, scrolling or a context-menu
preview a second surface can briefly exist for the same clip. With a single field, detaching the
newer view cleared the output and froze the list bubble's picture. The stack hands the video back
to the view underneath, whatever order the attach and detach calls come in.

## 5. Ring seek vs tap on the bubble

`presentation/screens/chat/VideoNoteBubble.kt:223`:

```kotlin
.pointerInput(interactive, isActive) {
    if (!interactive) return@pointerInput
    awaitEachGesture {
        val down = awaitFirstDown(requireUnconsumed = false)
        val cx = size.width / 2f
        val cy = size.height / 2f
        val radius = size.width / 2f - ringInsetPx
        val onRing = isActive &&
            isOnRing(down.position.x, down.position.y, cx, cy, radius, ringBandPx)
        if (onRing) {
            down.consume()
            seek(ringFraction(down.position.x, down.position.y, cx, cy))
            while (true) {
                val change = awaitPointerEvent().changes.firstOrNull { it.id == down.id } ?: break
                change.consume()
                if (!change.pressed) break
                seek(ringFraction(change.position.x, change.position.y, cx, cy))
            }
        } else {
            val up = waitForUpOrCancellation()
            if (up != null && up.uptimeMillis - down.uptimeMillis < viewConfiguration.longPressTimeoutMillis) {
                up.consume()
                tap()
            }
        }
    }
},
```

The geometry (`:85`):

```kotlin
internal fun ringFraction(x: Float, y: Float, cx: Float, cy: Float): Float {
    val turns = atan2(x - cx, -(y - cy)) / (2f * PI.toFloat())
    return (turns + 1f) % 1f
}

internal fun isOnRing(x: Float, y: Float, cx: Float, cy: Float, radius: Float, bandPx: Float): Boolean =
    abs(hypot(x - cx, y - cy) - radius) <= bandPx
```

`atan2(dx, -dy)` puts 0 at 12 o'clock and grows clockwise. A ring drag consumes every move; this
only stops the bubble from moving because `SwipeToReply` now refuses to claim an
already-consumed gesture (`presentation/components/chat/ChatSwipeGestures.kt:103`,
`if (change.isConsumed) break`). Off the ring nothing is consumed until the up, so long press
and swipe-to-reply still reach the wrappers.

## 6. Tap or hold on the mic

`presentation/components/chat/RecordMicButton.kt:60`:

```kotlin
val early = withTimeoutOrNull(HOLD_TO_RECORD_MS) {
    var outcome: EarlyEnd? = null
    while (outcome == null) {
        val change = awaitPointerEvent().changes.firstOrNull { it.id == down.id }
        outcome = when {
            change == null -> EarlyEnd.LOST
            !change.pressed -> EarlyEnd.TAP
            else -> null
        }
        change?.consume()
    }
    outcome
}
when (early) {
    EarlyEnd.TAP -> {
        tap()
        return@awaitEachGesture
    }
    EarlyEnd.LOST -> return@awaitEachGesture
    null -> Unit
}
start()
```

`withTimeoutOrNull` here is the `AwaitPointerEventScope` member, driven by input time. A release
within 200 ms is a tap and returns before `start()`, so a tap can never turn on the camera or the
microphone. `RecordGestureTest` pins this down.

## Related

[[Video Messages]] - [[Video Messages - Explained Simply]] - [[Video Messages - Basic Example]] -
[[Voice Messages - Code]] - [[Async and Reactivity]]
