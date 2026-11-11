# Calls (Voice and Video)

One-to-one voice and video calls over Agora RTC, driven by a foreground service that owns the
call state machine. The UI - full-screen call, the docked active-call bar with
swipe/bubble/snap-back gestures, PiP video - is a pure reflection of a single global state
object. Missed/ended/unreached calls become chat messages and feed a call-history tab.

## Pieces

- `CallForegroundService` (`data/remote/call/CallForegroundService.kt:50`) - the state machine
  and lifecycle owner. `@AndroidEntryPoint`.
- `ActiveCallHolder` (`data/remote/call/ActiveCallHolder.kt:9`) - the single global
  `StateFlow<ActiveCall?>`.
- `AgoraCallService` (`data/remote/call/AgoraCallService.kt:21`) - the `ICallService` RTC
  wrapper.
- `FirebaseCallSignalingService` (`ICallSignalingService`) - call signaling over Realtime DB.
- `AgoraTokenProvider` (`data/remote/call/AgoraTokenProvider.kt:10`; calls the `getRtcToken` Cloud Function), `CallUids`,
  `MissedCallRecorder`, `IncomingCallCoordinator`.
- Feature-flag gates: `CallMediaGate` (`data/remote/call/CallMediaGate.kt:12`) and
  `IncomingCallGate` (`data/remote/call/IncomingCallGate.kt:15`), see "Feature flag" below.
- Telecom integration: `data/remote/call/telecom/` (`TelecomCallManager`,
  `MessengerConnectionService`, `MessengerConnection`) for system phone-account/PiP behaviour.
- Presentation: `CallViewModel` (`presentation/screens/call/CallViewModel.kt:40`), `CallScreen`,
  `CallsViewModel`/`CallsScreen`, and components in `presentation/components/call/`
  (`ActiveCallBar`, `CallBarPresenter`, `CallBubble`, `CallControls`, `IncomingCallBar`,
  `PipCallSurface`).

## Agora RTC

Dependency `io.agora.rtc:full-sdk:4.6.3` declared directly in `app/build.gradle.kts:204`
(not in the version catalog). The full SDK is used because video needs the native video
modules; the App ID resolves from `local.properties` -> Gradle property -> empty string into
`BuildConfig.AGORA_APP_ID` (`app/build.gradle.kts:20`-`:24`, `:53`). No App ID is committed to the
repo. Without one the build still succeeds and the app still launches: `AgoraCallService` logs an
error and leaves the engine `null`, so calls simply do not connect.

`AgoraCallService` creates one `RtcEngine` in its `init` (`AgoraCallService.kt:106`) and adapts
the `IRtcEngineEventHandler` callbacks into the domain `CallEventListener`
(`AgoraCallService.kt:30`): `onUserJoined -> onRemoteUserJoined`, `onUserOffline` split into
intentional quit (`onRemoteUserLeft`) vs network drop (`onRemoteConnectionLost`),
`onConnectionStateChanged -> onConnectionStateChanged`, `onTokenPrivilegeWillExpire ->` token
renew. `joinChannel` (`:130`) sets `ChannelMediaOptions` (communication profile, broadcaster
role, publish mic, publish camera when video is on). Video is lazily enabled
(`ensureVideoModule`, `:238`) so audio-only calls never spin up the video pipeline.

## The state machine lives in the foreground service

`ActiveCallHolder` is an `object` with a `MutableStateFlow<ActiveCall?>`
(`ActiveCallHolder.kt:37`). Its `set`/`update`/`clear` are `internal` (`:40`-`:48`) - only the service writes;
everything else reads `state` or `snapshot()`. This is the core architectural decision: the
service is the single writer of call state and the UI is a stateless mirror, so there is no way
for the screen and the notification and the bar to disagree.

`CallForegroundService` runs everything on `CoroutineScope(SupervisorJob() + Dispatchers.IO)`
(`CallForegroundService.kt:58`) and is driven by intent actions (`onStartCommand`, `:109`):
`START_OUTGOING`, `START_INCOMING`, `ACCEPT`, `DECLINE`, `END`, plus mute/speaker/camera
toggles. Highlights:

