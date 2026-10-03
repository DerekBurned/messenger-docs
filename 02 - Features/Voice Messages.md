# Voice Messages

Hold-to-record voice notes (shipped in 0.1.13.1): record with slide-to-cancel and slide-up lock,
keep interrupted recordings as per-chat drafts, send through a crash-safe outbox with a
confirmed Firestore write, and play back with ExoPlayer under transient audio focus with
auto-advance, a waveform bubble with seek, and a mini player (with 1x/1.5x/2x) that follows you
across tabs. Video clips share the model and the wire format but only render a placeholder
bubble.

## Pieces

- Model: `Message.Clip` (`domain/model/Message.kt:57`) carrying a `Clip`
  (`domain/model/voiceMessage/Clip.kt:6`): `Clip.Voice` (url, duration, size, `fileKey`,
  `waveform`) and `Clip.Video`. Each subtype owns its `previewLabel` ("Voice message" /
  "Video message"), `fileExtension` (`m4a` / `mp4`) and `maxDurationMs` (voice: 1 hour).
- Recording: `VoiceRecorder` (`data/media/VoiceRecorder.kt:77`), an app-wide `@Singleton`
  confined to the main thread, exposing `StateFlow<VoiceRecordState>`.
- Drafts: `VoiceDraftStore` (`data/media/VoiceDraftStore.kt:33`) - one draft per conversation,
  persisted in `filesDir/voice_drafts/`.
- Sending, downloading, recovery, cleanup: `ClipRepositoryImpl`
  (`data/repository/ClipRepositoryImpl.kt:48`) behind `IClipRepository`.
- Playback: `ClipPlaybackController` (`data/media/ClipPlaybackController.kt:80`) - one ExoPlayer,
  exposing `StateFlow<ClipPlaybackState>`.
- Files: `MediaCache` (`clip_outbox`, `voice_drafts`, `chat_media`) and `MediaFileTransfer`
  (encrypted upload/download), see [[Data Layer]].
- UI: `RecordMicButton.kt` (the `recordGesture` modifier), `RecordingBar`, `VoiceDraftBar`,
  `VoiceWaveform`, `ReplyQuote` in `presentation/components/chat/`; `VoiceBubble` and
  `MessageComposer` in `presentation/screens/chat/`; `ClipMiniPlayer`
  (`presentation/components/common/ClipMiniPlayer.kt:64`) with `ClipPlayerViewModel`, mounted in
  `MainDisplay` (`presentation/navigation/graph/MainDisplay.kt:620`).
- Wiring: `ChatIntent` voice intents handled in `ChatViewModel` (`StartRecording`,
  `LockRecording`, `CancelRecording`, `FinishRecording`, `InterruptRecording`, `SendVoiceDraft`,
  `DiscardVoiceDraft`, `MicPermissionDenied`, `DismissMicPrompt`, `PlayClip`, `SeekClip`,
  `RetryClip`, `CancelClipSend`).
- Backend: `firestore.rules` (`hasValidClip`), `storage.rules` (`chatMedia/...`), and the
  `onMessageCreated` function's push preview (`CLIP_PREVIEWS`, `functions/src/index.ts:19`).

## Recording

The mic button carries `Modifier.recordGesture`
(`presentation/components/chat/RecordMicButton.kt`). Touch down calls `StartRecording`;
`ChatViewModel.startRecording` refuses if this chat already has a draft, pauses any playing clip,
and calls `voiceRecorder.start(conversationId)` (`VoiceRecorder.kt:130`).

`start` refuses during a call, then takes `AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE` (refuses if
denied), and configures `MediaRecorder` for AAC in MPEG-4, mono, 44.1 kHz, 64 kbps, with
`setMaxDuration(Clip.Voice.MAX_DURATION_MS)`. A failed `prepare()`/`start()` releases the
recorder and deletes the file. While recording it polls `maxAmplitude` every 100 ms: the state
keeps the last 64 normalised samples for the live bar, and the full list is downsampled to 50
buckets (`downsampleWaveform`) when the recording ends.

