# Conversations List - Basic Example

The inherited pattern behind [[Conversations List]], in its plain textbook form - not the real
`ConversationsViewModel`, but the minimal shape it specialises. Then see
[[Conversations List - Code]] for the real thing.

(Teaching comments only; the real source has none.)

---

## What is inherited

An offline-first list is two base patterns stacked: a **cold Flow from the local database**
(pattern 4 in [[Reading the Code - Simple Patterns]]) with a standard
`onStart` / `catch` / `collect` framing, and a **`combine` fan-out** that turns one flow per row
into a single map the whole list can read.

## The basic example (offline-first list + per-row fan-out)

```kotlin
// The list is a cold stream off the local store. onStart seeds the spinner;
// catch degrades to an error state instead of crashing the collector.
fun load() = viewModelScope.launch {
    repository.conversationsStream()                       // Flow<List<Conversation>> from the DB
        .onStart  { _state.update { it.copy(isLoading = true) } }
        .catch    { e -> _state.update { it.copy(isLoading = false, error = e.message) } }
        .collect  { list ->
            _state.update { it.copy(isLoading = false, conversations = list) }
            observePresence(list)                          // fan out per emission
        }
}

// combine turns "one presence flow per user" into "one map for the whole list".
fun observePresence(list: List<Conversation>) = viewModelScope.launch {
    val perUser: List<Flow<UserPresence>> = list.map { presenceRepo.presenceOf(it.partnerId) }
    combine(perUser) { values -> list.map { it.partnerId }.zip(values).toMap() }
        .collect { presenceMap -> _state.update { it.copy(presence = presenceMap) } }
}
```

## How the real code specialises it

- The exact `onStart`/`catch`/`collect` framing is `loadConversations`
  (`presentation/viewmodel/ConversationsViewModel.kt:62`); the stream is the ObjectBox query
  refreshed by a Firestore snapshot listener (`FirestoreService.getAllConversations`,
  `data/remote/firebase/FirestoreService.kt:422`).
- The `combine` fan-out is `observeMultiple` in `ObserveUserPresenceUseCase`, driven by
  `observeParticipantPresence` (`ConversationsViewModel.kt:123`), which cancels the previous
  `presenceJob` before resubscribing so a stale participant set cannot leak an observer.
- Avatars fan out the same way but fetch-once-guarded (`loadParticipantAvatars`,
  `ConversationsViewModel.kt:84`), and per-user deletion rides `visibleTo` + `clearedAt` instead
  of server-side copies.

## Related

[[Conversations List]] - [[Conversations List - Code]] -
[[Conversations List - Explained Simply]] - [[Reading the Code - Simple Patterns]] -
[[Presence and Typing]] - [[Home]]
