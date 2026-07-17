# Reading the Code - Simple Patterns

The same few patterns show up everywhere in this app. Learn these six and the rest of the code
reads easily. Each one: a one-line idea, the real code with plain-English comments, and why it is
written that way.

(Note: the comments below are teaching comments for this doc. The real source has no comments -
that is a project rule. Here they are only to explain.)

---

## 1. The backing field: `_state` (private) and `state` (public)

Idea: the ViewModel is the only one allowed to CHANGE the data. Everyone else may only READ it.

Real code from `presentation/base/MviViewModel.kt:25`:

```kotlin
// PRIVATE + MUTABLE. The underscore "_" means "internal, do not touch from outside".
// Only this class can call _state.value = ... to change the screen's data.
private val _state = MutableStateFlow(initialState)

// PUBLIC + READ-ONLY. asStateFlow() removes the "can change" ability.
// The UI reads/observes this, but it physically cannot modify it.
val state: StateFlow<S> = _state.asStateFlow()
```

Why: if the UI could change state directly, any screen could scribble on the data from anywhere
and you would never know who caused a bug. By exposing only a read-only copy, all changes are
forced to go through the ViewModel (through `setState`, see pattern 6). One door in, one door out.

You will see the exact same shape for one-shot events:

```kotlin
private val _effect = MutableSharedFlow<E>(...)   // only the ViewModel emits
val effect: SharedFlow<E> = _effect.asSharedFlow() // the UI only listens
```

and for the global call state in `data/remote/call/ActiveCallHolder.kt:37`:

```kotlin
private val _state = MutableStateFlow<ActiveCall?>(null) // only the call service writes
val state: StateFlow<ActiveCall?> = _state.asStateFlow() // the whole UI just reads
```

---

## 2. `operator fun invoke`: a class you call like a function

Idea: a "use case" is one action (send a message, log in). It is written as a class but you call
it like a function.

Real code from `domain/usecase/message/SendMessageUseCase.kt:17`:

```kotlin
class SendMessageUseCase(...) {
    // "operator fun invoke" is the magic name. It lets you write
    //     sendMessageUseCase(conversationId, text)
    // instead of
    //     sendMessageUseCase.execute(conversationId, text)
    suspend operator fun invoke(conversationId: String, text: String, replyTo: Message? = null): Flow<Resource<Unit>> = ...
}
```

Why: it reads like plain English at the call site (`sendMessageUseCase(id, text)`), while still
being a real object that Hilt can build and tests can fake. One class = one verb.

---

## 3. Repository: the thing that hides "where data comes from"

Idea: the screen should not know or care whether a message is in the local database or on
Firebase. It just asks the repository.

```kotlin
// The screen/ViewModel depends on the INTERFACE (a promise of what you can do):
interface IMessageRepository {
    fun getMessagesStream(conversationId: String): Flow<List<Message>>
    suspend fun sendMessage(message: Message): Result<Unit>
}

// The real class (the "Impl") is the only place that touches ObjectBox + Firebase:
class MessageRepositoryImpl(...) : IMessageRepository { ... }
```

Why: you can swap the storage (this app already swapped Room for ObjectBox) without touching a
single screen, because screens only ever saw the interface. Hilt decides which real class to plug
in - see [[Dependency Injection (Hilt)]].

---

## 4. Cold Flow: a tap, not a bucket

Idea: a `Flow` is a stream of values over time. "Cold" means it does nothing until someone starts
drinking, and it stops when they walk away.

Real code from `data/repository/MessageRepositoryImpl.kt:34`:

```kotlin
fun getMessagesStream(conversationId: String): Flow<List<Message>> {
    return messageBox.query(...)         // "give me this chat's messages"
        .build()
        .asFlow()                        // turn the database query into a live tap
        .map { rows -> rows.map { it.toDomain() } } // convert storage rows -> app objects
        .flowOn(Dispatchers.Default)     // do that converting off the main thread
}
```

Why cold: the database connection should only be open while a chat screen is on screen watching
it. When you leave the chat, the tap closes automatically and nothing leaks. Every time the
database changes, the tap pushes a fresh list, so the screen updates by itself.

`StateFlow` (pattern 1) is a "hot" flow: it always has a current value ready, which is what a
screen needs on its very first frame. Full detail in [[Async and Reactivity]].

---

## 5. `suspend`: "this might take a moment, wait without freezing"

Idea: talking to the network or disk is slow. `suspend` lets the app pause that one job and keep
the screen smooth, then continue when the answer arrives.

```kotlin
// "suspend" = callable only from a coroutine; it can pause here without blocking the UI thread.
suspend fun sendMessage(message: Message): Result<Unit> {
    messageBox.put(message.toObx())              // save locally (instant)
    val result = firestoreService.sendMessage(m) // talk to the server (this may wait)
    ...
}
```

Why: without it, waiting for the server would freeze the whole app. A coroutine (started with
`viewModelScope.launch { ... }`) is just "a background job tied to this screen; if the screen
dies, the job is cancelled automatically."

---

## 6. State is data; `setState { copy(...) }` is how it changes

Idea: the screen's data is one immutable object. To "change" it you make a copy with one field
different.

```kotlin
// ChatUiState is a data class (immutable). You never edit it in place.
setState {                          // "here is the current state as 'this'"
    copy(isSending = true)          // make a NEW state, same as before but isSending = true
}
```

Why: because it is a brand-new object each time, the UI can cheaply check "did anything actually
change?" and only redraw when it did. And because only the ViewModel has `setState`, every change
is in one place. This ties back to pattern 1.

The three data shapes per screen:

```kotlin
sealed interface ChatIntent : UiIntent { ... }   // things the USER does (send, delete, type)
data class    ChatUiState(...)  : UiState        // what the screen currently LOOKS like
sealed interface ChatEffect : UiEffect { ... }   // one-time events (show an error toast once)
```

Data flows in a circle: user does something (`Intent`) -> ViewModel updates the data
(`setState`) -> screen redraws from the new `State`. See [[Presentation Layer (MVI)]].

---

## Related

[[Presentation Layer (MVI)]] - [[Async and Reactivity]] - [[Domain Layer]] - [[Data Layer]] -
[[End-to-End Encryption - Explained Simply]] - [[Home]]
