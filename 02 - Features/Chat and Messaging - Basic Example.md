# Chat and Messaging - Basic Example

The inherited pattern behind [[Chat and Messaging]], in its plain textbook form - not the real
700-line `ChatViewModel`/`MessageRepositoryImpl`, but the two base ideas they are built on. Then
read [[Chat and Messaging - Code]] for the real source.

(Teaching comments only; the real source has none.)

---

## What is inherited

Chat is two base patterns: **write-local-first** (save to the local store, then try the network;
never block the UI on the server) and the **cold Flow list** off a local query
(pattern 4 in [[Reading the Code - Simple Patterns]]) that re-emits the whole list on any change,
so the screen updates itself.

## The basic example (write-local-first + a live query)

```kotlin
// The screen reads this. A local query turned into a live tap: any DB write re-emits the list.
fun messagesStream(conversationId: String): Flow<List<Message>> =
    messageBox.query(Message_.conversationId equal conversationId)
        .order(Message_.timestamp)
        .build()
        .asFlow()                                    // ObjectBox reactive query -> Flow
        .map { rows -> rows.map { it.toDomain() } }
        .flowOn(Dispatchers.Default)                 // map off the main thread

// Sending: local first (instant), network second (may fail).
suspend fun send(message: Message): Result<Unit> {
    messageBox.put(message.copy(status = SENDING).toObx())   // 1) shows instantly via the stream
    val remote = api.send(message)                            // 2) try the server
    if (remote.isFailure) update(message.id) { it.status = FAILED }   // 3) mark + queue retry
    return Result.success(Unit)
}
```

The `Query.asFlow()` bridge itself is the inherited `callbackFlow` shape (see
[[Presence and Typing - Basic Example]] and [[Async and Reactivity]]).

## How the real code specialises it

- The stream is `getMessagesStream` (`data/repository/MessageRepositoryImpl.kt:34`), filtered to
  `deleted = false` and a per-conversation cutoff (`cutoffFor`, `:47`).
- `sendMessage` (`MessageRepositoryImpl.kt:53`) adds preview updates, a `SyncWorker` retry queue,
  and read-state preservation on remote refresh (`putPreservingReadState`, `:178`).
- Real chat layers on top: replies with a capped snippet, backward paging with a per-page
  `withTimeoutOrNull`, grouped media upload with per-item parallelism and failed-item splitting,
  read receipts confirmed only after the server write, and two deletion modes
  (`deleted` field vs `deletedFor` array) - all in [[Chat and Messaging]].

## Related

[[Chat and Messaging]] - [[Chat and Messaging - Code]] -
[[Chat and Messaging - Explained Simply]] - [[Reading the Code - Simple Patterns]] -
[[Async and Reactivity]] - [[Home]]
