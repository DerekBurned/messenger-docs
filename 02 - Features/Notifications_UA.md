---
tags:
  - _UA
---
# Сповіщення

Push поверх Firebase Cloud Messaging, з офлайн-збереженням повідомлень, сповіщеннями
MessagingStyle, ярликами й бульбашками розмов, вбудованою відповіддю та окремим шляхом вхідного
дзвінка.

## Складові

- `MyFirebaseMessagingService` (`data/remote/firebase/MyFirebaseMessagingService.kt:32`) - приймач
  FCM.
- `ChatNotifier` (`presentation/notification/ChatNotifier.kt:30`) - будує й постить сповіщення чату.
- `ConversationShortcuts`, `NotificationChannels`, `NotificationReplyReceiver`,
  `NotificationPermission`, `CurrentConversationHolder` (`presentation/notification/`).
- `FirebaseMessagingManager`, `FcmTokenSyncer` (`data/remote/firebase/`).
- `BubbleActivity` (`presentation/BubbleActivity.kt`) - активність, у яку розгортається бульбашка.

## Життєвий цикл токена

FCM-токен записується в документ Firestore користувача, щоб відправники (Cloud Functions,
`functions/`) могли адресувати пристрій. При вході `AuthViewModel.syncFcmTokenForCurrentUser`
(`presentation/viewmodel/AuthViewModel.kt:259`) отримує й зберігає його; `onNewToken`
(`MyFirebaseMessagingService.kt:36`) пересинхронізує при ротації; `FcmTokenSyncer.start()` (з
`MessengerApplication.onCreate`) тримає його актуальним. `FirestoreService.updateFcmToken` /
`deleteFcmToken` (`FirestoreService.kt:111`, `:122`) роблять записи. При виході
`AndroidSessionCleaner.clearOnLogout` видаляє токен у Firestore й на пристрої
(`FirebaseMessaging.deleteToken()`) і скасовує всі опубліковані сповіщення, тож чати попереднього
акаунта не приходять далі й не лишаються в шторці (див. [[Authentication_UA|Автентифікація]]).

## Отримання push-повідомлення

`MyFirebaseMessagingService` конструюється ОС, тож він використовує ручний Hilt `@EntryPoint`
(`FcmEntryPoint`, `MyFirebaseMessagingService.kt:25`), щоб дістатися графа - див.
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]]. `onMessageReceived` (`:53`)
ігнорує пуші, коли ніхто не увійшов, і диспетчеризує за `data["type"]`:

- `message` -> `handleMessagePayload` (`:74`). На ефемерному
  `CoroutineScope(Dispatchers.IO)` (`:92`, довгоживучого scope немає в сервісі, який створює ОС -
  див. [[Async and Reactivity_UA|Асинхронність і реактивність]]) він:
  1. парсить вхідний DTO; якщо навантаження зашифроване, але обрізане
     (`enc == 1 && ciphertext.isBlank()`), витягує повне повідомлення з Firestore
     (`getMessageOnce`, `:94`),
  2. розшифровує через E2EE-кодек (`:98`) - див. [[End-to-End Encryption_UA|Наскрізне шифрування]],
  3. зберігає текстові повідомлення локально через `messageRepository.persistIncomingMessage`
     (`:100`), тож повідомлення є в ObjectBox, навіть якщо застосунок був у фоні (офлайн-доставка),
     потім
  4. якщо розмова наразі не відкрита (`CurrentConversationHolder.isOpen`, `:103`), постить
     сповіщення через `ChatNotifier.notifyIncoming`. Тіло обирається по черзі: голосовий чи
     відеокліп отримує свою мітку (`clipLabelFor` в `IncomingMessagePayload.kt`, відображається
     на `R.string.clip_voice_message` / `clip_video_message`, `:104`); розшифрований текст - свій
     текст (ніколи не заглушку нерозшифрованого); медіа - підпис або "Media"; зашифроване
     повідомлення, яке не вдалося розшифрувати, - "New message"; інакше `preview` із сервера.
- `call` -> `handleCallPayload` (`:131`): стартує `CallForegroundService` з intent-ом вхідного
  дзвінка й реєструє вхідне Telecom-з'єднання. Див. [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].
- будь-що інше -> запасне сповіщення з блоку `notification` FCM.

