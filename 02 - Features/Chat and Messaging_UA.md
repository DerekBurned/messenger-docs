---
tags:
  - _UA
---
# Чат і обмін повідомленнями

Екран чату: надсилання/отримання тексту й медіа, відповіді, підтвердження прочитання, пагінація
назад і два режими видалення повідомлень. Це найнасиченіша функціями область застосунку.

## Складові

- `ChatViewModel` (`presentation/viewmodel/ChatViewModel.kt:60`) - MVI
  (`MviViewModel<ChatUiState, ChatIntent, ChatEffect>`).
- State/Intent/Effect: `ChatUiState` (`presentation/state/ChatUiState.kt:16`), `ChatIntent`
  (`presentation/intent/ChatIntent.kt:8`), `ChatEffect` (`presentation/effect/ChatEffect.kt:6`).
- `MessageRepositoryImpl` (`data/repository/MessageRepositoryImpl.kt`) та `MediaRepositoryImpl`
  (`data/repository/MediaRepositoryImpl.kt`).
- Use case у `domain/usecase/message/`.
- UI: `ChatScreen` (`presentation/screens/ChatScreen.kt`) і компоненти в
  `presentation/components/chat/` та `presentation/components/media/`.

## Реактивний список повідомлень

Екран рендерить `ChatUiState.messages`, який живиться `loadMessages`
(`presentation/viewmodel/ChatViewModel.kt:223`), що збирає `getMessagesUseCase(conversationId)`
(це `Flow<Resource<List<Message>>>`). Джерело - холодний ObjectBox-потік `getMessagesStream`
(`data/repository/MessageRepositoryImpl.kt:34`), тож будь-який локальний запис повторно випромінює
весь упорядкований невидалений список. Паралельно `syncMessages` (`ChatViewModel.kt:213`) збирає
Firestore-потік оновлення, щоб втягнути віддалені рядки в ObjectBox. Повний аналіз потоку в
[[Async and Reactivity_UA|Асинхронність і реактивність]].

На кожному `Resource.Success` редьюсер також обчислює якір першого непрочитаного один раз
(`ChatViewModel.kt:231`) і сплющує медіа-елементи для внутрішньочатової галереї
(`conversationMedia`).

## Надсилання

`ChatIntent.SendMessage(text)` -> `sendMessage` (`ChatViewModel.kt:373`). Якщо є очікувані
вкладення, маршрутизує до `sendMediaMessage`; інакше очищає набір, встановлює `isSending` і збирає
`sendMessageUseCase(conversationId, text, replyTo)`. Репозиторій спершу пише в ObjectBox
(повідомлення показується миттєво), потім пробує Firestore; за збою рядок іде у `FAILED` і ставиться
повторна спроба в чергу - див. наскрізний прохід у [[Architecture Overview_UA|Огляд архітектури]] та
розділ offline-first у [[Data Layer_UA|Шар даних]].

## Відповіді

`ChatIntent.SetReplyTo(message)` встановлює `replyingTo` у стані (`ChatViewModel.kt:144`);
`SendMessageUseCase` захоплює обмежений сніпет повідомлення-відповіді
(`domain/usecase/message/SendMessageUseCase.kt:47`, `REPLY_SNIPPET_MAX = 200`) плюс
`replyToMessageId`/`replyToSenderId`, тож бульбашка може відрендерити цитату, а
`ChatIntent.JumpToMessage` - прокрутити до оригіналу. `jumpToMessage` (`ChatViewModel.kt:181`)
гортає назад (обмежено `MAX_JUMP_PAGES` і `withTimeoutOrNull` на сторінку), поки ціль не буде
завантажена, потім підсвічує її на 2.5 с.

## Медіа-вкладення

Вкладення стейджаться, потім надсилаються групою:
- `ChatIntent.ToggleAttachment` / `AddAttachment` оновлюють `pendingAttachments`
  (`ChatViewModel.kt:352`).
- `sendMediaMessage` (`ChatViewModel.kt:396`) делегує
  `mediaRepository.sendMediaMessage(conversationId, uris, caption)` і очищає стейджинг.
- `MediaRepositoryImpl.sendMediaMessage` (`data/repository/MediaRepositoryImpl.kt:65`) виконується
  на власному scope репозиторію рівня застосунку `CoroutineScope(SupervisorJob() + Dispatchers.IO)`
  (`:60`), тож завантаження переживає вихід з екрана чату. Він будує локальне `Message.Media`
  (копіюючи кожен обраний файл у `MediaCache`, витягуючи ширину/висоту/тривалість/BlurHash через
  `MediaMetadata`), зберігає його як `SENDING`, потім `uploadAndSend`.
- `uploadAndSend` (`:163`) завантажує кожен елемент паралельними `launch` усередині `coroutineScope`,
  обмеженими `withTimeoutOrNull(SEND_TIMEOUT_MS = 35_000)` (`:200`); елементи, що завершилися,
  отримують свій Storage-`url` назад, потім повідомлення йде у Firestore. Часткова успішність
  відокремлює невдалі елементи в окрему `FAILED`-бульбашку (`splitFailed`, `:244`), тож користувач
  може повторити лише їх (`ChatIntent.RetryMedia` -> `MediaRepositoryImpl.retry`, `:117`).
