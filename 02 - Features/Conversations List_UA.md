---
tags:
  - _UA
---
# Список розмов

Вкладка чатів: offline-first список розмов із живими прев'ю, присутністю й аватарами по учасниках,
лічильниками непрочитаного та свайп-діями, зокрема двома режимами видалення. Список прокручується
під плаваючими верхнім баром і таб-баром.

## Складові

- `ChatsViewModel` (`presentation/screens/chats/ChatsViewModel.kt:37`, раніше `ConversationsViewModel`) -
  звичайна `ViewModel` із `MutableStateFlow<ConversationsUiState>` (стиль метод-і-StateFlow, див.
  [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]]).
- Use case (`domain/usecase/conversation/`): `GetConversationsUseCase`,
  `SyncConversationsUseCase`, `CreateConversationUseCase`, `DeleteConversationUseCase`,
  `DeleteConversationForEveryoneUseCase`, `ClearConversationForMeUseCase`,
  `MarkConversationAsReadUseCase`, `MarkConversationAsUnreadUseCase`.
- `ConversationRepositoryImpl` (`IConversationRepository`) та `ConversationReconciler` у
  `data/repository/`.
- UI: `presentation/screens/chats/` (`ChatsScreen`, `ChatsTabContent`, `ChatsList`, `ChatsEmpty`,
  `ChatsError`, `ChatsLoading`, `ConversationsListPreview`), рядки в
  `presentation/components/list/` (`ConversationListItem`, `SwipeableConversationRow`).

## Завантаження списку

`loadConversations` (`ChatsViewModel.kt:84`) запускає потік розмов усередині `flatMapLatest` над
станом автентифікації, з обрамленням завантаження/помилки:

```kotlin
observeAuthStateUseCase()
    .flatMapLatest { user ->
        if (user == null) {
            flowOf(emptyList())
        } else {
            getConversationsUseCase()
                .onStart { _uiState.update { it.copy(isLoading = true) } }
        }
    }
    .catch { e ->
        _uiState.update {
            it.copy(
                isLoading = false,
                error = e.message?.toUiText()
            )
        }
    }
    .collect { conversations ->
        _uiState.update {
            it.copy(isLoading = false, conversations = conversations, error = null)
        }
        observeParticipantPresence(conversations)
        loadParticipantAvatars(conversations)
    }
```

Базовий потік - локальний запит розмов ObjectBox (offline-first, див. [[Data Layer_UA|Шар даних]]),
що оновлюється snapshot-слухачем Firestore (`FirestoreService.getAllConversations`,
`data/remote/firebase/FirestoreService.kt:504`), який також повідомляє id видалених документів і
стан кешу. `onStart` запускає спінер; `catch` переводить у стан помилки замість падіння збирання.
На кожному випромінюванні ViewModel розгалужується, щоб завантажити присутність і аватари учасників.

Чому `flatMapLatest` над станом автентифікації: `ChatsViewModel` скоупована до Activity, тож
переживає вихід. Раніше її слухач Firestore продовжував працювати на старому uid і міг записати
розмови старого акаунта назад в ObjectBox уже після скидання при виході. Тепер вихід перемикає
внутрішній потік на `flowOf(emptyList())`, що скасовує слухача й очищує список.

## Ідентичність співрозмовника: живі рядки користувачів, а не заморожені масиви

`Conversation.participantNames` і `participantAvatars` записуються один раз при створенні розмови
й ніколи не переписуються. Пряме їх показування означало, що перейменування співрозмовника не
доходило, а видалений акаунт зберігав старе ім'я й фото. Тепер список читає співрозмовників із
кешованих рядків користувачів:

- `observeParticipants` (`ChatsViewModel.kt:67`) збирає `userRepository.observeCachedUsers()` у
  `ConversationsUiState.participants`.
- `ChatsList` (`presentation/screens/chats/ChatsList.kt`) бере живого `participant` для кожного
  рядка, відкочується до заморожених масивів лише до надходження живого значення й обчислює
  `partnerDeleted = participant?.deleted == true`. Видалений співрозмовник зветься "Unknown user"
  через `resolveDisplayName(..., deleted = true)`, не має URL аватара (нейтральний силует), а
  `ConversationListItem` ховає його присутність.

## Присутність і аватари в рядках

- `observeParticipantPresence` (`ChatsViewModel.kt:158`) скасовує попередній `presenceJob`, збирає
  `observeUserPresenceUseCase.observeMultiple(...)` для унікальної множини інших учасників
  (`combine` потоків присутності по користувачах, див.
  [[Presence and Typing_UA|Присутність і набір тексту]]) і записує `presenceMap` у стан. Скасування
  попереднього job перед повторною підпискою не дає протікати спостерігачу застарілої множини.
- `loadParticipantAvatars` (`ChatsViewModel.kt:113`) спершу засіває URL аватарів, уже вбудовані в
  розмову (`participantAvatars`), потім отримує ще не відомих користувачів через
  `userRepository.getUserById`, відстежуючи `fetchedUserIds`, тож кожного користувача отримують
  один раз (невдале отримання повторюється наступного разу).
- `observeAliases` (`ChatsViewModel.kt:75`) оновлює й спостерігає псевдоніми контактів, тож локально
  перейменований контакт показує власне ім'я.

## Прокручування під плаваючими барами

Список заповнює весь екран і починається під плаваючим верхнім баром: `ChatsList` дає `LazyColumn`
`contentPadding(top = topInset, bottom = 116.dp або 96.dp)` залежно від того, чи таб-бар унизу.
Оскільки вміст тепер починається під баром, індикатор pull-to-refresh передається явно з тим самим
відступом `topInset`, тож спінер з'являється під баром, а не за ним.

## Створення розмови