- Outgoing (`handleStartOutgoing`, `:129`): asks `CallMediaGate` what media this call may use,
  seeds `ActiveCallHolder`, starts foreground with a microphone (and camera, if video) service
  type, then `signalThenJoin` (`:1033`): the `RINGING` `CallSignal` is written first, audio focus
  and local video/speaker setup run while the write is in flight, and only after the write has
  completed does the caller fetch its RTC token and join the Agora channel (`fetchTokenAndJoin`,
  `:726`). It then observes the remote status and the callee's ringing ack and arms a
  "requesting" timeout.
- Incoming (`handleStartIncoming`, `:177`): if already on another call it declines the new one;
  otherwise it posts a full-screen ringing notification (or a silent one if the app is
  foreground), acks ringing, observes remote status, and arms a 60s ring timeout.
- Accept (`handleAccept`, `:225`): re-checks media with `CallMediaGate` (a video call whose flag has since been switched off degrades to audio), flips to active, upgrades the foreground service type to
  include mic/camera, writes `ACTIVE` status, joins the channel, starts the call timer and
  presence tracking.
- Terminate (`terminate`, `:271`): records an ended call if it was active, writes the terminal
  status with a 2s timeout, clears the signaling nodes, then `stopSelfClean`.

Timeouts and reconnection are explicit jobs: ring (60s), requesting (30s), outgoing/no-answer
(65s), and a 20s reconnect window (`startReconnectTimeout`, `:517`) that ends the call if it is
not fully connected (`isFullyConnected` = active + remote present + Agora CONNECTED). Presence
of the peer is tracked over signaling (`startPresence`, `:598`) so if the peer's app dies the
call ends after the reconnect window.

