---
tags:
  - _UA
---
# Чат і обмін повідомленнями

Екран чату: надсилання/отримання тексту й медіа, відповіді, підтвердження прочитання, пагінація
назад і два режими видалення повідомлень. Це найнасиченіша функціями область застосунку. Голосові
й відеокліпи ділять той самий екран, але мають власний конвеєр, описаний у
[[Voice Messages_UA|Голосові повідомлення]] та [[Video Messages_UA|Відеоповідомлення]].

## Складові

- `ChatViewModel` (`presentation/screens/chat/ChatViewModel.kt:63`) - MVI
  (`MviViewModel<ChatUiState, ChatIntent, ChatEffect>`).
- State/Intent/Effect: `ChatUiState` (`presentation/screens/chat/ChatUiState.kt:16`), `ChatIntent`
  (`presentation/screens/chat/ChatIntent.kt:8`), `ChatEffect` (`presentation/screens/chat/ChatEffect.kt:6`).
- `MessageRepositoryImpl` (`data/repository/MessageRepositoryImpl.kt`), `MediaRepositoryImpl`
  (`data/repository/MediaRepositoryImpl.kt`) і, для голосових кліпів, `ClipRepositoryImpl`
  (`data/repository/ClipRepositoryImpl.kt`).
- Use case у `domain/usecase/message/`.
- UI: `presentation/screens/chat/` (`ChatScreenWithNav`, `ChatScreenContent`, `MessageBubble`,
  `MessageComposer`, `VoiceBubble`, `VideoNoteBubble`, ...) і компоненти в `presentation/components/chat/` та
  `presentation/components/media/`.

## Реактивний список повідомлень

Екран рендерить `ChatUiState.messages`, який живиться `loadMessages`
(`presentation/screens/chat/ChatViewModel.kt:429`), що збирає `getMessagesUseCase(conversationId)`
(це `Flow<Resource<List<Message>>>`). Джерело - холодний ObjectBox-потік `getMessagesStream`
(`data/repository/MessageRepositoryImpl.kt:34`), тож будь-який локальний запис повторно випромінює
весь упорядкований невидалений список. Паралельно `syncMessages` (`ChatViewModel.kt:419`) збирає
Firestore-потік оновлення, щоб втягнути віддалені рядки в ObjectBox. Повний аналіз потоку в
[[Async and Reactivity_UA|Асинхронність і реактивність]].

На кожному `Resource.Success` редьюсер також обчислює якір першого непрочитаного один раз
(`ChatViewModel.kt:437`) і сплющує медіа-елементи для внутрішньочатової галереї
(`conversationMedia`).

## Надсилання

`ChatIntent.SendMessage(text)` -> `sendMessage` (`ChatViewModel.kt:589`). Якщо є очікувані
вкладення, маршрутизує до `sendMediaMessage`; інакше очищає набір, встановлює `isSending` і збирає
`sendMessageUseCase(conversationId, text, replyTo)`. Репозиторій спершу пише в ObjectBox
(повідомлення показується миттєво), потім пробує Firestore; за збою рядок іде у `FAILED` і ставиться
повторна спроба в чергу - див. наскрізний прохід у [[Architecture Overview_UA|Огляд архітектури]] та
розділ offline-first у [[Data Layer_UA|Шар даних]].

## Відповіді

