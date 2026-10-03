# Calls (Voice and Video) - Basic Example

The inherited pattern behind [[Calls (Voice and Video)]], in its plain textbook form - not the
real `CallForegroundService` state machine, but the two ideas it is built on. Then read
[[Calls (Voice and Video) - Code]] for the real source.

(Teaching comments only; the real source has none.)

---

## What is inherited

A call is Agora RTC's own **join-a-channel** API plus one app pattern: a **single-writer global
`StateFlow`** (the backing-field pattern 1 from [[Reading the Code - Simple Patterns]], scaled to
app-wide scope) so that every surface - full screen, docked bar, notification - reads one truth
and none of them can disagree.

## The basic example (canonical Agora join + a single-writer holder)

```kotlin
// Agora's own minimal call: create an engine, react to events, join a channel.
val engine = RtcEngine.create(context, appId, object : IRtcEngineEventHandler() {
    override fun onJoinChannelSuccess(channel: String, uid: Int, elapsed: Int) { /* connected */ }
    override fun onUserJoined(uid: Int, elapsed: Int) { /* the other person is here */ }
    override fun onUserOffline(uid: Int, reason: Int) { /* they left or dropped */ }
})
engine.joinChannel(token, channelName, /* optionalInfo */ null, /* uid */ 0)
// ... later ...
engine.leaveChannel(); RtcEngine.destroy()
```

```kotlin
// One writer, many readers. Only the owner may write; the whole UI just observes.
object ActiveCall {
    private val _state = MutableStateFlow<Call?>(null)   // only the service writes this
    val state: StateFlow<Call?> = _state.asStateFlow()   // every screen/bar/notification reads it
    internal fun set(call: Call?) { _state.value = call }
}
```

## How the real code specialises it

- The Agora wrapper is `AgoraCallService` (`data/remote/call/AgoraCallService.kt:21`): it adapts
  `IRtcEngineEventHandler` into a domain `CallEventListener`, sets `ChannelMediaOptions`
  (`joinChannel`, `:130`), and lazily enables video (`ensureVideoModule`, `:238`).
- The single-writer holder is `ActiveCallHolder` (`data/remote/call/ActiveCallHolder.kt:37`); its
  `set`/`update`/`clear` are `internal` so only `CallForegroundService`
  (`CallForegroundService.kt:50`) writes. The service owns the state machine
  (`START_OUTGOING`/`INCOMING`/`ACCEPT`/`DECLINE`/`END`) because only a foreground service
  reliably outlives the app during a call.
- Everything else - named timeout jobs (ring 60s, reconnect 20s), the swipe/bubble/snap-back bar
  (`ActiveCallBar.kt:149`), screen-on and proximity blanking, PiP/Telecom, and call-history-as-messages - is layered on this base in
  [[Calls (Voice and Video)]].

## Related

[[Calls (Voice and Video)]] - [[Calls (Voice and Video) - Code]] -
[[Calls (Voice and Video) - Explained Simply]] - [[Reading the Code - Simple Patterns]] -
[[Async and Reactivity]] - [[Home]]
