# Voice Messages

Hold-to-record voice notes (shipped in 0.1.13.1, recording UX and waveform reworked on branch
`feat/voice_messages` on 2026-10-12): a quick tap switches the mic between voice and video, holding
records with a live waveform that grows left to right, slide left cancels, sliding up a visible
lock rail locks; interrupted recordings are kept as per-chat drafts; sending goes through a
crash-safe outbox with a confirmed Firestore write; playback uses ExoPlayer under transient audio
focus with auto-advance, a waveform bubble with seek, and a mini player (1x/1.5x/2x) that follows
you across tabs. Round video notes share all of this and are documented in [[Video Messages]].

`path:line` refs are relative to `app/src/main/java/com/example/messenger/`.

## Pieces

- Model: `Message.Clip` (`domain/model/Message.kt:57`) carrying a `Clip`
  (`domain/model/voiceMessage/Clip.kt:6`): `Clip.Voice` (url, duration, size, `fileKey`,
  `waveform`) and `Clip.Video`. Each subtype owns its `previewLabel` ("Voice message" /
  "Video message"), `fileExtension` (`m4a` / `mp4`) and `maxDurationMs` (voice: 1 hour).
  `ClipKind` (`domain/model/voiceMessage/ClipKind.kt`) names the two kinds for recording code.
- Recording: `VoiceRecorder` (`data/media/VoiceRecorder.kt:100`), an app-wide `@Singleton`
  confined to the main thread, exposing `StateFlow<VoiceRecordState>`. A finished take is a
  `ClipRecording(conversationId, file, durationMs, waveform, kind)` (`:55`).
- Drafts: `VoiceDraftStore` (`data/media/VoiceDraftStore.kt:34`) - one draft per conversation,
  voice or video, persisted in `filesDir/voice_drafts/`.
- Sending, downloading, recovery, cleanup: `ClipRepositoryImpl`
  (`data/repository/ClipRepositoryImpl.kt:48`) behind `IClipRepository`.
- Playback: `ClipPlaybackController` (`data/media/ClipPlaybackController.kt:103`) - one ExoPlayer,
  exposing `StateFlow<ClipPlaybackState>`.
- Files: `MediaCache` (`clip_outbox`, `voice_drafts`, `chat_media`) and `MediaFileTransfer`
  (encrypted upload/download), see [[Data Layer]].
- UI: `RecordMicButton.kt` (the `recordGesture` modifier), `RecordingBar`,
  `LiveRecordingWaveform`, `RecordLockRail`, `VoiceDraftBar`, `VoiceWaveform`, `ReplyQuote` in
  `presentation/components/chat/`; `VoiceBubble` and `MessageComposer` in
  `presentation/screens/chat/`; `ClipMiniPlayer`
  (`presentation/components/common/ClipMiniPlayer.kt:64`) with `ClipPlayerViewModel`, mounted in
  `MainDisplay` (`presentation/navigation/graph/MainDisplay.kt:620`).
- Wiring: `ChatIntent` intents handled in `ChatViewModel` (`StartRecording`, `LockRecording`,
  `CancelRecording`, `FinishRecording`, `InterruptRecording`, `SendVoiceDraft`,
  `DiscardVoiceDraft`, `ToggleClipMode`, `MicPermissionDenied`, `DismissMicPrompt`, `PlayClip`,
  `SeekClip`, `RetryClip`, `CancelClipSend`). Each recording intent goes to whichever recorder
  (voice or video) is active.
- Backend: `firestore.rules` (`hasValidClip`), `storage.rules` (`chatMedia/...`), and the
  `onMessageCreated` function's push preview (`CLIP_PREVIEWS`, `functions/src/index.ts:19`).

## Recording

### Tap or hold

The mic button carries `Modifier.recordGesture`
(`presentation/components/chat/RecordMicButton.kt:31`). After touch-down it waits up to 200 ms
(`withTimeoutOrNull(HOLD_TO_RECORD_MS)`, `:60`):

- released inside the window -> a tap: `ToggleClipMode` switches voice <-> video and a hint
  bubble explains the mode for 2 s; nothing is recorded;
- pointer lost inside the window -> nothing happens;
- still held after 200 ms -> `StartRecording`.

