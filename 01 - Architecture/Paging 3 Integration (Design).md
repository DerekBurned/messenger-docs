# Paging 3 Integration (Design)

> **Status (checked against release 0.2.2.4, 2026-11-11): Reverted.** The design was built in
> pieces between 2026-07-25 (the `pagedQuery` ObjectBox adapter, `2fb1942`) and 2026-07-27 (paged
> chat messages, `baa5758`), then taken out: the chat list was restored to non-paged on 2026-07-27
> (`f4fdd98`) and the remaining Paging 3 code was removed on 2026-07-28 (`09705fb`), because a paged
> query does not refresh live when a soft-deleted row leaves the `deleted == false` filter.
> Nothing from this design is in the 0.2.2.4 tree: `gradle/libs.versions.toml` has no `androidx.paging`
> entry, `app/src` has no `PagingSource`, `LazyPagingItems` or `RemoteMediator`, and the files the
> design names (`data/local/obx/ObxPagingSource.kt`, `ObxMappers.kt`) do not exist. Lists are
> reactive `Flow<List<T>>` again (see [[Data Layer]] and [[Async and Reactivity]]); older chat
> history is fetched on demand through `MessageRepositoryImpl.loadOlderMessages`
> (`data/repository/MessageRepositoryImpl.kt:151`), which is explicit "load more", not Paging 3.
> Kept as a historical record only.

This spec introduces Jetpack Paging 3 to replace unbounded list loading across the app.
It is deliberately kept DB-agnostic so a future move off ObjectBox (see the SQLDelight
discussion in [[Data Layer]]) reuses the entire UI and ViewModel layer unchanged.

## Motivation

The read side described in [[Data Layer]] and [[Async and Reactivity]] observes whole local
queries. For chat this is the scaling cliff: `MessageRepositoryImpl` observes an unbounded
per-conversation query (`data/repository/MessageRepositoryImpl.kt:36`) with no `LIMIT`, and the
`Query.subscribe().observer` bridge in `data/local/obx/ObxFlow.kt:8` re-emits the entire list
on every change, each row paying JSON deserialization for `mediaItemsJson`. Older-message
loading pages against Firestore (`LoadOlderMessagesUseCase`), not the local store, so the
observed list only grows. That cost profile is engine-independent - it is the query shape, not
ObjectBox, so Paging 3 is the fix.

## Scope

Four lists move to Paging 3:

1. Chat messages (highest value, highest risk).
2. Conversation list.
3. User search results.
4. Recent contacts.

They fall into two data-source families:

- ObjectBox-backed local lists: messages, conversation list, recent contacts.
- Firestore query results: user search.

Non-goals: no DB migration, no Firestore schema change, no change to the E2EE or sync pipeline
beyond wrapping older-message fetches in a `RemoteMediator`, no full-text search.

## Key design decision: DB-agnostic paging

