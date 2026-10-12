# Video Messages - Basic Example

The inherited pattern behind [[Video Messages]], in its plain textbook form - not the real
`VideoNoteRecorder` / `VideoNoteBubble`, but the minimal shape they are built on. Read this
first, then [[Video Messages - Code]] to see how the app specialises it.

(Teaching comments only; the real source has none - that is a project rule.)

---

## What is inherited

Two standard pieces: CameraX's `VideoCapture<Recorder>` to film into an `.mp4`, and ExoPlayer
to play it back into a view. Everything else (outbox, encryption, upload) is the same as
[[Voice Messages]].

## The basic example

```kotlin
// 1) Bind a preview and a video capture to a lifecycle.
val provider = ProcessCameraProvider.awaitInstance(context)
val recorder = Recorder.Builder()
    .setQualitySelector(QualitySelector.from(Quality.SD))
    .build()
val capture = VideoCapture.withOutput(recorder)
val preview = Preview.Builder().build().also { it.setSurfaceProvider(previewView.surfaceProvider) }
provider.bindToLifecycle(lifecycleOwner, CameraSelector.DEFAULT_FRONT_CAMERA, preview, capture)

// 2) Record to a file; the file is complete only after the Finalize event.
val recording = recorder.prepareRecording(context, FileOutputOptions.Builder(file).build())
    .withAudioEnabled()
    .start(ContextCompat.getMainExecutor(context)) { event ->
        if (event is VideoRecordEvent.Finalize && !event.hasError()) upload(file)
    }
// ... user releases the button
recording.stop()

// 3) Play it back in a round view.
val player = ExoPlayer.Builder(context).build()
player.setVideoTextureView(textureView)
player.setMediaItem(MediaItem.fromUri(Uri.fromFile(file)))
player.prepare()
player.play()
```

## How the real code specialises it

- `awaitInstance` runs under `NonCancellable`, so letting go during camera start-up cannot
  cancel CameraX's shared init future (`data/media/VideoNoteRecorder.kt:211`).
- The recorder is a `@Singleton` bound to `ProcessLifecycleOwner`, with a bitrate cap, a
  2-minute duration limit and a 38 MB file limit; it is interrupted by calls, backgrounding,
  focus loss and leaving the chat, and every interruption becomes a draft.
- "Upload on Finalize" is replaced by a pure decision, `videoFinalizeOutcome`, plus a draft
  fallback when nobody is waiting for the result (`:87`, `:303`).
- The view is a `TextureView` clipped to a circle with a centre-crop matrix, attached through a
  stack so two surfaces for one clip never freeze each other
  (`data/media/ClipPlaybackController.kt:73`).
- Seeking is done by dragging along the circle's edge (`ringFraction`,
  `presentation/screens/chat/VideoNoteBubble.kt:85`).

## Related

[[Video Messages]] - [[Video Messages - Code]] - [[Video Messages - Explained Simply]] -
[[Voice Messages - Basic Example]] - [[Home]]
