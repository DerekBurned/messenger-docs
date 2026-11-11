---
tags:
  - _UA
---
# Сповіщення

Push поверх Firebase Cloud Messaging, з офлайн-збереженням повідомлень, сповіщеннями
MessagingStyle, ярликами й бульбашками розмов, вбудованою відповіддю та окремим шляхом вхідного
дзвінка.

## Складові

- `MyFirebaseMessagingService` (`data/remote/firebase/MyFirebaseMessagingService.kt:39`) - приймач
  FCM.
- `ChatNotifier` (`presentation/notification/ChatNotifier.kt:33`) - будує й постить сповіщення чату.
- `ConversationShortcuts`, `NotificationChannels`, `NotificationReplyReceiver`,
  `NotificationPermission`, `CurrentConversationHolder` (`presentation/notification/`).
- `FirebaseMessagingManager`, `FcmTokenSyncer` (`data/remote/firebase/`) і `DeviceRegistry`
  (`data/device/DeviceRegistry.kt:14`), що зберігає токен для кожного пристрою.
- `BubbleActivity` (`presentation/BubbleActivity.kt`) - активність, у яку розгортається бульбашка.

## Життєвий цикл токена

FCM-токен зберігається **для кожного пристрою**, щоб відправники (Cloud Functions, `functions/`) могли
адресувати кожен пристрій, на якому користувач увійшов. `DeviceRegistry.register(uid, token)`
(`data/device/DeviceRegistry.kt:20`) пише рядок у `users/{uid}/devices/{deviceId}` (назва, платформа,
версія застосунку, `lastSeenAt`, `fcmToken`, скидає `signedOutAt`) і, для старіших збірок Cloud Functions,
також оновлює застаріле поле `fcmToken` у документі користувача. Його викликають із трьох місць: при
вході `AuthViewModel.syncFcmTokenForCurrentUser` (`presentation/screens/auth/AuthViewModel.kt:323`); при
ротації `onNewToken` (`MyFirebaseMessagingService.kt:43`); і щоразу, коли змінюється uid увійшовшого,
`FcmTokenSyncer.start()` (`data/remote/firebase/FcmTokenSyncer.kt:23`, викликається з
`MessengerApplication.onCreate`, `MessengerApplication.kt:113`). При виході
`AndroidSessionCleaner.clearOnLogout` (`data/service/AndroidSessionCleaner.kt:35`) викликає
`DeviceRegistry.signOut` (прибирає `fcmToken` цього пристрою, ставить `signedOutAt`), видаляє токен на
пристрої (`FirebaseMessaging.deleteToken()`) і скасовує всі опубліковані сповіщення, тож чати
попереднього акаунта не приходять далі й не лишаються в шторці (див.
[[Authentication_UA|Автентифікація]]).

## Отримання push-повідомлення

`MyFirebaseMessagingService` конструюється ОС, тож він використовує ручний Hilt `@EntryPoint`
(`FcmEntryPoint`, `MyFirebaseMessagingService.kt:29`), щоб дістатися графа - див.
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]]. `onMessageReceived` (`:60`)
ігнорує пуші, коли ніхто не увійшов, і диспетчеризує за `data["type"]`:

- `message` -> `handleMessagePayload` (`:81`). На ефемерному
  `CoroutineScope(Dispatchers.IO)` (`:99`, довгоживучого scope немає в сервісі, який створює ОС -
  див. [[Async and Reactivity_UA|Асинхронність і реактивність]]) він:
  1. парсить вхідний DTO; якщо навантаження зашифроване, але обрізане
     (`enc == 1 && ciphertext.isBlank()`), витягує повне повідомлення з Firestore
     (`getMessageOnce`, `:101`),
  2. розшифровує через E2EE-кодек (`:105`) - див. [[End-to-End Encryption_UA|Наскрізне шифрування]],
  3. зберігає текстові повідомлення локально через `messageRepository.persistIncomingMessage`
     (`:107`), тож повідомлення є в ObjectBox, навіть якщо застосунок був у фоні (офлайн-доставка),
     потім
  4. якщо розмова наразі не відкрита (`CurrentConversationHolder.isOpen`, `:110`), постить
     сповіщення через `ChatNotifier.notifyIncoming`. Тіло обирається по черзі: голосовий чи
     відеокліп отримує свою мітку (`clipLabelFor` в `IncomingMessagePayload.kt`, відображається
     на `R.string.clip_voice_message` / `clip_video_message`, `:112`-`:116`); розшифрований текст - свій
     текст (ніколи не заглушку нерозшифрованого); медіа - підпис або "Media"; зашифроване
     повідомлення, яке не вдалося розшифрувати, - "New message"; інакше `preview` із сервера.
- `call` -> `handleCallPayload` (`:139`): спершу питає `IncomingCallGate.admit` (`:151`-`:152`), що відхиляє
  дзвінок, коли прапорець `calls` вимкнено, потім стартує `CallForegroundService` з intent-ом вхідного
  дзвінка й реєструє вхідне Telecom-з'єднання. Див. [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].
- будь-що інше -> запасне сповіщення з блоку `notification` FCM.

`CurrentConversationHolder` встановлюється `ChatScreenWithNav`, поки відкрито
(`presentation/screens/chat/ChatScreenWithNav.kt:156`), і саме так працює правило "не сповіщати
про чат, на який я дивлюся".

