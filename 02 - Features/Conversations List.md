# Conversations List

The chats tab: an offline-first list of conversations with live previews, per-participant
presence and avatars, unread counts, and swipe actions including the two deletion modes.

## Pieces

- `ConversationsViewModel` (`presentation/viewmodel/ConversationsViewModel.kt:38`) - plain
  `ViewModel` with a `MutableStateFlow<ConversationsUiState>` (the method-and-StateFlow style,
  see [[Presentation Layer (MVI)]]).
- Use cases (`domain/usecase/conversation/`): `GetConversationsUseCase`,
  `SyncConversationsUseCase`, `CreateConversationUseCase`, `DeleteConversationUseCase`,
  `DeleteConversationForEveryoneUseCase`, `ClearConversationForMeUseCase`.
- `ConversationRepositoryImpl` (`IConversationRepository`) and `ConversationReconciler` in
  `data/repository/`.
- UI: `ConversationsListScreen` / `ChatsScreen`, rows in `presentation/components/list/`
  (`ConversationListItem`, `SwipeableConversationRow`).

## Loading the list

`loadConversations` (`ConversationsViewModel.kt:62`) collects `getConversationsUseCase()` with
loading/error framing:

```kotlin
getConversationsUseCase()
    .onStart { _uiState.update { it.copy(isLoading = true) } }
    .catch { e -> _uiState.update { it.copy(isLoading = false, error = e.message?.toUiText()) } }
    .collect { conversations ->
        _uiState.update { it.copy(isLoading = false, conversations = conversations, error = null) }
        observeParticipantPresence(conversations)
        loadParticipantAvatars(conversations)
    }
```

The underlying stream is the local ObjectBox conversation query (offline-first, see
[[Data Layer]]), refreshed by a Firestore snapshot listener
(`FirestoreService.getAllConversations`, `data/remote/firebase/FirestoreService.kt:422`) that
also reports removed ids and cache state. `onStart` seeds the spinner; `catch` degrades to an
error state rather than crashing the collect. On every emission the ViewModel fans out to load
presence and avatars for the participants.

## Presence and avatars per row

- `observeParticipantPresence` (`ConversationsViewModel.kt:123`) cancels any previous
  `presenceJob`, collects the distinct set of other participants'
  `observeUserPresenceUseCase.observeMultiple(...)` (a `combine` of per-user presence flows,
  see [[Presence and Typing]]), and writes a `presenceMap` into state. Cancelling the prior job
  before resubscribing prevents leaking an observer for a stale participant set.
- `loadParticipantAvatars` (`ConversationsViewModel.kt:84`) first seeds avatar URLs already
  embedded in the conversation (`participantAvatars`), then fetches any not-yet-known user via
  `userRepository.getUserById`, tracking `fetchedUserIds` so each user is fetched once.
- `observeAliases` (`ConversationsViewModel.kt:53`) refreshes and observes contact aliases so a
  locally renamed contact shows the custom name.

## Creating a conversation

`createConversation(participantId)` (`ConversationsViewModel.kt:176`) calls
`CreateConversationUseCase`, which finds an existing 1:1 conversation
(`FirestoreService.findExistingConversation`, `FirestoreService.kt:329`) or creates one seeded
with `visibleTo = participantIds` (`FirestoreService.createConversation`, `:352`). Search-driven
creation happens through `SearchResultsOverlay` (see [[Navigation]]), which then navigates to
the chat.

## Deletion: for-me vs for-everyone

Three distinct operations, mirrored at the message level in [[Chat and Messaging]]:

- `deleteForMe(conversationId)` (`ConversationsViewModel.kt:162`) ->
  `ClearConversationForMeUseCase` -> `FirestoreService.clearConversationForMe`
  (`FirestoreService.kt:373`): writes `clearedAt.{uid} = now` and removes the uid from
  `visibleTo`. The conversation disappears from this user's list and messages before the
  cutoff are hidden locally (`clearedAtForMe` on `ObxConversation`, used as the message cutoff
  in `MessageRepositoryImpl.cutoffFor`, `data/repository/MessageRepositoryImpl.kt:46`). The
  other participant is unaffected.
- `deleteForEveryone(conversationId)` (`ConversationsViewModel.kt:169`) ->
  `DeleteConversationForEveryoneUseCase` -> deletes the conversation document for all
  participants.
- `deleteConversation(conversationId)` (`ConversationsViewModel.kt:144`) ->
  `DeleteConversationUseCase` -> `FirestoreService.deleteConversation` (`FirestoreService.kt:363`)
  hard-deletes the document.

`SwipeableConversationRow` surfaces these as swipe actions.

## Refresh and errors

`refresh` (`ConversationsViewModel.kt:194`) toggles `isRefreshing` around
`syncConversationsUseCase()` (a one-shot server pull, `fetchAllConversationsOnce`,
`FirestoreService.kt:404`, uses `Source.SERVER`). `clearError` resets the error field.

## Why this design

- Offline-first: the list renders from ObjectBox immediately and updates live as the Firestore
  listener writes through; no blank-while-loading after first run.
- Fanning presence/avatar loading out of the list stream keeps the row data current without
  bloating the conversation documents.
- `visibleTo` + `clearedAt` implement per-user deletion without server-side per-user copies of
  the conversation, and reuse the same cutoff mechanic the message stream already honours.

## Related

[[Chat and Messaging]] - [[Presence and Typing]] - [[Data Layer]] -
[[Presentation Layer (MVI)]] - [[Navigation]]
