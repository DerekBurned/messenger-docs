# Chat Profile and Shared Media - Code

Companion to [[Chat Profile and Shared Media]]. A block-by-block walkthrough of the shared-media
source. Excerpts are kept to a signature or a line; the behaviour is described in prose, and the
"why" lives in the main note. `path:line` refs are relative to
`app/src/main/java/com/example/messenger/` unless they are repo-root paths.

Files covered:
- `presentation/screens/chatuserprofile/SharedMedia.kt`, `MediaTab.kt`, `ChatUserProfileUiState.kt`
- `presentation/screens/chatuserprofile/ChatUserProfileViewModel.kt`
- `presentation/screens/chatuserprofile/ChatUserProfileScreenContent.kt`, `ChatUserProfileScreen.kt`
- `presentation/components/profile/ProfileSharedMedia.kt`
- `presentation/components/media/MediaAlbum.kt` (`MediaCell`)
- `domain/repository/IMessageRepository.kt`, `data/repository/MessageRepositoryImpl.kt`,
  `data/remote/firebase/FirestoreService.kt`
- `data/repository/ClipRepositoryImpl.kt` (`registerRemoteClip`)

---

## 1. The data shapes

`presentation/screens/chatuserprofile/MediaTab.kt:3` is a one-line enum:

```kotlin
enum class MediaTab { PHOTO, VIDEO, VOICE, FILE }
```

`SharedMedia.kt` defines what a tab shows. `SharedMediaEntry` (`:9`) pairs one `MediaItem` with the
`Message.Media` that carries it (an album message has several items, and the parent is needed for
the sent status and the cancel-upload call). `SharedMedia` (`:15`) is three lists: `photos`,
`videos` (both of `SharedMediaEntry`) and `voice` (of `Message.Clip`).

`sharedMediaOf` (`:21`) is a pure function and the only place that classifies messages:

1. `filterNot { it.deleted }` then `sortedByDescending { it.timestamp }` (`:22`) - newest first,
   deleted rows removed.
2. Every `Message.Media` is flattened into one entry per item (`:23`-`:24`).
3. `photos` are entries whose `item.kind == MediaItem.IMAGE`, `videos` those with
   `MediaItem.VIDEO`; `voice` is the `Message.Clip` rows whose `clip is Clip.Voice` (`:25`-`:29`).
   A `Clip.Video` (video note) is deliberately excluded.

`ChatUserProfileUiState.kt:8` carries the screen state. The shared-media additions are `mediaTab`
(default `PHOTO`), `sharedMedia`, `mediaLoading`, `mediaEndReached` (`:18`), `mediaPagesLoaded`
(`:19`, a counter used as a recomposition key) and `transfers`, the live upload/download progress
map reused from the chat.

---

## 2. ChatUserProfileViewModel

`ChatUserProfileViewModel.kt:38` is a Hilt `ViewModel` (not the MVI base class). It injects the
user, conversation, message, media and clip repositories, the shared `ClipPlaybackController` and
`FeatureFlags`. State is a private `MutableStateFlow` exposed read-only; `clipPlayback` re-exports
the controller's state (`:52`). Two private fields drive paging: `remoteMedia`, a
`MutableStateFlow<Map<String, Message>>` (`:54`), and `oldestRemoteId` (`:56`), the cursor.

### Startup (`init`, `:59`)

It reads the nav arguments from `SavedStateHandle` (`userId`, `avatarUrl`, `partnerName`,
`conversationId`, blanks turned into null), seeds the header with the avatar and name so the screen
is not empty while the user loads, and then calls `load(userId)` and `loadPhotos(userId)`. Only
when `featureFlags.refuses(FeatureFlag.PARTNER_PROFILE_SHARED_MEDIA)` is false does it call
`loadSharedMedia` (`:76`-`:78`).

### loadSharedMedia (`:118`)

Sets `mediaLoading = true`, then in `viewModelScope`:

- resolves the conversation: the navigation id, else `findConversationWith(partnerId)` (`:121`),
  which takes `conversationRepository.getAllConversations().first()` and picks the first
  conversation with exactly two participants that includes the partner (`:140`-`:143`). If none is
  found it sets `mediaLoading = false, mediaEndReached = true` and stops (`:122`-`:124`);
- starts collecting `mediaRepository.transfers` into `state.transfers` (`:127`-`:129`);
- builds the merged stream: `combine(messageRepository.getMessagesStream(resolved), remoteMedia)`
  maps to `sharedMediaOf((remote + local.associateBy { it.id }).values)` (`:130`-`:132`). The
  `remote + local` order matters: a later map entry with the same key wins, so a local row replaces
  its remote twin;
- clears the loading flag and calls `loadMoreMedia()` once to fetch the first server page
  (`:135`-`:136`).

### loadMoreMedia (`:147`)

Three guards in a row: the feature flag (`:148`), a known conversation (`:149`), and
"no page job running and the end not reached" (`:150`). It then sets `mediaLoading`, and launches
`messageRepository.fetchSharedMediaPage(conversation, oldestRemoteId, MEDIA_PAGE_SIZE)`
(`:153`). On success it advances the cursor to the last returned message id (`:155`), merges the
page into `remoteMedia` keyed by id (`:156`), and updates the state: `mediaEndReached = page.size <
MEDIA_PAGE_SIZE` and `mediaPagesLoaded + 1` (`:157`-`:163`). On failure it sets
`mediaEndReached = true` (`:165`-`:167`), which ends paging instead of retrying.
`MEDIA_PAGE_SIZE = 60L` is in the companion (`:197`).

### Playback and transfers (`:171`-`:194`)