`ChatIntent.SetReplyTo(message)` встановлює `replyingTo` у стані (`ChatViewModel.kt:199`);
`SendMessageUseCase` захоплює обмежений сніпет повідомлення-відповіді
(`domain/usecase/message/SendMessageUseCase.kt:49`) плюс `replyToMessageId`/`replyToSenderId`, тож
бульбашка може відрендерити цитату, а `ChatIntent.JumpToMessage` - прокрутити до оригіналу. Сніпет
дає `Message.replySnippet()` (`domain/model/Message.kt:72`, `REPLY_SNIPPET_MAX = 200` на `:80`):
текст обрізається до 200 символів, медіа бере підпис або "Media", кліп - свою мітку ("Voice
message" / "Video message"), а дзвінки сніпета не мають. Голосові відповіді йдуть через ту саму
функцію, тож голосове повідомлення зберігає цитату; надсилання медіа наразі очищає `replyingTo`, не
прикріплюючи його (`ChatViewModel.kt:618`). `jumpToMessage` (`ChatViewModel.kt:387`) гортає назад
(обмежено `MAX_JUMP_PAGES` і `withTimeoutOrNull` на сторінку), поки ціль не буде завантажена, потім
підсвічує її на 2.5 с.

## Медіа-вкладення

Вкладення стейджаться, потім надсилаються групою:
- `ChatIntent.ToggleAttachment` / `AddAttachment` оновлюють `pendingAttachments`
  (`ChatViewModel.kt:568`).
- `sendMediaMessage` (`ChatViewModel.kt:610`) делегує
  `mediaRepository.sendMediaMessage(conversationId, uris, caption)` і очищає стейджинг.
- `MediaRepositoryImpl.sendMediaMessage` (`data/repository/MediaRepositoryImpl.kt:62`) виконується
  на власному scope репозиторію рівня застосунку `CoroutineScope(SupervisorJob() + Dispatchers.IO)`
  (`:57`), тож завантаження переживає вихід з екрана чату. Він будує локальне `Message.Media`
  (копіюючи кожен обраний файл у `MediaCache`, витягуючи ширину/висоту/тривалість/BlurHash через
  `MediaMetadata`), зберігає його як `SENDING`, потім `uploadAndSend`.
- `uploadAndSend` (`:148`) завантажує кожен елемент паралельними `launch` усередині `coroutineScope`,
  обмеженими `withTimeoutOrNull(SEND_TIMEOUT_MS = 35_000)` (`:180`, константа на `:318`); елементи,
  що завершилися, отримують свій Storage-`url` назад, потім повідомлення йде у Firestore. Часткова
  успішність відокремлює невдалі елементи в окрему `FAILED`-бульбашку (`splitFailed`, `:224`), тож
  користувач може повторити лише їх (`ChatIntent.RetryMedia` -> `MediaRepositoryImpl.retry`,
  `:102`). Ліміт 35 с фіксований незалежно від розміру файлу; голосові кліпи натомість
  використовують таймаут, що масштабується з розміром (див.
  [[Voice Messages_UA|Голосові повідомлення]]).
- Прогрес - це `StateFlow<Map<String, MediaTransfer>>`, спостережуваний через
  `observeTransfers`/`ChatIntent`-обробники, тож прогрес завантаження/вивантаження по елементах
  рендериться живо.
- Якщо розмова піддається шифруванню, кожен файл шифрується ключем на файл перед завантаженням
  (`fileTransfer.upload(..., encryptActive)`, `MediaRepositoryImpl.kt:152`, `:161`); ключ
  подорожує всередині зашифрованого корисного навантаження повідомлення. Див.
  [[End-to-End Encryption_UA|Наскрізне шифрування]].

Завантаження - дзеркало: `ChatIntent.DownloadMedia` -> `downloadMedia`
(`MediaRepositoryImpl.kt:72`), який завантажує у файл кешу й розшифровує `fileKey`-ключем елемента,
коли той присутній.

## Підтвердження прочитання

`ChatIntent.MessagesSeen(messages)` -> `onMessagesSeen` (`ChatViewModel.kt:467`) позначає кожне
щойно побачене повідомлення прочитаним рівно один раз (захищено `markedAsReadIds`) через
`MarkMessageAsReadUseCase` і, якщо щось змінилося, `markConversationRead`. Репозиторій встановлює
`status = READ` локально лише після успішного запису Firestore
(`MessageRepositoryImpl.markMessageAsRead`, `data/repository/MessageRepositoryImpl.kt:125`), а стан
прочитання зберігається, коли віддалені оновлення перезаписують рядки
(`putPreservingReadState`, `:190`). Іконки статусу рендеряться з `MessageStatus`.

## Свайп для відповіді та навігаційний свайп

`SwipeToReply` (`presentation/components/chat/ChatSwipeGestures.kt:58`) обгортає текстові, медіа- та
кліп-бульбашки в `MessageBubble` (`presentation/screens/chat/MessageBubble.kt:123`, `:176`, `:243`).
Перетягування вправо захоплюється лише після touch slop і лише якщо воно більш горизонтальне, ніж
вертикальне (`:106`); бульбашка йде за пальцем з коефіцієнтом 0.6 і обмежена 80 dp (`:114`),
гаптичний відгук спрацьовує на порозі 56 dp, а відпускання за порогом викликає `onReply`, поки
бульбашка повертається за 200 мс (`:124`-`:127`). `pointerInput` оголошено перед модифікатором
`offset` (`:88` проти `:131`), тож перетягування вимірюється в просторі, що не рухається разом
із бульбашкою: від PR #39 (коміт `14db4da`, 2026-10-25) для відповіді потрібно близько 93 dp руху
пальця замість близько 150 dp, бо виміряне перетягування більше не зменшується, коли бульбашка
зсувається.

`chatNavigationSwipe` (`ChatSwipeGestures.kt:33`) - жест рівня чату на корені екрана
(`ChatScreenContent.kt:326`): перетягування на 72 dp або більше вправо повертає назад, вліво -
відкриває профіль співрозмовника ([[Chat Profile and Shared Media_UA|Профіль чату та спільні медіа]]).

## Видалення повідомлення: для себе проти для всіх

`ChatIntent.DeleteMessage(message, forEveryone)` -> `deleteMessage` (`ChatViewModel.kt:627`) ->
`DeleteMessageUseCase(message, forEveryone)`. Репозиторій розгалужується за прапорцем:

- Для всіх - `MessageRepositoryImpl.deleteMessageForEveryone`
  (`data/repository/MessageRepositoryImpl.kt:91`). Спершу він забезпечує авторизацію:
  ```kotlin
  if (currentUserId == null || message.senderId != currentUserId)
      return Result.failure(SecurityException("Only own messages can be deleted for everyone"))
  ```
  Лише відправник може видаляти для всіх. Він позначає локальний рядок `deleted = true` і викликає
  `FirestoreService.deleteMessage` (`FirestoreService.kt:198`), який встановлює `deleted = true` на
  документі, тож потік кожного учасника ховає його. За збою ставить у чергу повтор `DELETE`. Коли
  видалене повідомлення - голосовий чи відеокліп, `DeleteMessageUseCase` також просить репозиторій кліпів
  прибрати завантажений файл зі Storage (див. [[Voice Messages_UA|Голосові повідомлення]]).
- Для себе - `deleteMessageForMe` (`MessageRepositoryImpl.kt:110`): позначає локальний рядок
  видаленим і викликає `FirestoreService.deleteMessageForMe` (`FirestoreService.kt:169`), який додає
  поточний uid у масив `deletedFor` документа. Під час читання `applyDeletedFor`
  (`FirestoreService.kt:193`) позначає повідомлення видаленим лише для uid у `deletedFor`, тож воно
  зникає для цього користувача, залишаючись для іншого. Запасна дія `DELETE_FOR_ME`.

Локальний запит повідомлень уже фільтрує `deleted = false` (`MessageRepositoryImpl.kt:38`), тож
видалення повторно випромінює список без рядка.

## Видалені співрозмовники

Коли акаунт співрозмовника видалено, чат стежить за живим рядком користувача, а не за іменами й
аватарами, замороженими в документі розмови. `ChatViewModel` встановлює `partnerDeleted`
(`ChatViewModel.kt:176`, поле в `ChatUiState.kt:24`), і екран показує "Unknown user" з нейтральним
аватаром і "Last seen long time ago" (`ChatScreenContent.kt:141`), вимикає обидві кнопки дзвінка в
`ChatTopBar` (`presentation/components/chat/ChatTopBar.kt:55`, `:136`, `:151`), ховає присутність і
замінює композер пояснювальним повідомленням (`MessageComposer.kt:105`). Почати нову розмову з
видаленим акаунтом не можна. Як видаляється акаунт - див. [[Settings_UA|Налаштування]].

## Нерозшифровані повідомлення

Повідомлення, яке не вдається розшифрувати на цьому пристрої, показується як "Message cannot be
decrypted on this device" (`UNDECRYPTABLE_TEXT`, `data/crypto/E2eeMessageCodec.kt:191`) замість
старої мітки "Encrypted message". Див. [[End-to-End Encryption_UA|Наскрізне шифрування]].

## Індикатор набору (сторона надсилання)

`ChatIntent.TextChanged` -> `onTextChanged` (`ChatViewModel.kt:530`). Порожнє поле одразу викликає
`stopTyping`. Перша непорожня зміна запускає heartbeat-job, що викликає `typingService.setTyping`
кожні `TYPING_HEARTBEAT_MS` (3 с), а кожна зміна перезапускає stop-job, який викликає `stopTyping`
після `TYPING_STOP_DELAY_MS` (2 с) тиші (`:536`-`:552`). `stopTyping` (`:555`) скасовує обидва jobs і
викликає `typingService.clearTyping`; `onCleared` викликає `clearTypingNow` (`:651`). Надсилання
повідомлення чи кліпа також зупиняє набір. Сторону отримання задокументовано в
[[Presence and Typing_UA|Присутність і набір тексту]].

## Придушення сповіщень поки відкрито

`ChatScreenWithNav` позначає розмову відкритою через `CurrentConversationHolder`
(`presentation/screens/chat/ChatScreenWithNav.kt:156`, очищення на `:161`), що перевіряє
`MyFirebaseMessagingService` (`data/remote/firebase/MyFirebaseMessagingService.kt:110`), щоб не
сповіщати про чат, на який користувач уже дивиться. Див. [[Notifications_UA|Сповіщення]]. Той
самий `onDispose` також надсилає `ChatIntent.InterruptRecording` (`:163`-`:165`), якщо активність
не просто змінює конфігурацію, тож перехід із чату на інший екран перетворює запущений (навіть
заблокований) голосовий чи відеозапис на чернетку замість того, щоб лишати мікрофон чи камеру
увімкненими.

## Бульбашки чату потребують входу

Бульбашка розмови відкриває `BubbleActivity` (`presentation/BubbleActivity.kt:26`), яка рендерить
чат через власний `NavDisplay`. Тепер вона одразу завершується, коли ніхто не ввійшов (`:41`), і
тримає `FirebaseAuth.AuthStateListener` (`:34`), що закриває відкриту бульбашку в мить, коли сесія
завершується, тож бульбашка, залишена в шторці, не може читати локальний кеш після виходу.

## Feature flag

Можливості чату вмикаються й вимикаються окремо через Firebase Remote Config (див.
[[Feature Flags and Remote Config_UA|Feature flags і Remote Config]]). Прапорець застосовується у двох шарах, а дочірній прапорець
вимкнений, щойно вимкнено батьківський (`FeatureFlagState.isEnabled`,
`domain/featureflag/FeatureFlagState.kt:5`).

- Шлюз у ViewModel: `ChatViewModel.handleIntent` (`ChatViewModel.kt:190`) питає `requiredFlag(intent)`
  (`:230`) і відкидає intent, коли `featureFlags.refuses(flag)` істинне (guard пише в лог
  "Refused: <key> is switched off"). Відповідність: `SetReplyTo` -> `MESSAGE_REPLY`, `JumpToMessage`
  -> `MESSAGE_REPLY_JUMP`, `DeleteMessage` -> `MESSAGE_DELETE_FOR_EVERYONE` або
  `MESSAGE_DELETE_FOR_ME`, `DownloadMedia` -> `MEDIA_DOWNLOAD`, `ViewerDownload` -> `MEDIA_VIEWER`,
  `CancelUpload`/`CancelDownload` -> `MEDIA_CANCEL_TRANSFER`, `ToggleAttachment` ->
  `MEDIA_SENDING`, `AddAttachment` -> `MEDIA_CAMERA`, `StartRecording`/`ToggleClipMode` ->
  `VOICE_MESSAGES` або `VIDEO_NOTES` залежно від режиму кліпа, `LockRecording` -> `RECORDING_LOCK`,
  `PlayClip` -> `CLIP_PLAYBACK` (пропускається, коли цей кліп уже грає, тож пауза працює), `SeekClip`
  -> `CLIP_SEEK`.
- Шлюз в UI: `MessageBubble.kt:61`-`:66` читає `MESSAGE_COPY`, `MESSAGE_REPLY`,
  `MESSAGE_DELETE_FOR_ME`, `MESSAGE_DELETE_FOR_EVERYONE`, `MESSAGE_REPLY_JUMP` і `MEDIA_VIEWER`, щоб
  скласти список дій; `MessageContextMenuHost.kt:59` вимагає `MESSAGE_MENU`; `SwipeToReply` вимагає
  `MESSAGE_SWIPE_REPLY` (`ChatSwipeGestures.kt:64`); навігаційний свайп - `CHAT_SWIPE_NAVIGATION`
  (`ChatScreenContent.kt:327`); кнопки дзвінків у верхній панелі - `CALLS` і `CALLS_VIDEO`
  (`ChatTopBar.kt:131`, `:146`); кнопка вкладення - `MEDIA_SENDING` (`MessageComposer.kt:215`);
  комірка камери - `MEDIA_CAMERA` (`ChatScreenWithNav.kt:253`).

Прапорці кліпів описано в [[Voice Messages_UA|Голосові повідомлення]] і
[[Video Messages_UA|Відеоповідомлення]].

## Чому такий дизайн

- ObjectBox-як-істина робить send/delete/read миттєвими на відчуття й тримає UI живим офлайн;
  мережа узгоджує за ним.
- Групове завантаження медіа з паралелізмом по елементах, жорстким таймаутом і відокремленням
  невдалих елементів дає часткову успішність замість спінера "все або нічого".
- Застосування кожного прапорця у ViewModel, а не лише приховування елемента, означає, що застарілий
  UI, дія сповіщення чи бульбашка не дістануться вимкненої можливості.
- Два чітко розділені шляхи видалення (поле `deleted` проти масиву `deletedFor`) з захистом
  "лише відправник" на видалення для всіх відповідають очікуванням користувача й забезпечуються в
  репозиторії, а не лише в UI.

## Пов'язане

[[Data Layer_UA|Шар даних]] - [[Async and Reactivity_UA|Асинхронність і реактивність]] -
[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] -
[[Presence and Typing_UA|Присутність і набір тексту]] - [[Notifications_UA|Сповіщення]] -
[[End-to-End Encryption_UA|Наскрізне шифрування]] - [[Conversations List_UA|Список розмов]] -
[[Feature Flags and Remote Config_UA|Feature flags і Remote Config]] - [[Chat Profile and Shared Media_UA|Профіль чату та спільні медіа]] -
[[Voice Messages_UA|Голосові повідомлення]] - [[Video Messages_UA|Відеоповідомлення]]