The UI and ViewModel layers consume `Flow<PagingData<Domain>>` exposed from the repositories.
The concrete `PagingSource` implementation lives behind the repository interface and is
therefore swappable. The expensive work - the Compose and ViewModel refactor, especially the
chat list - is done once and survives a later `PagingSource` replacement (for example
SQLDelight's `QueryPagingSource`). Choosing a custom ObjectBox source now does not lock the
project out of that later.

## Dependencies

Add to `gradle/libs.versions.toml` (latest 3.3.x at implementation time):

- `androidx.paging:paging-runtime`
- `androidx.paging:paging-compose`
- `androidx.paging:paging-testing` (test only)

## Core infrastructure

A single generic local source, `data/local/obx/ObxPagingSource.kt`:

```kotlin
class ObxPagingSource<T : Any>(
    private val box: Box<T>,
    private val queryBuilder: () -> Query<T>,
) : PagingSource<Int, T>() {
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, T>
    override fun getRefreshKey(state: PagingState<Int, T>): Int?
}
```

Behaviour:

- `load` uses an integer offset key (`params.key ?: 0`) and returns
  `query.find(offset.toLong(), params.loadSize.toLong())`; `nextKey`/`prevKey` derive from
  offset and returned size.
- A `DataObserver` on the query calls `invalidate()` on any DB change, so writes from the sync
  pipeline drive Paging refreshes. This replaces the per-screen `queryFlow`/`asFlow` bridge in
  `data/local/obx/ObxFlow.kt` for paged screens only; non-paged observers stay as they are.
- `getRefreshKey` anchors around `state.anchorPosition` so a refresh keeps the user's scroll
  position.

Each repository gains a `pagedX(): Flow<PagingData<Domain>>` built from a `Pager` and mapped to
domain models with the existing mappers in `data/local/obx/ObxMappers.kt`.

Default `PagingConfig`: `pageSize = 30`, `prefetchDistance = 30`, `enablePlaceholders = false`.
Placeholders are off in v1 because they need a stable total count.

## Per-list designs

### Conversation list and recent contacts

Local `ObxPagingSource` over the existing queries, ordered as today (conversations by
`lastMessageTimestamp` descending; recent contacts by `lastOpenedAt` descending). The ViewModel
exposes `Flow<PagingData<...>>.cachedIn(viewModelScope)`; Compose swaps its `items(list)` for
`items(lazyPagingItems.itemCount)` via `collectAsLazyPagingItems()`. Lowest risk, so these ship
first and prove the pattern.

### User search

A separate `FirestorePagingSource` keyed by Firestore query cursors (`startAfter(lastSnapshot)`),
independent of ObjectBox. It wires into `SearchUsersViewModel` and the
`presentation/components/search/SearchResultsOverlay.kt` list. Self-contained and does not touch
the local-cache path.

### Chat messages

`ObxPagingSource` plus a `RemoteMediator` (canonical Paging 3 network-plus-cache).

Ordering: the source returns messages newest-first (`timestamp` descending) and the message
`LazyColumn` uses `reverseLayout = true`, so the newest message sits at the bottom (index 0).
Consequences:

- `APPEND` (scrolling up) requests older messages and drives the `RemoteMediator` to Firestore.
- `PREPEND` (newer) always returns `endOfPaginationReached = true`; new messages arrive through
  the live sync stream writing to ObjectBox, not through paging.

`RemoteMediator.load(APPEND)`:

1. Computes the cursor from the oldest cached message (the existing implicit cursor used by
   `LoadOlderMessagesUseCase`).
2. Calls the current Firestore older-fetch logic, which writes results into ObjectBox.
3. The `DataObserver` invalidates the source, so the next local page renders.
4. Returns `MediatorResult.Success(endOfPaginationReached = fetched < pageSize)` or
   `MediatorResult.Error`.

Remote-key state (the cost of using `RemoteMediator` on a non-Room source): a small
`ObxPagingState` entity `{ conversationId, endOfOlderReached: Boolean }` records when Firestore
history is exhausted, so the app stops hitting the network after the end. `initialize()` returns
`SKIP_INITIAL_REFRESH` (cache-first: show local immediately, network only on scroll).

Data flow:

```
LazyColumn (reverseLayout)  --scroll up / APPEND-->  Pager
Pager  -->  RemoteMediator.load(APPEND)  -->  Firestore fetchOlder  -->  ObjectBox
ObjectBox  --DataObserver invalidate-->  ObxPagingSource  -->  Pager  -->  LazyColumn
Live sync stream  -->  ObjectBox (drives the same invalidation)
```

## Chat-list feature rework

These features currently assume the full `List<Message>` lives in `ChatUiState`
(`presentation/screens/chat/ChatViewModel.kt`). They move off it, per [[Presentation Layer (MVI)]]:

- `conversationMedia` moves to its own lightweight query (`conversationId` and `type == MEDIA`,
  timestamp order) exposed as a small non-paged flow. Media count is far smaller than message
  count, so it does not need paging.
- `jumpToMessage` keeps its bounded loop (the `MAX_JUMP_PAGES` guard and per-page timeout at
  `ChatViewModel.kt:201`) but drives it by triggering appends and inspecting
  `lazyPagingItems.itemSnapshotList` until the id is present, then calls
  `LazyListState.scrollToItem(index)`.
- The unread anchor (`firstUnreadMessageId`) is resolved once from the first loaded page
  snapshot when it becomes non-empty.
- Read receipts (`onMessagesSeen`) still receive visible `Message`s from the Compose layer,
  which knows visible items from `LazyListState` plus the snapshot.
- The manual `isLoadingOlder` and `hasMoreOlder` flags are removed in favour of Paging
  `LoadState`.

## ViewModel and Compose consumption

The message list leaves `ChatUiState`. `ChatViewModel` exposes
`val messages: Flow<PagingData<Message>> = pager.flow.cachedIn(viewModelScope)`. Everything else
in `ChatUiState` (typing, presence, reply, attachments, error) is unchanged. Compose collects
with `val items = vm.messages.collectAsLazyPagingItems()`.

## Error and loading handling

Paging `LoadState` replaces the manual older-load plumbing: a refresh spinner from
`loadState.refresh`, an inline "loading older" indicator from `loadState.append`, and a retry
affordance via `lazyPagingItems.retry()` when `append` is `LoadState.Error` (a
`RemoteMediator`/Firestore failure).

## Testing

Following [[Presentation Layer (MVI)]] and the JUnit5 / MockK / AssertK / Turbine setup:

- `ObxPagingSource` unit tests: refresh and append keys, and invalidation on a DB change
  (Robolectric for ObjectBox).
- `RemoteMediator` tests with a fake Firestore service: APPEND success, `endOfPagination`, and
  the error path.
- ViewModel tests using `paging-testing` `asSnapshot { }` together with Turbine.

## Phasing

Each phase is independently shippable, so this does not become one long-lived branch.

- Phase 0: dependencies, `ObxPagingSource`, shared `PagingConfig`.
- Phase 1: conversation list and recent contacts (lowest risk, proves the pattern).
- Phase 2: user search (`FirestorePagingSource`).
- Phase 3: chat messages, `RemoteMediator`, and the chat-list feature rework (highest risk,
  done last once the pattern is proven).

## Risks

The chat-list rework concentrates the risk. `jumpToMessage` and the newest-first plus
`reverseLayout` ordering are the fiddly parts; the `paging-testing` `asSnapshot` helper de-risks
them. Placeholders are intentionally off in v1 because there is no stable total count.