Drag deltas are measured in root coordinates (`positionInRoot`, `:54`), so the gesture still works
if the composer moves under the finger. That was a real bug: the lock rail made the composer
taller, the mic jumped up 57 px under a 120 px slide, the lock never engaged, and releasing sent
the note. The rail and the hint are now laid out by `floatAboveEnd`
(`presentation/screens/chat/MessageComposer.kt:372`), a `layout` modifier that reports a 0 x 0
size so floating overlays never resize the composer.

### The recorder

`ChatViewModel.startRecording` refuses if this chat already has a draft, pauses any playing clip,
and calls `voiceRecorder.start(conversationId)` (`VoiceRecorder.kt:152`). `start` refuses during
a call, then takes `AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE` (refuses if denied), and configures
`MediaRecorder` for AAC in MPEG-4, mono, 44.1 kHz, 64 kbps, with
`setMaxDuration(Clip.Voice.MAX_DURATION_MS)`. A failed `prepare()`/`start()` releases the recorder
and deletes the file.

### Waveform levels

`pollAmplitude` (`:265`) reads `maxAmplitude` every 100 ms (`VOICE_SAMPLE_INTERVAL_MS`, `:65`) and
maps it with `amplitudeToLevel` (`:81`): decibels relative to full scale, with -50 dB as silence,
scaled to 0..100. The state keeps the last 256 levels plus `sampleCount` for the live bar. When
the recording ends the levels are downsampled to 50 buckets (`downsampleWaveform`, `:67`) and
peak-normalised (`normalizeWaveform`, `:87`) so the loudest bucket is 100.

`VoiceWaveform` (`presentation/components/chat/VoiceWaveform.kt:28`) also runs
`normalizeWaveform` on every stored waveform before drawing. This repairs notes recorded before
2026-10-12: the old code mapped amplitude linearly (`amp * 100 / 32767`), so phones with quiet
microphones produced levels of 3 to 12 that all sat on the 12% minimum bar height and looked flat,
while a louder tablet looked fine. Peak-normalising old and new waveforms alike shows the real
shape on every device; for new recordings it is a no-op.

### Live waveform

`LiveRecordingWaveform` (`presentation/components/chat/LiveRecordingWaveform.kt:29`) draws bars at
a fixed 5 dp pitch (3 dp bar + 2 dp gap). Bar `i` (absolute index from `sampleCount`) sits at
`i * pitch - scroll`; `scroll` is an `Animatable` animated to `liveWaveformOffset` (`:25`) over
100 ms with linear easing, so bars grow from the left edge and, once the row is full, the strip
glides left with the newest bar at the right edge. The newest bar's height eases in over the same
100 ms. The previous implementation divided the whole width by the visible sample count, so bars
started huge and shrank with every sample.

### Gesture outcomes

Decided by `recordGestureEnd(locked, slidToCancel, released)` (`RecordMicButton.kt:19`):

- slide left past 100 dp -> `CancelRecording`, the file is deleted; the "Slide to cancel" label
  follows the finger and fades with `dragProgress` (`:25`);
- slide up past 80 dp -> `LockRecording`; while sliding, `RecordLockRail`
  (`presentation/components/chat/RecordLockRail.kt:40`) fills from the bottom towards a padlock
  whose shackle closes with the same progress, with a bobbing chevron as the idle hint;
  `RecordingBar` then shows discard and stop, and both stop and the send button finish and send
  (`MessageComposer.kt:257`);
- release -> `FinishRecording` -> `stop(conversationId)` -> send;
- any other end of the gesture (the system cancelling the pointer) -> `InterruptRecording`, which
  keeps a draft.

A recording belongs to one chat: `VoiceRecordState.Recording` carries `conversationId`, and
`visibleTo(conversationId)` (`VoiceRecorder.kt:50`) shows it as `Idle` to every other chat, so a
locked recording in chat A cannot be finished or cancelled from chat B. Recordings shorter than
500 ms are discarded. `MessageComposer` keeps the screen on while either recorder is recording
(`KeepScreenOn(enabled = recordingActive)`, `:103`).

## Interruptions become drafts

