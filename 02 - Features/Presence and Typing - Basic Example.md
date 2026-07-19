# Presence and Typing - Basic Example

The inherited pattern behind [[Presence and Typing]], in its plain textbook form - not the real
`PresenceManager`/`FirebasePresenceService`, but the two Firebase ideas they specialise. Then read
[[Presence and Typing - Code]] for the real source.

(Teaching comments only; the real source has none.)

---

## What is inherited

Presence is two Realtime Database features: **`onDisconnect()`**, a write the Firebase *server*
runs when your socket drops (so a crash still flips you offline, no client heartbeat), and the
**`callbackFlow` bridge** - the canonical way to turn a listener-based API into a cold Flow
(pattern 4 in [[Reading the Code - Simple Patterns]], full detail in [[Async and Reactivity]]).

## The basic example (server-side offline + a listener-to-Flow bridge)

```kotlin
// Register the offline write ONCE, while online. The server fires it if the socket dies.
fun goOnline(uid: String) {
    val ref = Firebase.database.getReference("presence/$uid")
    ref.onDisconnect().setValue(mapOf("state" to "offline", "lastSeen" to ServerValue.TIMESTAMP))
    ref.setValue(mapOf("state" to "online", "lastSeen" to ServerValue.TIMESTAMP))
}

// Turn a ValueEventListener into a cold Flow. awaitClose removes the listener when nobody listens.
fun observePresence(uid: String): Flow<String> = callbackFlow {
    val ref = Firebase.database.getReference("presence/$uid/state")
    val listener = object : ValueEventListener {
        override fun onDataChange(snap: DataSnapshot) { trySend(snap.getValue(String::class.java) ?: "offline") }
        override fun onCancelled(error: DatabaseError) { close(error) }
    }
    ref.addValueEventListener(listener)
    awaitClose { ref.removeEventListener(listener) }   // no leak when the screen goes away
}
```

## How the real code specialises it

- `onDisconnect()` is `RealtimeDbService.setupOnDisconnect` (`RealtimeDbService.kt:46`), armed on
  every ONLINE transition and cleared on clean logout (`cancelOnDisconnect`).
- The `callbackFlow` bridge is `FirebasePresenceService.observePresence`
  (`FirebasePresenceService.kt:21`); multiple users are merged with `combine`
  (`observeMultiplePresence`, `:30`) for the conversations list.
- `PresenceManager` (`data/presence/PresenceManager.kt:16`) adds a `Channel.CONFLATED` so a burst
  of foreground/background flips collapses to the latest state before it is written.
- Typing reuses the same bridge under `typing/{conversationId}/{uid}`, with a 500ms debounce out
  and a current-user filter in - see [[Presence and Typing]].

## Related

[[Presence and Typing]] - [[Presence and Typing - Code]] -
[[Presence and Typing - Explained Simply]] - [[Reading the Code - Simple Patterns]] -
[[Async and Reactivity]] - [[Conversations List]] - [[Home]]