`toggleVoice` finds the clip in `sharedMedia.voice`, works out whether this tap is a pause
(`:175`), and refuses with `CLIP_PLAYBACK` only when it is not a pause (`:176`) so a switched-off
flag cannot leave a clip stuck playing. It then calls `clipRepository.registerRemoteClip` for every
voice clip in the list (`:177`) and `playbackController.toggle(ClipQueueItem(...))` (`:178`).
`downloadMedia`, `cancelDownload` and `cancelUpload` are one-line delegations to `mediaRepository`
behind `MEDIA_DOWNLOAD` (`:182`) and `MEDIA_CANCEL_TRANSFER` (`:187`, `:192`).

---

## 3. The repository chain

`IMessageRepository.kt:20` declares `fetchSharedMediaPage(conversationId, beforeMessageId, limit =
60)` returning `Result<List<Message>>`.

`MessageRepositoryImpl.kt:171` is a one-expression delegate to `firestoreService.fetchMessagesOfTypes`
with `types = SHARED_MEDIA_TYPES` (`:249`, `TYPE_MEDIA` and `TYPE_VOICE_CLIP`) and
`cutoff = cutoffFor(conversationId)` (`:180`). `cutoffFor` (`:47`) reads the conversation's
`clearedAtForMe` from ObjectBox, defaulting to 0.

`FirestoreService.fetchMessagesOfTypes` (`:317`) builds the query on the conversation's `messages`
subcollection:

- `whereIn("type", types)`, `whereGreaterThan("timestamp", cutoff)` and
  `orderBy("timestamp", DESCENDING)` (`:329`-`:331`);
- if a `beforeMessageId` is given it fetches that document and calls `startAfter(anchor)`
  (`:333`-`:336`); a missing anchor returns an empty success;
- `limit(limit).get().await()`, then each document becomes a `RemoteMessageDto` and is run through
  `applyDeletedFor(dto, codec.decode(dto))` (`:339`-`:340`), which is what decrypts E2EE messages and
  marks "deleted for me" ones;
- cancellation is rethrown and any other exception becomes `Result.failure` after a log line.

The query needs the composite index `messages(type ASC, timestamp DESC)` declared in
`firestore.indexes.json` (`:20`-`:25`).

`ClipRepositoryImpl.registerRemoteClip` (`:114`) stores the message in a `ConcurrentHashMap`
(`:73`). `loadClip` (`:338`) falls back to that map when ObjectBox has no row, and
`nextVoiceClip` passes `remoteClips.values` into the three-argument `nextVoiceClipAfter` (`:160`,
`:383`) so auto-advance sees remote clips too.

---

## 4. The screen and the section

`ChatUserProfileScreen.kt:10` collects state and `clipPlayback` with lifecycle awareness and wires
each callback to a ViewModel method (`:27`-`:33`).

`ChatUserProfileScreenContent.kt:64` reads the flag (`:81`) and keeps two pieces of local UI state:
`showViewer` for the avatar pager and `openMediaId` for the media pager (`:82`-`:83`). In the
`LazyColumn` (`:142`) the header, info card and safety-number rows come first; then, only when the
flag is on, a `MediaTabsRow` item and `sharedMediaSection(...)` (`:177`-`:189`). The tap callback
is just `openMediaId = entry.item.id` (`:182`).

When `openMediaId` is set (`:198`-`:218`) it picks the tab's entries, maps each to a `MediaSource`
only if `mediaCacheFile(context, entry.item)` exists and is non-empty (`:200`-`:211`), and shows
`FullscreenMediaPager` starting at `sources.indexOfFirst { it.key == itemId }` (`:213`-`:217`).

`ProfileSharedMedia.kt:62` `sharedMediaSection` is a `LazyListScope` extension, so it appends items
straight into the parent list. It computes `isEmpty` per tab (`:72`-`:77`, `FILE` is always empty),
then emits the tab body (`:78`-`:98`) and a tail (`:99`-`:106`):

```kotlin
state.mediaLoading && state.mediaTab != MediaTab.FILE -> item(key = "media-loading") { MediaLoading() }
```

followed by the sentinel (`media-load-more`, whose `LaunchedEffect(state.mediaPagesLoaded,
state.mediaTab)` calls `onLoadMore`, `:102`-`:104`) and the empty state (`:105`). A fixed 24 dp
spacer closes the list (`:107`).

`mediaGrid` (`:110`) chunks entries by `GRID_COLUMNS = 4` (`:60`, `:118`) and emits one `Row` per
chunk, keyed by the first item id, with a `MediaCell` per entry and trailing `Spacer`s to pad a
short row (`:141`). `SharedVoiceRow` (`:147`) derives `isCurrent`, `isPlaying`, `isLoading` and
`failed` from the shared `ClipPlaybackState` with `derivedStateOf` so only the playing row
recomposes as progress ticks (`:155`-`:162`); the row is clickable when not loading and either
playing or `CLIP_PLAYBACK` is on (`:170`). The button shows spinner, pause, retry or play (`:181`-
`:202`), and the subtitle is the live `VoiceWaveform` for the current clip, otherwise the date and
duration (`:215`-`:235`).

`MediaCell` (`MediaAlbum.kt:205`) is shared with the chat. It resolves the cache file, treats the
cell as downloaded when it exists and is non-empty (`:217`-`:218`), is clickable only when
downloaded and not transferring (`:227`), and picks one of three renderings by `transfer`:
uploading (image plus progress scrim), downloading (blur plus scrim), or idle (image, or blur plus
a download button when `MEDIA_DOWNLOAD` is on) (`:230`-`:247`).

---

## Related

[[Chat Profile and Shared Media]] - [[Chat Profile and Shared Media - Explained Simply]] -
[[Chat Profile and Shared Media - Basic Example]] - [[Chat and Messaging - Code]] -
[[Voice Messages - Code]] - [[Feature Flags and Remote Config]]