`interrupt()` (`VoiceRecorder.kt:213`) finishes the recording and hands it to
`VoiceDraftStore.save` instead of sending. It fires when:

- the app leaves the foreground (`ProcessLifecycleOwner` drops below `STARTED`);
- a call becomes active (`ActiveCallHolder.state` turns non-null);
- audio focus is lost permanently or transiently (`interruptsRecording`), which is how a normal
  cellular call ends a recording;
- the 1-hour cap is reached (`MEDIA_RECORDER_INFO_MAX_DURATION_REACHED`);
- the gesture ends without a release (see above);
- the chat screen leaves composition for anything but a configuration change, e.g. opening the
  partner's profile on top of the chat
  (`presentation/screens/chat/ChatScreenWithNav.kt:166`);
- the chat's ViewModel is cleared (`onCleared` -> `interrupt(conversationId)`).

`VoiceDraftStore.save` moves the file to `filesDir/voice_drafts/<conversationId>.m4a` and writes a
JSON sidecar `VoiceDraftMeta(durationMs, waveform, kind)` (`VoiceDraftStore.kt:20`); video drafts
use `.mp4` and `kind = VIDEO`, and a sidecar written before video support decodes as voice. `load`
runs when a chat opens and drops orphaned or corrupt drafts, so a draft survives process death.
While a draft exists the composer shows `VoiceDraftBar` (duration, waveform or a round video
thumbnail, discard) in place of the text field, and the action button sends it
(`SendVoiceDraft` -> `take` -> the normal send path). Only slide-to-cancel and the explicit
discard throw a recording away.

Known behaviour: pulling down the notification shade while holding the mic does not cancel the
pointer on every device, so the recording simply continues.

## Sending through the outbox

`sendRecording` builds a `Clip.Voice(durationMs, sizeBytes, waveform)` (or a `Clip.Video`, see
[[Video Messages]]) and calls `clipRepository.sendClip(conversationId, file, clip, replyTo)`.
`sendClip` (`ClipRepositoryImpl.kt:82`):

1. rejects a clip over its duration limit;
2. moves the file into `filesDir/clip_outbox/<messageId>.m4a`, writes a `Message.Clip` row with
   status `SENDING` (including the reply quote from `replyTo.replySnippet()`) and updates the
   conversation preview to "Voice message";
3. `launchSend` registers a lazy `Job` in `sendJobs` and starts it.

`uploadAndSend` (`:244`):

- uploads the outbox file to `chatMedia/<conversationId>/<messageId>/clip.m4a`, encrypted with a
  per-file key when the conversation is encryptable, under
  `withTimeoutOrNull(clipUploadTimeout(size))` - 30 s plus one second per 8000 bytes (`:389`);
- on failure: cancel and clear the transfer, mark `FAILED`;
- on success: `updateClip` writes the URL and key inside an ObjectBox transaction that re-reads
  the row and returns `null` if the clip was deleted meanwhile, in which case the uploaded file is
  deleted from Storage and nothing is sent;
- sends the Firestore document with `confirmWithin = 20.seconds`, so a write that never reaches
  the server becomes `FAILED` instead of hanging in `SENDING`;
- marks `SENT` and promotes the file from the outbox into the cache.

Retry (`RetryClip` -> `retry`, `:195`) atomically claims only a `FAILED`, non-deleted row that has
no live job. Cancel during upload (`CancelClipSend` -> `cancelSend`, `:213`) cancels the job and
transfer and, unless the clip already went out, deletes the row and the outbox file.

## Encryption

A voice clip is encrypted like media: the audio file with a per-file AES-GCM key, and the key
plus the waveform travel inside the encrypted message payload as `ClipSecret(key, waveform)`
(`data/crypto/model/ClipSecret.kt`, built by `Clip.toSecret()` in
`data/crypto/E2eeMessageCodec.kt:119`). The plaintext side of the document keeps the URL,
duration and size but an empty waveform. See [[End-to-End Encryption]].

## Recovery after process death

`ClipRepositoryImpl.init` runs `recoverAfterRestart` (`ClipRepositoryImpl.kt:291`):