- Прогрес - це `StateFlow<Map<String, MediaTransfer>>`, спостережуваний через
  `observeTransfers`/`ChatIntent`-обробники, тож прогрес завантаження/вивантаження по елементах
  рендериться живо.
- Якщо розмова піддається шифруванню, кожен файл шифрується ключем на файл перед завантаженням
  (`MediaCipher.encrypt`, `MediaRepositoryImpl.kt:176`); ключ подорожує всередині зашифрованого
  корисного навантаження повідомлення. Див. [[End-to-End Encryption_UA|Наскрізне шифрування]].

Завантаження - дзеркало: `ChatIntent.DownloadMedia` -> `downloadMedia`
(`MediaRepositoryImpl.kt:75`), який завантажує у файл кешу й розшифровує `fileKey`-ключем елемента,
коли той присутній.

## Підтвердження прочитання

`ChatIntent.MessagesSeen(messages)` -> `onMessagesSeen` (`ChatViewModel.kt:261`) позначає кожне
щойно побачене повідомлення прочитаним рівно один раз (захищено `markedAsReadIds`) через
`MarkMessageAsReadUseCase` і, якщо щось змінилося, `markConversationRead`. Репозиторій встановлює
`status = READ` локально лише після успішного запису Firestore
(`MessageRepositoryImpl.markMessageAsRead`, `data/repository/MessageRepositoryImpl.kt:124`), а стан
прочитання зберігається, коли віддалені оновлення перезаписують рядки
(`putPreservingReadState`, `:169`). Іконки статусу рендеряться з `MessageStatus`.

## Видалення повідомлення: для себе проти для всіх

`ChatIntent.DeleteMessage(message, forEveryone)` -> `deleteMessage` (`ChatViewModel.kt:413`) ->
`DeleteMessageUseCase(message, forEveryone)`. Репозиторій розгалужується за прапорцем:

- Для всіх - `MessageRepositoryImpl.deleteMessageForEveryone`
  (`data/repository/MessageRepositoryImpl.kt:90`). Спершу він забезпечує авторизацію:
  ```kotlin
  if (currentUserId == null || message.senderId != currentUserId)
      return Result.failure(SecurityException("Only own messages can be deleted for everyone"))
  ```
  Лише відправник може видаляти для всіх. Він позначає локальний рядок `deleted = true` і викликає
  `FirestoreService.deleteMessage` (`FirestoreService.kt:173`), який встановлює `deleted = true` на
  документі, тож потік кожного учасника ховає його. За збою ставить у чергу повтор `DELETE`.
- Для себе - `deleteMessageForMe` (`MessageRepositoryImpl.kt:109`): позначає локальний рядок
  видаленим і викликає `FirestoreService.deleteMessageForMe` (`FirestoreService.kt:145`), який додає
  поточний uid у масив `deletedFor` документа. Під час читання `applyDeletedFor`
  (`FirestoreService.kt:168`) позначає повідомлення видаленим лише для uid у `deletedFor`, тож воно
  зникає для цього користувача, залишаючись для іншого. Запасна дія `DELETE_FOR_ME`.

Локальний запит повідомлень уже фільтрує `deleted = false` (`MessageRepositoryImpl.kt:36`), тож
видалення повторно випромінює список без рядка.

## Індикатор набору (сторона надсилання)

`ChatIntent.TextChanged` -> `onTextChanged` (`ChatViewModel.kt:324`) дебаунсить: після 500 мс
набору викликає `typingService.setTyping`, а 5-секундний TTL-job очищає це; порожнє поле очищає
одразу. Сторону отримання задокументовано в [[Presence and Typing_UA|Присутність і набір тексту]].

## Придушення сповіщень поки відкрито

`ChatScreen` позначає розмову відкритою через `CurrentConversationHolder`
(`presentation/screens/ChatScreen.kt:129`), що перевіряє `MyFirebaseMessagingService`
(`data/remote/firebase/MyFirebaseMessagingService.kt:103`), щоб не сповіщати про чат, на який
користувач уже дивиться. Див. [[Notifications_UA|Сповіщення]].

## Чому такий дизайн

- ObjectBox-як-істина робить send/delete/read миттєвими на відчуття й тримає UI живим офлайн;
  мережа узгоджує за ним.
- Групове завантаження медіа з паралелізмом по елементах, жорстким таймаутом і відокремленням
  невдалих елементів дає часткову успішність замість спінера "все або нічого".
- Два чітко розділені шляхи видалення (поле `deleted` проти масиву `deletedFor`) з захистом
  "лише відправник" на видалення для всіх відповідають очікуванням користувача й забезпечуються в
  репозиторії, а не лише в UI.

## Пов'язане

[[Data Layer_UA|Шар даних]] - [[Async and Reactivity_UA|Асинхронність і реактивність]] -
[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] -
[[Presence and Typing_UA|Присутність і набір тексту]] - [[Notifications_UA|Сповіщення]] -
[[End-to-End Encryption_UA|Наскрізне шифрування]] - [[Conversations List_UA|Список розмов]]