Gesture outcomes, decided by `recordGestureEnd(locked, slidToCancel, released)`:

- slide left past 100 dp -> `CancelRecording`, the file is deleted;
- slide up past 80 dp -> `LockRecording`, the finger can lift and the recording continues;
  `RecordingBar` then shows cancel and stop, and both stop and the composer's send button finish
  and send (`MessageComposer.kt:158`, `:212`);
- release -> `FinishRecording` -> `stop(conversationId)` -> send;
- any other end of the gesture (the system cancelling the pointer) -> `InterruptRecording`, which
  keeps a draft.

A recording belongs to one chat: `VoiceRecordState.Recording` carries `conversationId`, and
`visibleTo(conversationId)` (`VoiceRecorder.kt:46`) shows it as `Idle` to every other chat, so a
locked recording in chat A cannot be finished or cancelled from chat B. Recordings shorter than
500 ms are discarded. `MessageComposer` keeps the screen on while recording
(`KeepScreenOn(enabled = uiState.recording is VoiceRecordState.Recording)`).

## Interruptions become drafts

`interrupt()` (`VoiceRecorder.kt:191`) finishes the recording and hands it to
`VoiceDraftStore.save` instead of sending. It fires when:

- the app leaves the foreground (`ProcessLifecycleOwner` drops below `STARTED`);
- a call becomes active (`ActiveCallHolder.state` turns non-null);
- audio focus is lost permanently or transiently (`interruptsRecording`), which is how a normal
  cellular call ends a recording;
- the 1-hour cap is reached (`MEDIA_RECORDER_INFO_MAX_DURATION_REACHED`);
- the gesture ends without a release (see above);
- the chat's ViewModel is cleared (`onCleared` -> `interrupt(conversationId)`).

`VoiceDraftStore.save` moves the file to `filesDir/voice_drafts/<conversationId>.m4a` and writes
a JSON sidecar `{durationMs, waveform}`. `load` runs when a chat opens and drops orphaned or
corrupt drafts, so a draft survives process death. While a draft exists the composer shows
`VoiceDraftBar` (duration, waveform, discard) in place of the text field, and the action button
sends it (`SendVoiceDraft` -> `take` -> the normal send path). Only slide-to-cancel and the
explicit discard throw a recording away.

Known behaviour: pulling down the notification shade while holding the mic does not cancel the
pointer on every device, so the recording simply continues.

## Sending through the outbox

`sendVoice` builds a `Clip.Voice(durationMs, sizeBytes, waveform)` and calls
`clipRepository.sendClip(conversationId, file, clip, replyTo = replyingTo)`.
`sendClip` (`ClipRepositoryImpl.kt:82`):

1. rejects a clip over its duration limit;
2. moves the file into `filesDir/clip_outbox/<messageId>.m4a`, writes a `Message.Clip` row with
   status `SENDING` (including the reply quote from `replyTo.replySnippet()`) and updates the
   conversation preview to "Voice message";
3. `launchSend` registers a lazy `Job` in `sendJobs` and starts it.

`uploadAndSend` (`:236`):

- uploads the outbox file to `chatMedia/<conversationId>/<messageId>/clip.m4a`, encrypted with a
  per-file key when the conversation is encryptable, under
  `withTimeoutOrNull(clipUploadTimeout(size))` - 30 s plus one second per 8000 bytes (`:381`);
- on failure: cancel and clear the transfer, mark `FAILED`;
- on success: `updateClip` writes the URL and key inside an ObjectBox transaction that re-reads
  the row and returns `null` if the clip was deleted meanwhile, in which case the uploaded file is
  deleted from Storage and nothing is sent;
- sends the Firestore document with `confirmWithin = 20.seconds`, so a write that never reaches
  the server becomes `FAILED` instead of hanging in `SENDING`;
- marks `SENT` and promotes the file from the outbox into the cache.

Retry (`RetryClip` -> `retry`, `:187`) atomically claims only a `FAILED`, non-deleted row that has
no live job. Cancel during upload (`CancelClipSend` -> `cancelSend`, `:205`) cancels the job and
transfer and, unless the clip already went out, deletes the row and the outbox file.