`CurrentConversationHolder` встановлюється `ChatScreenWithNav`, поки відкрито
(`presentation/screens/chat/ChatScreenWithNav.kt:144`), і саме так працює правило "не сповіщати
про чат, на який я дивлюся".

Серверна частина - Cloud Function `onMessageCreated`. Вона надсилає data-пуш із `preview`: тип
кліпу відображається через `CLIP_PREVIEWS` на "Voice message" / "Video message"
(`functions/src/index.ts:19`), зашифроване повідомлення отримує порожнє прев'ю, а звичайний текст
обрізається до 200 символів. Див. [[Voice Messages_UA|Голосові повідомлення]].

## Побудова сповіщення (ChatNotifier)

`ChatNotifier` (`presentation/notification/ChatNotifier.kt:30`) - глобальний `object`, що тримає
невелику історію по розмовах (`ConvState`, обмежену `MAX_MESSAGES = 8`), тож сповіщення читається як
розмова. `notifyIncoming` (`:58`) резолвить `IconCompat` аватара (кешований, інакше завантажений з
URL через Coil із таймаутом 5 с, інакше згенерована монограма), додає повідомлення, пушить
динамічний ярлик і постить.

`post` (`:137`) будує сповіщення `MessagingStyle` із `Person` на кожного відправника, прив'язане до
ярлика розмови та `LocusId` (`:165`), тож система асоціює його з ярликом і може бульбашити. Додані
дії:
- Відповісти - дія `RemoteInput` (`replyAction`, `:249`) з `SEMANTIC_ACTION_REPLY` + згенерованими
  відповідями, опрацьована `NotificationReplyReceiver`.
- Відкрити у вікні - відкриває розмову як бульбашку (`openInWindowAction`, `:300`).
- Позначити прочитаним - `SEMANTIC_ACTION_MARK_AS_READ` (`markReadAction`, `:280`).

Коли 2+ розмов мають активні сповіщення, `postSummary` (`:324`) постить групову зведення
`InboxStyle` під `GROUP_KEY`. `appendOutgoing` (`:87`) додає власну відповідь користувача
(надіслану зі сповіщення) у тред тихо, тож сповіщення лишається синхронним після вбудованої
відповіді.

## Бульбашки розмов

`openAsBubble` (`ChatNotifier.kt:96`) постить сповіщення з `BubbleMetadata`
(`bubbleMetadata`, `:177`), що вказує на `BubbleActivity`, із `setAutoExpandBubble(true)` та
`setSuppressNotification(true)`. Бульбашки потребують відповідного динамічного ярлика й `LocusId`,
що надає `ConversationShortcuts.push`. Чат, відкритий усередині бульбашки, - це той самий Compose UI
чату, хостований `BubbleActivity`, яка одразу завершується, коли ніхто не ввійшов, і закриває себе,
коли сесія закінчується (див. [[Chat and Messaging_UA|Чат і обмін повідомленнями]]).

## Канали й дозвіл

`NotificationChannels.ensureCreated` виконується з `MessengerApplication.onCreate`
(`MessengerApplication.kt:100`) і визначає канали, використані вище
(`CHAT_MESSAGES`, `INCOMING_CALL`, `ONGOING_CALL`). `NotificationPermission` /
`NotificationPermissionGate` запитують runtime-дозвіл на сповіщення, щойно сесія стає `Active`
(`presentation/navigation/graph/AppNavigation.kt:49`).

## Чому такий дизайн

- Збереження розшифрованого повідомлення при отриманні (а не лише показ тосту) означає
  офлайн-доставку: повідомлення вже в ObjectBox, коли користувач відкриває застосунок, тож чат
  повний без очікування foreground-синхронізації.
- MessagingStyle + ярлики + бульбашки - це нативний для платформи спосіб отримати вбудовану
  відповідь, згруповані треди й плаваючі вікна розмов; notifier тримає рівно стільки історії, щоб
  добре їх відрендерити.
- Розділення пушів дзвінка на шлях foreground-сервісу (а не звичайне сповіщення) - це те, що дає
  дзвінку дзвонити повноекранно й інтегруватися з Telecom навіть із холодного старту.

## Пов'язане

[[Chat and Messaging_UA|Чат і обмін повідомленнями]] -
[[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]] -
[[End-to-End Encryption_UA|Наскрізне шифрування]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]] - [[Authentication_UA|Автентифікація]]