`createConversation(participantId)` (`ChatsViewModel.kt:229`) викликає `CreateConversationUseCase`,
який знаходить наявну розмову 1:1 (`FirestoreService.findExistingConversation`,
`FirestoreService.kt:411`) або створює нову із `visibleTo = participantIds`
(`FirestoreService.createConversation`, `:434`). Почати розмову з видаленим акаунтом не можна: це
відхиляється в репозиторії ("Cannot start a conversation with a deleted account",
`data/repository/ConversationRepositoryImpl.kt:102`-`:104`).
Створення з пошуку відбувається через `SearchResultsOverlay` (див.
[[Search and Suggestions_UA|Пошук і підказки]]), який потім переходить до чату.

## Видалення: для себе проти для всіх

Три окремі операції, віддзеркалені на рівні повідомлень у
[[Chat and Messaging_UA|Чат і обмін повідомленнями]]:

- `deleteForMe(conversationId)` (`ChatsViewModel.kt:197`) -> `ClearConversationForMeUseCase` ->
  `FirestoreService.clearConversationForMe` (`FirestoreService.kt:455`-`:464`): пише
  `clearedAt.{uid} = now` і прибирає uid з `visibleTo`. Розмова зникає зі списку цього користувача,
  а повідомлення до межі ховаються локально (`clearedAtForMe` на `ObxConversation`, використовується
  як межа повідомлень у `MessageRepositoryImpl.cutoffFor`, `data/repository/MessageRepositoryImpl.kt:47`).
  Інший учасник не зачіпається.
- `deleteForEveryone(conversationId)` (`ChatsViewModel.kt:205`) ->
  `DeleteConversationForEveryoneUseCase` -> видаляє документ розмови для всіх учасників.
- `deleteConversation(conversationId)` (`ChatsViewModel.kt:179`) -> `DeleteConversationUseCase` ->
  `FirestoreService.deleteConversation` (`FirestoreService.kt:445`) жорстко видаляє документ. Жоден
  composable у `presentation/` цей метод ViewModel не викликає; екран редагування чатів доходить до
  того самого use case через `EditChatViewModel.deleteSelected`
  (`presentation/screens/editchat/EditChatViewModel.kt:56`).

`SwipeableConversationRow` виводить видалення для себе й для всіх (для всіх - лише на рядках
один-на-один, `isOneOnOne`, `ChatsList.kt:106`), плюс позначити прочитаним / непрочитаним
(`ChatsViewModel.kt:213`, `:221`) і Edit, як свайп-дії.

## Оновлення й помилки

`refresh` (`ChatsViewModel.kt:247`) перемикає `isRefreshing` навколо `syncConversationsUseCase()`
(одноразове витягування з сервера, `fetchAllConversationsOnce`, `FirestoreService.kt:486`,
використовує `Source.SERVER`). `clearError` скидає поле помилки.

## Feature flag

Дії списку вмикаються й вимикаються окремо (див. [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]]):
`CHAT_LIST_SWIPE_ACTIONS` - батько для `CHAT_LIST_MARK_READ`, `CHAT_LIST_DELETE`,
`CHAT_LIST_DELETE_FOR_EVERYONE` і `CHAT_LIST_EDIT`; `CHAT_LIST_PULL_REFRESH` стоїть окремо. Кожен
застосовується двічі:

- UI: `SwipeableConversationRow` читає `CHAT_LIST_EDIT`, `CHAT_LIST_MARK_READ`, `CHAT_LIST_DELETE` і
  (лише на рядках один-на-один) `CHAT_LIST_DELETE_FOR_EVERYONE`
  (`presentation/components/list/SwipeableConversationRow.kt:77`-`:80`); кнопка дії показується, лише
  якщо ввімкнено один із її прапорців, а рядок взагалі не можна свайпнути, коли дій не лишилося
  (`:81`-`:85`). `ChatsList` передає `enabled = pullEnabled` у `pullToRefresh` з
  `CHAT_LIST_PULL_REFRESH` (`presentation/screens/chats/ChatsList.kt:51`, `:58`).
- ViewModel: `ChatsViewModel.deleteForMe` (`:198`), `deleteForEveryone` (`:206`), `markAsRead` і
  `markAsUnread` (`:214`, `:222`, обидва на `CHAT_LIST_MARK_READ`) та `refresh` (`:248`) одразу
  повертаються через `featureFlags.refuses(...)`. `EditChatViewModel.deleteSelected` і `markAllRead`
  (`EditChatViewModel.kt:57`, `:66`) перевіряють `CHAT_LIST_EDIT`. `retry()` і `createConversation`
  перевірки прапорця не мають.

## Чому такий дизайн

- Offline-first: список рендериться з ObjectBox одразу й оновлюється наживо, коли слухач Firestore
  пише наскрізь; без порожнечі під час завантаження після першого запуску.
- Розгалуження завантаження присутності/аватарів із потоку списку тримає дані рядків актуальними,
  не роздуваючи документи розмов.
- Читання співрозмовників із живих рядків користувачів замість денормалізованих масивів означає,
  що перейменування й видалення акаунтів доходять до списку без переписування кожної розмови.
- `visibleTo` + `clearedAt` реалізують видалення для окремого користувача без серверних копій
  розмови на користувача й перевикористовують ту саму межу, яку вже поважає потік повідомлень.

## Пов'язане

[[Chat and Messaging_UA|Чат і обмін повідомленнями]] -
[[Presence and Typing_UA|Присутність і набір тексту]] - [[Data Layer_UA|Шар даних]] -
[[Feature Flags and Remote Config_UA|Feature flags і Remote Config]] - [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] - [[Navigation_UA|Навігація]] -
[[Search and Suggestions_UA|Пошук і підказки]] - [[Settings_UA|Налаштування]]