## Encryption

A voice clip is encrypted like media: the audio file with a per-file AES-GCM key, and the key
plus the waveform travel inside the encrypted message payload as `ClipSecret(key, waveform)`
(`data/crypto/model/ClipSecret.kt`, built by `Clip.toSecret()` in
`data/crypto/E2eeMessageCodec.kt:119`). The plaintext side of the document keeps the URL,
duration and size but an empty waveform. See [[End-to-End Encryption]].

## Recovery after process death

`ClipRepositoryImpl.init` runs `recoverAfterRestart` (`ClipRepositoryImpl.kt:283`):

- delete stale temporaries (`recording_*`, `*.dl`, `*.part`, `*.enc`) older than this process;
- adopt pre-outbox cache files of unsent clips into the outbox;
- mark `FAILED` every clip still `SENDING` from an earlier process with no live job
  (`strandedClipIds`, `ClipRepositoryImpl.kt:358`), so it shows a retry button;
- reconcile the outbox (`outboxFate`, `ClipRepositoryImpl.kt:371`): keep files of `SENDING`/`FAILED` rows, promote
  files of sent rows into the cache, drop files with no row or a deleted row.

Uploads themselves only survive while the process lives; a killed upload becomes a retry on the
next launch.

## Receiving and playing

The bubble's play button calls `PlayClip` -> `playbackController.toggle(ClipQueueItem(...))`.
`clipToggle` (`ClipPlaybackController.kt:60`) decides PLAY / PAUSE / RESUME / IGNORE from the
state and the clip the player actually holds, so tapping B while A plays switches cleanly and a
tap while loading is ignored.

`play` (`:208`) refuses during a call, unloads the previous clip, shows a loading state and asks
`clipRepository.ensureLocal(id)` (`ClipRepositoryImpl.kt:112`) for a file: the outbox copy, then
the cache copy, else a download. Downloads go through `InFlight`, so concurrent requests for the
same clip share one download; the file is decrypted with `fileKey` and moved into place
atomically. If no file comes back the state records `failedMessageId` and the bubble shows a
retry-play icon; a player error also evicts the cached copy. The player is built once with
speech audio attributes and `setHandleAudioBecomingNoisy(true)` (unplugging headphones pauses),
and the next voice clip is prefetched.

Audio focus is `AUDIOFOCUS_GAIN_TRANSIENT`, so a music app resumes after a note ends.
`focusReaction` (`:69`): a permanent loss pauses and releases focus; a transient loss (including
"can duck" - speech is paused rather than ducked) pauses and resumes on regain, but only if it
was playing. An in-app call pauses playback with no auto-resume.

When a clip ends (`onCompleted`, `:272`) it is marked played (clears the unplayed dot), and the
next voice clip in the same chat by timestamp then id is played - from any sender, including your
own and already-played clips, and including clips that arrived while listening
(`nextVoiceClipAfter`, `ClipRepositoryImpl.kt:362`). Otherwise playback stops.

`VoiceBubble` (`presentation/screens/chat/VoiceBubble.kt`) shows the waveform with seek,
duration, an unplayed dot, a reply quote, and one action button chosen by `voiceButton(...)`
(`:60`): cancel-send with a progress ring while uploading, a ring while downloading, retry-send for
your own `FAILED` clip, a spinner while loading, retry-play after a failure, else pause/play.

`ClipMiniPlayer` sits at the navigation root, so playback continues across tabs. It has
play/pause, close, the speed cycle 1x -> 1.5x -> 2x (`nextClipSpeed`,
`ClipMiniPlayer.kt:51`), and a tap that opens the clip's chat
(`ClipPlayerViewModel.resolveTarget`).

## Deletion and logout

