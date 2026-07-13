# Presence and Typing

Online/away/offline presence and per-conversation typing indicators, both over Firebase Realtime
Database (chosen over Firestore for these because RTDB has native `onDisconnect` and cheap
high-frequency writes).

## Pieces

- `RealtimeDbService` (`data/remote/firebase/RealtimeDbService.kt`) - raw RTDB reads/writes.
- `FirebasePresenceService` (`data/service/FirebasePresenceService.kt`) - `IPresenceService`.
- `FirebaseTypingService` (`data/service/FirebaseTypingService.kt`) - `ITypingService`.
- `PresenceManager` (`data/presence/PresenceManager.kt`) - drives the current user's presence
  state.
- `AppLifecycleObserver` (`data/presence/AppLifecycleObserver.kt`) - foreground/background hook.
- Use cases: `ObserveUserPresenceUseCase`, `ObserveTypingUseCase`.
- UI: `PresenceIndicator`, `TypingIndicator`.

## Presence data model

Presence is written under `presence/{uid}` in RTDB. `RealtimeDbService.setPresence`
(`RealtimeDbService.kt:37`) writes `{ state, lastSeen, lastChanged: ServerValue.TIMESTAMP }`.
The key reliability mechanism is `setupOnDisconnect` (`RealtimeDbService.kt:46`):

```kotlin
fun setupOnDisconnect(userId: String) {
    val offlineData = mapOf("state" to "offline", "lastSeen" to ServerValue.TIMESTAMP, ...)
    presenceRef.child(userId).onDisconnect().setValue(offlineData)
}
```

`onDisconnect()` is executed by the Firebase server when the client's socket drops, so a crash
or lost network still flips the user offline with a server timestamp - no client heartbeat
needed. `cancelOnDisconnect` clears it on a clean logout.

## Driving the current user's presence

`PresenceManager` (`data/presence/PresenceManager.kt:16`, `@Singleton`) turns lifecycle events
into presence writes through a conflated channel (`transitions: Channel<PresenceState>`,
`:24`). The collector (`ensureCollector`, `:75`) applies each requested state via
`presenceService.setMyPresence(state)`, skipping no-op repeats (`lastAppliedState`), and calls
`setupOnDisconnect` when going ONLINE.

Transitions:
- `goOnline` (`:30`) - ONLINE, cancels any pending offline timer.
- `goAway` (`:37`) - AWAY now, then a 5-minute timer to OFFLINE.
- `goIdleAway` (`:47`) - AWAY with no auto-offline.
- `goOffline` / `disconnect` (`:54`, `:61`) - OFFLINE / remove presence.

`Channel.CONFLATED` (`:24`) means a rapid burst of transitions collapses to the latest before it
is applied, avoiding a storm of RTDB writes on quick foreground/background flips. The manager is
called by `AuthViewModel` on login/logout (`goOnline`/`disconnect`, see [[Authentication]]) and
by `AppLifecycleObserver` on process foreground/background, wired in
`MessengerApplication.onCreate` (`MessengerApplication.kt:95`). The scope passed in is the
caller's (`viewModelScope` from auth, or the observer's), so the manager never owns a scope of
its own.

## Observing other users' presence

`FirebasePresenceService.observePresence` (`FirebasePresenceService.kt:21`) maps the RTDB
`ValueEventListener` flow into a `UserPresence(state, lastSeen)`. `observeMultiplePresence`
(`:30`) `combine`s a list of per-user flows into a single `Map<String, UserPresence>` - used by
the conversations list to light up every row (see [[Conversations List]]). In chat,
`ChatViewModel.observePartnerPresence` (`presentation/viewmodel/ChatViewModel.kt:283`) `combine`s
the presence flow with a 60s ticker so the "last seen N minutes ago" label keeps refreshing even
when presence itself has not changed. The underlying `callbackFlow` + `awaitClose` cleanup is in
[[Async and Reactivity]].

## Typing

Typing is written under `typing/{conversationId}/{uid}` in RTDB.
- Outgoing: `ChatViewModel.onTextChanged` (`presentation/viewmodel/ChatViewModel.kt:324`)
  debounces 500ms before `typingService.setTyping`, with a 5s TTL clear and an immediate clear
  on empty input. `FirebaseTypingService.setTyping` (`FirebaseTypingService.kt:24`) writes a
  timestamp node for the current user.
- Incoming: `FirebaseTypingService.observeTyping` (`FirebaseTypingService.kt:17`) maps the RTDB
  children to the set of typing uids and filters out the current user:
  ```kotlin
  realtimeDbService.observeTyping(conversationId).map { data ->
      val currentUserId = firebaseAuth.currentUser?.uid
      data.keys.filter { it != currentUserId }.toSet()
  }
  ```
  `ChatViewModel.observeTyping` (`ChatViewModel.kt:309`) collects it into `isPartnerTyping` /
  `typingUsernames`, rendered by `TypingIndicator`.

## Why Realtime Database (not Firestore) for these

- `onDisconnect()` gives server-side offline detection for free, which Firestore does not offer -
  essential so a killed app still shows offline.
- Presence and typing are high-frequency, tiny, ephemeral writes; RTDB's model is cheaper and
  lower-latency for that than Firestore document writes.
- Firestore remains the store of record for messages and conversations; RTDB is used only for
  the transient signals (presence, typing, and call signaling in [[Calls (Voice and Video)]]).

## Related

[[Conversations List]] - [[Chat and Messaging]] - [[Authentication]] -
[[Calls (Voice and Video)]] - [[Async and Reactivity]] - [[Data Layer]]