Cleanup discipline: `stopSelfClean` (`:551`) cancels every job, stops the ringback tone, leaves
the Agora channel, releases audio focus, clears `ActiveCallHolder`, disconnects the Telecom
connection through `ActiveConnectionHolder.disconnect` (see [[#PiP and Telecom]]), and stops the
foreground service. `onDestroy` (`:941`) is the safety net: it disconnects *all* remaining Telecom
connections (`disconnectAll`), clears `ActiveCallHolder`, and cancels the whole scope via
`scope.coroutineContext[Job]?.cancel()`, so no timeout job - or stray system "ongoing call" - can
outlive the service. This is the worked example of service scope management referenced in
[[Async and Reactivity]].

## The UI mirrors the state

`CallViewModel` (`presentation/screens/call/CallViewModel.kt:40`) is a plain `AndroidViewModel`
with a `MutableStateFlow<CallUiState>`. `observeActiveCall` (`:106`) collects
`ActiveCallHolder.state` with `collectLatest` and copies each field into `CallUiState`; when the
holder goes null it plays a short "call ended" state and routes the exit via a
`Channel<CallExit>` (`receiveAsFlow()`, `:55`) - Pop, or OpenChat for an answered incoming call.
User actions are thin `startService` intents to the foreground service (`acceptCall`,
`endCall`, `toggleMute`, `switchCamera`, ...). Video surfaces bind directly to Agora via
`bindLocalVideo`/`bindRemoteVideo` (`AgoraCallService.kt:198`, `:204`). Starting an outgoing call
(`startOutgoing`, `CallViewModel.kt:181`) builds a `TelecomCallMeta`, starts the foreground service, and places
the call through Telecom.

`observePartnerAvatar` (`CallViewModel.kt:92`) cancels the previous avatar observer job before resubscribing to
a new partner id, so switching the tracked partner does not leak a stale `observeUser`
collector.

## The active-call bar: swipe, bubble, snap-back

While a call is active and not on the full call screen, a docked bar shows at the top (mounted
at the navigation root, `presentation/navigation/graph/MainDisplay.kt:625`, so it persists across
screens). `ActiveCallBar` (`presentation/components/call/ActiveCallBar.kt:148`) reads
`ActiveCallHolder.state` and `CallBarPresenter.mode`, and handles a vertical drag
(`detectVerticalDragGestures`, `ActiveCallBar.kt:190`). On release the drag offset decides:

- Swipe up past the threshold - animate off and `CallBarPresenter.hide()` (mode HIDDEN).
- Swipe down past the threshold, and only while `CallBarPresenter.floatingEnabled` -
  `minimizeToBubble()` (mode BUBBLE), rendered as a draggable `CallBubble` floating over content
  and spawned where the bar was (`bubbleOffset`).
- Otherwise - `animateTo(0f)` snaps the bar back to its docked position.

`CallBarPresenter` (`presentation/components/call/CallBarPresenter.kt:9`) is a global `object`
holding a `MutableStateFlow<CallBarMode>` (`BAR`, `HIDDEN`, `BUBBLE`, `VIDEO`; the enum is in
`presentation/components/call/model/CallBarMode.kt:3`) plus the last bubble offset and a
remembered restore mode. `VIDEO` is a floating video tile: while a call has video and the bar is
a bubble, `MainDisplay` promotes it to the tile (`showVideoTile`) and collapses it back when the
video stops (`MainDisplay.kt:202`-`:213`); dismissing the tile is remembered (`videoDismissed`)
until restored. The presenter resets to BAR when the call ends (`MainDisplay.kt:194`). The bar
shows a live waveform and call timer when connected, or a status label
(`Requesting...`/`Ringing...`/`Connecting...`) otherwise (`callBarStatus`, `ActiveCallBar.kt:93`),
with inline mute and end-call buttons that post service actions.

An `IncomingCallBar` (`MainDisplay.kt:612`) is shown for a ringing incoming call, and a
`CallBarOverlay` (`MainDisplay.kt:679`) hosts the floating bubble and video tile. See
[[Navigation]].

## PiP and Telecom

The `telecom/` package registers a self-managed phone account (`TelecomCallManager`,
registered in `MessengerApplication.onCreate`) so calls integrate with the system
(`MessengerConnectionService`, `MessengerConnection`), which is what enables system PiP/behaviour
for the call surface (`PipCallSurface`). Incoming FCM call payloads add a Telecom incoming
connection (see [[Notifications]]).

The system connection is what makes Android show a call as "ongoing", so tearing it down reliably
matters. Telecom creates the `Connection` *asynchronously* (`onCreateOutgoingConnection` /
`onCreateIncomingConnection` fire after `placeCall`/`addNewIncomingCall`), which can race a fast
hang-up and leave a phantom ongoing call. `ActiveConnectionHolder`
(`data/remote/call/telecom/ActiveConnectionHolder.kt`) is therefore idempotent and race-proof:

- `disconnect(callId, cause)` (`:28`) disconnects the connection if registered, otherwise records
  the callId in a bounded `pendingDisconnects` map.
- `put(callId, connection)` (`:12`) consults that map, so a connection that arrives *after*
  teardown disconnects itself immediately instead of lingering.
- `disconnectAll(cause)` (`:38`) drops every remaining connection - the `onDestroy` backstop.
- The busy branch of `handleStartIncoming` (`CallForegroundService.kt:187`) disconnects the
  rejected concurrent call's connection, and `stopSelfClean` routes teardown through `disconnect`
  rather than a one-shot lookup.

`MessengerConnection.markDisconnected` (`MessengerConnection.kt:60`) does the actual
`setDisconnected` + `destroy`.

## The screen during a call

Until 0.1.13.1 the display timed out mid-call because nothing held it awake. Now:

- `KeepScreenOn(enabled)` (`presentation/components/common/KeepScreenOn.kt:9`) is a
  `DisposableEffect` that adds `FLAG_KEEP_SCREEN_ON` to the host window while `enabled` and
  clears it on dispose. `CallScreenContent` uses it with `enabled = !state.callEnded` (`CallScreenContent.kt:26`); the voice
  recorder's composer reuses it while recording (see [[Voice Messages]]).
- `IncomingCallActivity` sets the same flag next to `setShowWhenLocked(true)` when it is created
  (`presentation/IncomingCallActivity.kt:56`, `:58`).
- Proximity blanking: `CallForegroundService` collects `ActiveCallHolder.state` in `onCreate`
  (`CallForegroundService.kt:95`-`:105`) and wants the lock while the call is active, not ended, with
  the local camera off and the speaker off - an earpiece audio call. `acquireProximityLock`
  (`:731`) takes a `PROXIMITY_SCREEN_OFF_WAKE_LOCK` (if the device supports it) so the screen
  turns off against the ear; switching to video or speaker, or ending the call, releases it with
  `RELEASE_FLAG_WAIT_FOR_NO_PROXIMITY` (`:748`), and `stopSelfClean` releases it as well. The
  lock is acquired with a 10-minute timeout and is only re-acquired when the call state changes.
  This needs the `WAKE_LOCK` permission.

`MainActivity` is declared with `android:screenOrientation="fullSensor"`, so the app rotates
freely, including the call screen.

## Call history

Ended/missed/unreached calls are recorded as call-type chat messages (`MissedCallRecorder`,
invoked from the service, e.g. `recordEndedCall`, `CallForegroundService.kt:420`). The calls tab
derives its log from those messages: `CallHistoryRepository.getCallHistory`
(`data/repository/CallHistoryRepository.kt:25`) queries `ObxMessage.type.oneOf(CALL_TYPES)`
(indexed - `ObxMessage.kt:24`), orders by timestamp desc, filters out deleted rows, batches the
needed conversations in a single `uid.oneOf(...)` query, and maps each row to a
`CallHistoryEntry` (partner, direction, duration, video flag). Returning a `Query.asFlow().map`
keeps the history live. `CallsViewModel` exposes it to `CallsScreen`; call-message bubbles in
chat render via `CallMessageCards`.

## RTC tokens are issued only to the caller and callee

PR #64 (merged 2026-11-07, fixes #33) closed a hole in `getRtcToken`
(`functions/src/index.ts:259`): it used to check only that the requester was signed in and that the
channel name looked like `call-<uuid>`, so anyone who learned a channel name could join that call.
Now:

- The caller writes the call signal and a create-only `callChannels/{channel}` entry
  (`callerId`, `calleeId`, `createdAt`) in one multi-path update
  (`callSignalUpdate`, `data/remote/call/FirebaseCallSignalingService.kt:159`, used by
  `sendCallSignal`, `:28`-`:30`). `database.rules.json` lets only the caller create that entry,
  requires a `call-<36 hex/dash chars>` key, and forbids changing the participants afterwards.
- `getRtcToken` rejects unauthenticated callers, malformed channel names (`isCallChannelName`,
  `functions/src/callAccess.ts:5`-`:7`), and any uid that is neither the entry's `callerId` nor
  `calleeId` (`isCallParticipant`, `callAccess.ts:9`-`:12`) with `permission-denied`
  (`functions/src/index.ts:259`-`:274`); only then does it build the token with the Agora
  credentials held as Cloud Functions secrets. Nothing from those secrets is in this repo.
- Because the entry must exist before a token request can succeed, the caller fetches its token
  only after the signal write completes (`signalThenJoin`, see Outgoing above); the callee asks
  after accepting, once the signal exists. If a token cannot be fetched `AgoraTokenProvider`
  returns `null` and the client joins without one (`AgoraTokenProvider.kt:14`-`:26`), which fails
  to connect on a project that requires tokens.

## Feature flag

Calls are controlled by `calls` and its children (`FeatureFlag.CALLS`, `CALLS_VIDEO`,
`CALLS_FLIP_CAMERA`, `CALLS_SPEAKER`, `CALLS_PIP`, `CALLS_FLOATING_BUBBLE`; turning `calls` off turns
every child off, `FeatureFlagState.isEnabled`). They are enforced in four layers (PR #68, merged
2026-11-07); see [[Feature Flags and Remote Config]]:

- Service/gate: `CallMediaGate` (`data/remote/call/CallMediaGate.kt:16`-`:31`) decides the media of
  a call: video only if `CALLS_VIDEO` is on, camera only if video and permission, speaker-on-start
  only if video and `CALLS_SPEAKER`. `CallForegroundService` asks it when starting an outgoing call
  (`:133`), accepting (`:229`), toggling speaker (`:313`), camera (`:335`), switching camera (`:350`)
  and showing remote video (`:702`). `IncomingCallGate.admit` (`IncomingCallGate.kt:22`) waits for a
  fresh config (`awaitFresh`) and, when `CALLS` is off, writes `DECLINED` to the signal and refuses;
  it is called for both RTDB signals (`IncomingCallCoordinator.kt:56`) and FCM payloads
  (`MyFirebaseMessagingService.kt:151`-`:152`), and its `startedCallIds` set also dedupes the same
  call arriving on both paths. A call already in progress is not ended.
- ViewModel: `CallViewModel.outgoingAllowed` (`CallViewModel.kt:177`) requires `CALLS` and, for
  video, `CALLS_VIDEO`; `startOutgoing` otherwise emits `CallExit.Unavailable` (`:181`-`:186`).
  `toggleSpeaker`, `toggleCamera` and `switchCamera` refuse their flags (`:229`, `:235`, `:240`),
  though turning a feature off is still allowed.
- UI: the chat top-bar call buttons (`ChatTopBar.kt:131`, `:146`), the profile buttons
  (`ChatUserProfileComponents.kt:98`, `:108`), call-back buttons in call cards and history
  (`CallMessageCards.kt:63`, `CallHistoryRow.kt:45`), and in-call controls
  (`CallScreen.kt:58`, `ActiveCallControls.kt:28`, `:38`, `:48`); a control stays visible when its
  state is already on, so it can be switched off. `CALLS_FLOATING_BUBBLE` is applied to
  `CallBarPresenter.setFloatingEnabled` from `MainDisplay.kt:154`-`:155`.
- Activity: `MainActivity.updateCallPipParams` ANDs the PiP request with `CALLS_PIP`
  (`MainActivity.kt:186`-`:187`), and the PiP action buttons for camera and speaker follow
  `CALLS_VIDEO` / `CALLS_SPEAKER` (`:216`, `:224`).

## Why this design

- A foreground service is the only component that reliably survives the app going to
  background/being swept during a call, so it owns the state machine; the UI cannot become the
  source of truth for something that must outlive it.
- A single global `StateFlow` as the one writer/many readers boundary removes an entire class of
  "the bar says connected but the screen says ringing" bugs.
- Explicit, named timeout jobs plus a reconnect window make the many call edge cases (no answer,
  peer app killed, network drop) deterministic rather than emergent.
- Gating in the service and the gates, not only the buttons, means a push, a stale screen or a
  PiP action cannot start a call or media that the flag has switched off.
- Issuing RTC tokens only to the participants recorded by the caller makes the channel name a
  capability that is no longer enough on its own.
- The bar/bubble/hidden presentation is decoupled into `CallBarPresenter` so gesture handling and
  rendering stay simple and the mode persists across navigation.

## Visualization

```mermaid
flowchart TD
    INTENTS[Intents START_OUTGOING ACCEPT END] --> SVC[CallForegroundService: state machine]
    SVC -->|only writer| HOLD[(ActiveCallHolder StateFlow)]
    SVC -->|1 write signal + callChannels entry| RTDB[(Realtime DB)]
    SVC -->|2 after the write: getRtcToken| FN[Cloud Function: caller or callee only]
    FN --> SVC
    GATE[CallMediaGate / IncomingCallGate: flags] --> SVC
    SVC --> AG[AgoraCallService joinChannel]
    AG --> RTC[Agora RTC channel]
    HOLD --> SCR[CallScreen]
    HOLD --> BAR[ActiveCallBar swipe bubble snap-back]
    HOLD --> NOTIF[Call notification]
    SVC --> REC[MissedCallRecorder writes call message]
    REC --> HIST[Calls tab history]
```

## Related

[[Async and Reactivity]] - [[Navigation]] - [[Notifications]] - [[Presence and Typing]] -
[[Feature Flags and Remote Config]] -
[[Data Layer]] - [[Architecture Overview]] - [[Calls (Voice and Video) - Basic Example]]