Серверна частина - Cloud Function `onMessageCreated` (`functions/src/index.ts:112`). Вона надсилає
data-пуш із `preview`: тип кліпу відображається через `CLIP_PREVIEWS` на "Voice message" / "Video message"
(`functions/src/index.ts:33`), зашифроване повідомлення отримує порожнє прев'ю, а звичайний текст
обрізається до 200 символів (`:165`-`:166`). Доставка розгалужується на токени пристроїв кожного
отримувача: `pushTargetsFor` (`index.ts:62`) читає рядки `devices` плюс застарілий токен, `targetsFor`
(`functions/src/push.ts:48`) пропускає відкликані рядки, лишає останні за `lastSeenAt`
`MAX_DEVICES_PER_USER` (10) і дедуплікує за токеном, `sendToTargets` (`index.ts:71`) шле пакетами, а
токени, які FCM позначає як незареєстровані, видаляються (`staleTargets`, `push.ts:101`). Пуш дзвінка
йде на один пристрій (`ringTargets`, `push.ts:69`, `MAX_CALL_DEVICES = 1`). Див.
[[Voice Messages_UA|Голосові повідомлення]].

## Побудова сповіщення (ChatNotifier)

`ChatNotifier` (`presentation/notification/ChatNotifier.kt:33`) - глобальний `object`, що тримає
невелику історію по розмовах (`ConvState`, обмежену `MAX_MESSAGES = 8`), тож сповіщення читається як
розмова. `notifyIncoming` (`:61`) резолвить `IconCompat` аватара (кешований, інакше завантажений з
URL через Coil із таймаутом 5 с, інакше згенерована монограма), додає повідомлення, пушить
динамічний ярлик і постить.

`post` (`:157`) будує сповіщення `MessagingStyle` із `Person` на кожного відправника, прив'язане до
ярлика розмови та `LocusId` (`:186`), тож система асоціює його з ярликом і може бульбашити. Додані
дії:
- Відповісти - дія `RemoteInput` (`replyAction`, `:282`) з `SEMANTIC_ACTION_REPLY` + згенерованими
  відповідями, опрацьована `NotificationReplyReceiver`.
- Відкрити у вікні - відкриває розмову як бульбашку (`openInWindowAction`, `:333`).
- Позначити прочитаним - `SEMANTIC_ACTION_MARK_AS_READ` (`markReadAction`, `:313`).

Коли 2+ розмов мають активні сповіщення, `postSummary` (`:357`) постить групову зведення
`InboxStyle` під `GROUP_KEY`. `appendOutgoing` (`:92`) додає власну відповідь користувача
(надіслану зі сповіщення) у тред тихо, тож сповіщення лишається синхронним після вбудованої
відповіді.

## Бульбашки розмов

`openAsBubble` (`ChatNotifier.kt:112`) постить сповіщення з `BubbleMetadata`
(`bubbleMetadata`, `:210`), що вказує на `BubbleActivity`, із `setAutoExpandBubble(true)` та
`setSuppressNotification(true)`. Бульбашки потребують відповідного динамічного ярлика й `LocusId`,
що надає `ConversationShortcuts.push`. Чат, відкритий усередині бульбашки, - це той самий Compose UI
чату, хостований `BubbleActivity`, яка одразу завершується, коли ніхто не ввійшов, і закриває себе,
коли сесія закінчується (див. [[Chat and Messaging_UA|Чат і обмін повідомленнями]]).

## Канали й дозвіл

`NotificationChannels.ensureCreated` виконується з `MessengerApplication.onCreate`
(`MessengerApplication.kt:109`) і визначає канали, використані вище
(`CHAT_MESSAGES`, `INCOMING_CALL`, `ONGOING_CALL`, `MISSED_CALLS`, `NotificationChannels.kt:15`-`:21`). `NotificationPermission` /
`NotificationPermissionGate` запитують runtime-дозвіл на сповіщення, щойно сесія стає `Active`
(`presentation/navigation/graph/AppNavigation.kt:68`).

## Feature flag

Чотири можливості сповіщень мають прапорці (див. [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]]):
`notification_inline_reply`, `notification_mark_read`, `notification_shortcuts` і `notification_bubbles`
(дочірній до `notification_shortcuts`, тож вимкнення ярликів вимикає й бульбашки). Оскільки сповіщення
будуються поза будь-якою ViewModel, перевірки живуть у notifier і receiver:

- `ChatNotifier` читає кожен прапорець через `flagOn` (`ChatNotifier.kt:204`), що бере `FeatureFlags` із
  точки входу `UiEntryPoints` і за збою вважає "увімкнено": ярлики пушаться лише коли ввімкнено
  `NOTIFICATION_SHORTCUTS` (`:84`); `openAsBubble` одразу повертається без `NOTIFICATION_BUBBLES` (`:113`);
  `post` додає дії Reply, Open-in-window і Mark-read лише за `NOTIFICATION_INLINE_REPLY` (`:188`),
  `NOTIFICATION_BUBBLES` (`:191`) і `NOTIFICATION_MARK_READ` (`:195`).
- `NotificationReplyReceiver` перевіряє повторно в момент дії, після `awaitFresh()`: дотик Reply чи
  Mark-read на старому сповіщенні обробляється, лише якщо прапорець досі ввімкнено, інакше воно просто
  оновлюється (`presentation/notification/NotificationReplyReceiver.kt:51`-`:62`).
- `MyFirebaseMessagingService` чекає свіжу конфігурацію перед публікацією (`:111`), тож push
  повідомлення використовує поточні значення прапорців.
- Дзвінки, що надходять пушем, обмежуються прапорцем `calls` через `IncomingCallGate` (див.
  [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]]).

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
[[End-to-End Encryption_UA|Наскрізне шифрування]] - [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]] - [[Authentication_UA|Автентифікація]]
