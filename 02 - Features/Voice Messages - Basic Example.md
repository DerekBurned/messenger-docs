# Voice Messages - Basic Example

The inherited pattern behind [[Voice Messages]], in its plain textbook form - not the real
`VoiceRecorder`/`ClipRepositoryImpl`/`ClipPlaybackController`, but the minimal shape they are
built on. Read this first, then [[Voice Messages - Code]] to see how the app specialises it.

(Teaching comments only; the real source has none - that is a project rule.)

---

## What is inherited

Three standard Android pieces: `MediaRecorder` to capture AAC audio into an `.m4a` file, an
upload of that file to Firebase Storage followed by a Firestore message that points at it, and
`ExoPlayer` to play it back while holding audio focus. On top sits one app pattern: **the local
row is the truth**, written first and updated as the upload progresses, so the UI never waits
on the network.

## The basic example

```kotlin
// 1) Record.
val recorder = MediaRecorder(context).apply {
    setAudioSource(MediaRecorder.AudioSource.MIC)
    setOutputFormat(MediaRecorder.OutputFormat.MPEG_4)
    setAudioEncoder(MediaRecorder.AudioEncoder.AAC)
    setOutputFile(file.absolutePath)
    prepare()
    start()
}
// ... user releases the button
recorder.stop()
recorder.release()

// 2) Upload, then post a message that points at the file.
val ref = Firebase.storage.reference.child("voice/${UUID.randomUUID()}.m4a")
ref.putFile(Uri.fromFile(file)).await()
val url = ref.downloadUrl.await().toString()
Firebase.firestore.collection("messages").add(mapOf("type" to "VOICE", "url" to url)).await()

// 3) Play it back, politely asking for audio focus first.
val player = ExoPlayer.Builder(context).build()
player.setAudioAttributes(speechAttributes, /* handleAudioFocus = */ true)
player.setMediaItem(MediaItem.fromUri(url))
player.prepare()
player.play()
```

## How the real code specialises it

- Recording is a `@Singleton` confined to the main thread, tied to one chat, with a 1-hour
  `setMaxDuration`, amplitude polling for the waveform, and automatic interruption on calls,
  backgrounding and focus loss - interruptions save a draft instead of losing the tape
  (`data/media/VoiceRecorder.kt:191`).
- The file is staged in `filesDir/clip_outbox` and a local `SENDING` row is written before any
  network call; the upload is encrypted per file when the chat is encrypted, bounded by a
  size-scaled timeout, and the Firestore write must be confirmed within 20 s
  (`data/repository/ClipRepositoryImpl.kt:236`).
- The local row is re-read in a transaction before the message is posted, so a clip deleted
  mid-upload is never resurrected; stranded clips are marked `FAILED` after a process restart.
- Playback plays a local file (outbox, cache, or one deduplicated download), manages audio focus
  manually as `AUDIOFOCUS_GAIN_TRANSIENT` instead of letting ExoPlayer hold it, pauses for calls
  and headphone unplugging, and auto-advances to the next voice note
  (`data/media/ClipPlaybackController.kt:272`).

## Related

[[Voice Messages]] - [[Voice Messages - Code]] - [[Voice Messages - Explained Simply]] -
[[Reading the Code - Simple Patterns]] - [[Chat and Messaging - Basic Example]] - [[Home]]