- delete stale temporaries (`recording_*`, `*.dl`, `*.part`, `*.enc`) older than this process;
- adopt pre-outbox cache files of unsent clips into the outbox;
- mark `FAILED` every clip still `SENDING` from an earlier process with no live job
  (`strandedClipIds`, `ClipRepositoryImpl.kt:366`), so it shows a retry button;
- reconcile the outbox (`outboxFate`, `ClipRepositoryImpl.kt:379`): keep files of
  `SENDING`/`FAILED` rows, promote files of sent rows into the cache, drop files with no row or a
  deleted row.

Uploads themselves only survive while the process lives; a killed upload becomes a retry on the
next launch.

## Receiving and playing

The bubble's play button calls `PlayClip` -> `playbackController.toggle(ClipQueueItem(...))`.
`clipToggle` (`ClipPlaybackController.kt:66`) decides PLAY / PAUSE / RESUME / IGNORE from the
state and the clip the player actually holds, so tapping B while A plays switches cleanly and a
tap while loading is ignored.

`play` (`:253`) refuses during a call, unloads the previous clip, shows a loading state and asks
`clipRepository.ensureLocal(id)` (`ClipRepositoryImpl.kt:112`) for a file: the outbox copy, then
the cache copy, else a download. Downloads go through `InFlight`, so concurrent requests for the
same clip share one download; the file is decrypted with `fileKey` and moved into place
atomically. If no file comes back the state records `failedMessageId` and the bubble shows a
retry-play icon; a player error also evicts the cached copy. The player is built once with speech
audio attributes and `setHandleAudioBecomingNoisy(true)` (unplugging headphones pauses), and the
next voice clip is prefetched.

Audio focus is `AUDIOFOCUS_GAIN_TRANSIENT`, so a music app resumes after a note ends.
`focusReaction` (`:92`): a permanent loss pauses and releases focus; a transient loss (including
"can duck" - speech is paused rather than ducked) pauses and resumes on regain, but only if it
was playing. An in-app call pauses playback with no auto-resume.

When a clip ends (`onCompleted`, `:319`) it is marked played (clears the unplayed dot), and - for
voice only (`shouldAutoAdvance`, `:88`) - the next voice clip in the same chat by timestamp then
id is played, from any sender, including your own and already-played clips, and including clips
that arrived while listening (`nextVoiceClipAfter`, `ClipRepositoryImpl.kt:370`). Otherwise
playback stops.

`VoiceBubble` (`presentation/screens/chat/VoiceBubble.kt`) shows the waveform with seek, duration,
an unplayed dot, a reply quote, and one action button chosen by `voiceButton(...)` (`:60`):
cancel-send with a progress ring while uploading, a ring while downloading, retry-send for your own
`FAILED` clip, a spinner while loading, retry-play after a failure, else pause/play. Dragging the
waveform to seek no longer also starts swipe-to-reply: `SwipeToReply` does not claim a gesture a
child already consumed (`presentation/components/chat/ChatSwipeGestures.kt:103`).

`ClipMiniPlayer` sits at the navigation root, so playback continues across tabs. It has
play/pause, close, the speed cycle 1x -> 1.5x -> 2x (`nextClipSpeed`,
`ClipMiniPlayer.kt:51`), a "Voice message" or "Video message" label from
`ClipPlaybackState.isVideo`, and a tap that opens the clip's chat
(`ClipPlayerViewModel.resolveTarget`).

## Deletion and logout

Deleting a voice clip for everyone also removes its Storage file
(`DeleteMessageUseCase` -> `clipRepository.deleteRemoteFile`). Every device watches
`observeDeletedClipIds` and purges its local copy, outbox file, send job and transfer
(`purgeLocal`, `ClipRepositoryImpl.kt:184`); the playback controller stops if its current clip was
deleted. Logout stops playback, cancels any voice or video recording, clears drafts, and wipes the
outbox, drafts and media cache (see [[Authentication]]).

## Microphone permission

Tapping the mic without `RECORD_AUDIO` (or, in video mode, `CAMERA` + `RECORD_AUDIO`) shows a
rationale dialog when Android allows it, otherwise requests the missing permissions. A one-off
denial shows a toast; a permanent denial sets `micDenied`, and `MicPermissionDialog`
(`ChatScreenWithNav.kt:295`) offers "Open settings" (`ACTION_APPLICATION_DETAILS_SETTINGS`), with
camera wording when the request came from video mode.

