---
tags:
  - _UA
---
# Список розмов

Вкладка чатів: offline-first список розмов із живими прев'ю, присутністю й аватарами по учасниках,
лічильниками непрочитаного та свайп-діями, зокрема двома режимами видалення.

## Складові

- `ConversationsViewModel` (`presentation/viewmodel/ConversationsViewModel.kt:38`) - звичайна
  `ViewModel` із `MutableStateFlow<ConversationsUiState>` (стиль метод-і-StateFlow, див.
  [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]]).
- Use case (`domain/usecase/conversation/`): `GetConversationsUseCase`,
  `SyncConversationsUseCase`, `CreateConversationUseCase`, `DeleteConversationUseCase`,
  `DeleteConversationForEveryoneUseCase`, `ClearConversationForMeUseCase`.
- `ConversationRepositoryImpl` (`IConversationRepository`) та `ConversationReconciler` у
  `data/repository/`.
- UI: `ConversationsListScreen` / `ChatsScreen`, рядки в `presentation/components/list/`
  (`ConversationListItem`, `SwipeableConversationRow`).

## Завантаження списку

`loadConversations` (`ConversationsViewModel.kt:62`) збирає `getConversationsUseCase()` з
обрамленням завантаження/помилки:

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

Джерело - це локальний ObjectBox-запит розмов (offline-first, див. [[Data Layer_UA|Шар даних]]),
оновлюваний snapshot-слухачем Firestore (`FirestoreService.getAllConversations`,
`data/remote/firebase/FirestoreService.kt:422`), що також повідомляє видалені id і стан кешу.
`onStart` засіває спінер; `catch` деградує до стану помилки, а не крашить collect. На кожному
випромінюванні ViewModel розгортає завантаження присутності й аватарів для учасників.

## Присутність і аватари по рядках

- `observeParticipantPresence` (`ConversationsViewModel.kt:123`) скасовує попередній `presenceJob`,
  збирає `observeUserPresenceUseCase.observeMultiple(...)` для унікального набору інших учасників
  (`combine` потоків присутності по користувачах, див. [[Presence and Typing_UA|Присутність і набір тексту]])
  і записує `presenceMap` у стан. Скасування попереднього job перед повторним підписуванням не дає
  протекти спостерігачу для застарілого набору учасників.
- `loadParticipantAvatars` (`ConversationsViewModel.kt:84`) спершу засіває URL аватарів, уже
  вбудовані в розмову (`participantAvatars`), потім отримує кожного ще невідомого користувача через
  `userRepository.getUserById`, відстежуючи `fetchedUserIds`, тож кожен користувач отримується один
  раз.
- `observeAliases` (`ConversationsViewModel.kt:53`) оновлює й спостерігає псевдоніми контактів, тож
  локально перейменований контакт показує кастомне ім'я.

## Створення розмови

`createConversation(participantId)` (`ConversationsViewModel.kt:176`) викликає
`CreateConversationUseCase`, який знаходить наявну розмову 1:1
(`FirestoreService.findExistingConversation`, `FirestoreService.kt:329`) або створює нову, засіяну
`visibleTo = participantIds` (`FirestoreService.createConversation`, `:352`). Створення через пошук
відбувається через `SearchResultsOverlay` (див. [[Navigation_UA|Навігація]]), що потім навігує до
чату.

## Видалення: для себе проти для всіх

Три різні операції, віддзеркалені на рівні повідомлень у [[Chat and Messaging_UA|Чат і обмін повідомленнями]]:

- `deleteForMe(conversationId)` (`ConversationsViewModel.kt:162`) ->
  `ClearConversationForMeUseCase` -> `FirestoreService.clearConversationForMe`
  (`FirestoreService.kt:373`): записує `clearedAt.{uid} = now` і прибирає uid з `visibleTo`. Розмова
  зникає зі списку цього користувача, а повідомлення до відсічки приховуються локально
  (`clearedAtForMe` на `ObxConversation`, використовується як відсічка повідомлень у
  `MessageRepositoryImpl.cutoffFor`, `data/repository/MessageRepositoryImpl.kt:46`). Інший учасник
  не зачеплений.
- `deleteForEveryone(conversationId)` (`ConversationsViewModel.kt:169`) ->
  `DeleteConversationForEveryoneUseCase` -> видаляє документ розмови для всіх учасників.
- `deleteConversation(conversationId)` (`ConversationsViewModel.kt:144`) ->
  `DeleteConversationUseCase` -> `FirestoreService.deleteConversation` (`FirestoreService.kt:363`)
  жорстко видаляє документ.

`SwipeableConversationRow` виводить це як свайп-дії.

## Оновлення та помилки

`refresh` (`ConversationsViewModel.kt:194`) перемикає `isRefreshing` навколо
`syncConversationsUseCase()` (одноразове витягування з сервера, `fetchAllConversationsOnce`,
`FirestoreService.kt:404`, використовує `Source.SERVER`). `clearError` скидає поле помилки.

## Чому такий дизайн

- Offline-first: список рендериться з ObjectBox одразу й оновлюється живо, поки слухач Firestore
  пише крізь; жодного порожнього-під-час-завантаження після першого запуску.
- Розгортання завантаження присутності/аватарів поза потоком списку тримає дані рядка актуальними
  без роздування документів розмов.
- `visibleTo` + `clearedAt` реалізують видалення на кожного користувача без серверних копій розмови
  на кожного, і повторно використовують той самий механізм відсічки, який уже шанує потік
  повідомлень.

## Пов'язане

[[Chat and Messaging_UA|Чат і обмін повідомленнями]] -
[[Presence and Typing_UA|Присутність і набір тексту]] - [[Data Layer_UA|Шар даних]] -
[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] - [[Navigation_UA|Навігація]]
