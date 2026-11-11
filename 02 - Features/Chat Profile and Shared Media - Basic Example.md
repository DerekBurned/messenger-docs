# Chat Profile and Shared Media - Basic Example

The inherited pattern behind [[Chat Profile and Shared Media]], in its plain textbook form - not
the real `ChatUserProfileViewModel`, but the cursor-paging idea it specialises. Then read
[[Chat Profile and Shared Media - Code]] for the real source. `path:line` refs are relative to
`app/src/main/java/com/example/messenger/`.

(Teaching comments only; the real source has none.)

---

## What is inherited

Two standard pieces: **Firestore cursor pagination** (order the query, take `limit` documents,
and continue `startAfter` the last document you got) and an **end-of-list trigger in a lazy list**
(a last item that, when it becomes visible, asks for the next page). On top sits one app pattern:
**merge a local cache with remote pages by id**, so what is already on the device is shown first and
the server only supplies the older tail.

## The basic example (canonical cursor paging into a merged list)

```kotlin
// 1) Page query: newest first, one page at a time, continue after the last document seen.
suspend fun fetchPage(after: DocumentSnapshot?, size: Long): List<DocumentSnapshot> {
    var q = collection.orderBy("timestamp", Query.Direction.DESCENDING)
    if (after != null) q = q.startAfter(after)
    return q.limit(size).get().await().documents
}

// 2) State: a cursor, an "end reached" flag, and the accumulated pages.
class AlbumViewModel : ViewModel() {
    private val remote = MutableStateFlow<Map<String, Item>>(emptyMap())
    private var cursor: DocumentSnapshot? = null
    var endReached = false

    fun loadMore() {
        if (endReached) return
        viewModelScope.launch {
            val page = fetchPage(cursor, PAGE)
            cursor = page.lastOrNull()
            remote.update { it + page.associate { d -> d.id to d.toItem() } }
            endReached = page.size < PAGE          // a short page means the end
        }
    }

    // 3) Merge: local rows and remote rows by id, the local one wins.
    val items = combine(localItems, remote) { local, rem ->
        (rem + local.associateBy { it.id }).values.sortedByDescending { it.timestamp }
    }
}

// 4) Trigger: the last item of the lazy list asks for the next page when it is composed.
LazyColumn {
    items(items) { Row(it) }
    if (!endReached) item { LaunchedEffect(pagesLoaded) { viewModel.loadMore() } }
}
```

## How the real code specialises it

- The cursor is a message **id**, not a snapshot held in memory: `oldestRemoteId`
  (`presentation/screens/chatuserprofile/ChatUserProfileViewModel.kt:56`); the Firestore call
  re-fetches that document to use as the `startAfter` anchor and returns an empty page if it has
  gone (`data/remote/firebase/FirestoreService.kt:333`-`:336`). That keeps the cursor serialisable
  and the repository free of Firestore types.
- The query is filtered as well as ordered: `type in (MEDIA, VOICE_CLIP)` and
  `timestamp > cutoff`, where the cutoff is the user's "clear for me" time
  (`FirestoreService.kt:329`-`:331`, `data/repository/MessageRepositoryImpl.kt:47`). That needs the
  composite index in `firestore.indexes.json`.
- Every page is decoded, not just mapped: encrypted messages are decrypted and "deleted for me"
  ones are marked (`FirestoreService.kt:340`).
- The local side is the chat's own live ObjectBox stream, and the merge is the `combine` in
  `ChatUserProfileViewModel.kt:130`; classification into photos / videos / voice is the pure
  `sharedMediaOf` (`presentation/screens/chatuserprofile/SharedMedia.kt:21`), which is
  unit-testable without Android.
- "End reached" has a second cause: a failed fetch also sets it (`:165`-`:167`), so an error stops
  paging instead of looping.
- Page size is 60 messages (`:197`), not items, and one cursor serves all tabs.
- The trigger is a sentinel item keyed on both the page counter and the tab
  (`presentation/components/profile/ProfileSharedMedia.kt:102`-`:104`), with a spinner taking its
  place while a page loads.
- Everything is behind a feature flag checked in the UI and the ViewModel - see
  [[Feature Flags and Remote Config]].
- The grid cell, viewer and voice player are the chat's own (`MediaCell`, `FullscreenMediaPager`,
  `ClipPlaybackController`), so encryption, transfer progress and audio focus come for free - see
  [[Chat and Messaging]] and [[Voice Messages]].

## Related

[[Chat Profile and Shared Media]] - [[Chat Profile and Shared Media - Code]] -
[[Chat Profile and Shared Media - Explained Simply]] - [[Chat and Messaging - Basic Example]] -
[[Reading the Code - Simple Patterns]] - [[Async and Reactivity]] - [[Home]]