## Labels, pushes and backend rules

- One label source: `Clip.previewLabel` feeds the chat-list preview (local and in the
  conversation document, also for encrypted chats) and reply quotes. FCM pushes map the message
  type with `clipLabelFor` to `R.string.clip_voice_message`, and `onMessageCreated` uses
  `CLIP_PREVIEWS` so a voice note's push reads "Voice message" even when the text is encrypted.
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
- Drafts instead of silent discards: interruptions (backgrounding, calls, the cap, leaving the
  chat) are not the user's choice, so the recording is kept; only explicit gestures throw it away.
- Decibel levels instead of linear amplitude: perceived loudness is logarithmic, and a dB scale
  with a fixed floor gives quiet and loud microphones comparable shapes; peak normalisation then
  makes every waveform use the full bar height, which also repairs already-sent notes without a
  data migration.
- Fixed-pitch live bars: a recording should look like a tape being written, so earlier bars must
  keep their size and position instead of being re-spread across the width on every sample.
- Tap-to-switch on the same button (as in Telegram) avoids a second button in a crowded composer;
  the 200 ms hold window is what keeps a tap from ever starting the microphone or camera.
- Main-thread confinement for the recorder and player keeps state transitions simple without
  locks; `ExoPlayer` must be used from the thread it was built on.
- Transient focus and no auto-resume after calls follow how the user expects a voice note to
  behave next to music and phone calls.
- Size-scaled timeouts let a one-hour note (about 29 MB at 64 kbps) upload on a slow link,
  unlike the flat 35 s media timeout.

## Alternatives considered / rejected

- Per-device gain calibration or `AudioRecord` RMS for the waveform: rejected; `MediaRecorder`'s
  `maxAmplitude` on a dB scale plus peak normalisation fixed the flat-waveform report without a
  second audio pipeline.
- Normalising only new recordings: rejected because the visible problem was on messages already
  sent from friends' phones.
- A separate camera button for video notes: rejected in favour of tap-to-switch.

## Known limits

- Upload survives only while the process lives (no WorkManager); a killed upload needs a retry.
- Encryption is whole-file AES-GCM, so a one-hour clip is about 29 MB buffered in memory.
- Holding now waits 200 ms before recording starts.
- `ChatViewModel` has no unit-test harness; the tap/hold and lock/cancel gesture is covered by the
  instrumented `RecordGestureTest` instead.
- Not built yet: listened receipts, media-session (lock screen) controls, review before send.

## Visualization

```mermaid
flowchart TD
    MIC[Touch mic] -->|released under 200 ms| TOGGLE[Toggle voice / video]
    MIC -->|held 200 ms| REC[VoiceRecorder, dB levels, live bars]
    REC -->|slide left| CANCEL[Delete file]
    REC -->|slide up past 80 dp| LOCK[Locked: discard / stop / send]
    REC -->|release| SEND[sendRecording]
    LOCK --> SEND
    REC -->|background, call, focus loss, cap, leave chat| DRAFT[(VoiceDraftStore)]
    DRAFT -->|Send| SEND
    SEND --> OUT[(clip_outbox) + row SENDING]
    OUT --> UP[Encrypt + upload, size-scaled timeout]
    UP -->|fail| FAILED[FAILED, retry]
    UP --> TX{Row still alive?}
    TX -->|deleted| DEL[Delete uploaded file]
    TX -->|yes| FS[Firestore write, confirmed in 20s]
    FS --> SENT[SENT, promote to cache]
    FS -->|timeout| FAILED
    PEER[Receiver bubble, peak-normalised waveform] --> ENS[ensureLocal: outbox, cache or deduplicated download]
    ENS --> PLAY[ExoPlayer, transient focus]
    PLAY -->|ended| NEXT[Mark played, next voice clip]
```

## Related

[[Video Messages]] - [[Chat and Messaging]] - [[End-to-End Encryption]] - [[Data Layer]] -
[[Async and Reactivity]] - [[Notifications]] - [[Authentication]] - [[Voice Messages - Code]] -
[[Voice Messages - Explained Simply]] - [[Voice Messages - Basic Example]]