Deleting a voice clip for everyone also removes its Storage file
(`DeleteMessageUseCase` -> `clipRepository.deleteRemoteFile`). Every device watches
`observeDeletedClipIds` and purges its local copy, outbox file, send job and transfer
(`purgeLocal`, `ClipRepositoryImpl.kt:176`); the playback controller stops if its current clip was deleted. Logout
stops playback, cancels any recording, clears drafts, and wipes the outbox, drafts and media
cache (see [[Authentication]]).

## Microphone permission

Tapping the mic without `RECORD_AUDIO` shows a rationale dialog when Android allows it, otherwise
requests the permission. A one-off denial shows a toast; a permanent denial sets `micDenied`, and
`MicPermissionDialog` (`ChatScreenWithNav.kt:267`) offers "Open settings"
(`ACTION_APPLICATION_DETAILS_SETTINGS`).

## Labels, pushes and backend rules

- One label source: `Clip.previewLabel` feeds the chat-list preview (local and in the
  conversation document, also for encrypted chats) and reply quotes. FCM pushes map the message type with `clipLabelFor` to
  `R.string.clip_voice_message`, and `onMessageCreated` uses `CLIP_PREVIEWS` so a voice note's
  push reads "Voice message" even when the text is encrypted.
- `firestore.rules`: `hasValidClip` accepts `VOICE_CLIP` / `VIDEO_CLIP` messages with a `clip`
  map and a URL up to 2048 chars.
- `storage.rules`: `chatMedia/{conversationId}/{messageId}/{fileName}` - read, write and delete
  only for conversation participants (a cross-service `firestore.get` on the conversation), clip
  files up to 40 MB, other media up to 100 MB, `application/octet-stream` or image/video/audio.

## Why this design

- An outbox in `filesDir`, separate from the cache, means an unsent recording is never evicted
  and can be reconciled after a crash; promoting it on success keeps one copy.
- Confirming the Firestore write and re-reading the row inside a transaction stop the two worst
  failure modes: a clip stuck in `SENDING` forever and a deleted clip being resurrected by a late
  upload.
- Drafts instead of silent discards: interruptions (backgrounding, calls, the cap) are not the
  user's choice, so the recording is kept; only explicit gestures throw it away.
- Main-thread confinement for the recorder and player keeps state transitions simple without
  locks; `ExoPlayer` must be used from the thread it was built on.
- Transient focus and no auto-resume after calls follow how the user expects a voice note to
  behave next to music and phone calls.
- Size-scaled timeouts let a one-hour note (about 29 MB at 64 kbps) upload on a slow link,
  unlike the flat 35 s media timeout.

## Known limits

- Upload survives only while the process lives (no WorkManager); a killed upload needs a retry.
- Encryption is whole-file AES-GCM, so a one-hour clip is about 29 MB buffered in memory.
- Not built yet: listened receipts, media-session (lock screen) controls, review before send.

## Visualization

```mermaid
flowchart TD
    MIC[Hold mic] -->|start| REC[VoiceRecorder]
    REC -->|release| SEND[sendVoice]
    REC -->|background, call, focus loss, cap| DRAFT[(VoiceDraftStore)]
    DRAFT -->|Send| SEND
    SEND --> OUT[(clip_outbox) + row SENDING]
    OUT --> UP[Encrypt + upload, size-scaled timeout]
    UP -->|fail| FAILED[FAILED, retry]
    UP --> TX{Row still alive?}
    TX -->|deleted| DEL[Delete uploaded file]
    TX -->|yes| FS[Firestore write, confirmed in 20s]
    FS --> SENT[SENT, promote to cache]
    FS -->|timeout| FAILED
    PEER[Receiver bubble] --> ENS[ensureLocal: outbox, cache or deduplicated download]
    ENS --> PLAY[ExoPlayer, transient focus]
    PLAY -->|ended| NEXT[Mark played, next voice clip]
```

## Related

[[Chat and Messaging]] - [[End-to-End Encryption]] - [[Data Layer]] - [[Async and Reactivity]] -
[[Notifications]] - [[Authentication]] - [[Voice Messages - Code]] -
[[Voice Messages - Explained Simply]] - [[Voice Messages - Basic Example]]
