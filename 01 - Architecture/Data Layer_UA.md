---
tags:
  - _UA
---
# Шар даних

`data/` реалізує кожен доменний інтерфейс і є єдиним шаром, що торкається ObjectBox, Firebase,
Agora, crypto-кодека, передавання медіа та ОС. Саме тут забезпечується модель offline-first.

Підпакети: `local/` (ObjectBox + prefs), `remote/` (auth, firebase, call, dto), `crypto/`
(E2EE), `device/` (рядок реєстру цього пристрою для push), `media/`, `presence/`, `service/`,
`repository/`, `sync/`.

## Принцип offline-first

Локальне сховище ObjectBox - джерело істини, яке читає UI. Мережа - писар і оновлювач за ним.
Метод репозиторію зазвичай:
1. одразу записує/оновлює ObjectBox (оптимістично), що
2. повторно випромінює локальний потік `Query.asFlow()`, який збирає UI, і лише тоді
3. пушить у Firestore, відкочуючись до черги синхронізації за збою.

`MessageRepositoryImpl.sendMessage` (`data/repository/MessageRepositoryImpl.kt:53`) - зразок:
`messageBox.put(...)`, потім `updateConversationPreview`, потім `firestoreService.sendMessage`; за
збою переводить рядок у `FAILED` і викликає `enqueueRetry`. Сторона читання ніколи не блокується на
мережі - див. аналіз потоків у [[Async and Reactivity_UA|Асинхронність і реактивність]].

## Сховище ObjectBox

Версія `5.4.2` (`gradle/libs.versions.toml:16`), обробка анотацій через kapt
(`app/build.gradle.kts:11`; сам плагін ObjectBox - `:12`). Сутності оголошені по одній на файл у `data/local/obx/`
(`ObxMessage.kt`, `ObxConversation.kt`, `ObxUser.kt`, `ObxRecentContact.kt`, `ObxSyncQueueItem.kt`,
...) і генеруються в `MyObjectBox`.

Домовленості сутностей (`data/local/obx/ObxMessage.kt:14`): `@Id var boxId: Long` - внутрішній
ключ; бізнес-ключ - це окремий `@Unique(onConflict = ConflictStrategy.REPLACE) var uid`, тож
upsert того самого віддаленого id замінює рядок, а не дублює його. Стовпці, "гарячі" для запитів,
несуть `@Index` - `ObxMessage.conversationId` та `ObxMessage.type`
(`ObxMessage.kt:16`, `:24`); індекс `type` існує, щоб запит історії дзвінків (`type.oneOf(...)`)
не був повним скануванням таблиці. Складні поля зберігаються через `@Convert` property-конвертери
(наприклад `MediaItemListConverter` для `mediaItemsJson`, `PhoneNumberConverter`,
`StringListConverter`), тож `List<MediaItem>` зберігається як JSON в одному стовпці. Голосові кліпи
використовують той самий прийом: `ObxMessage.clipJson` тримає `ClipPayload` через `ClipConverter`
(`ObxMessage.kt:36`), а `clipPlayed` (`:38`) керує крапкою непрослуханого. Видалені акаунти
лишаються рядками з `ObxUser.deleted` / `deletedAt` (`ObxUser.kt:23`).

Вісім боксів постачаються як singleton у `di/ObjectBoxModule.kt`: `provideBoxStore`
(`di/ObjectBoxModule.kt:30`) будує `MyObjectBox` і лише в debug відновлюється після
`DbSchemaException`, витираючи й перебудовуючи сховище - пониження схеми на dev-пристрої
самовиправляється замість краху під час запуску. Кожен `Box<Obx*>` - окремий
`@Provides @Singleton`, тож репозиторії впроваджують саме ті бокси, які їм потрібні.

`ObxRecentContact` (`data/local/obx/ObxRecentContact.kt:9`) забезпечує списки підказок пошуку: один рядок
на контакт за унікальним `@Unique userId`, з полями `username`, `avatarUrl` і двома мітками часу
`lastSearchedAt` / `lastOpenedAt`. Запис пошуку чи відкриття чату оновлює (upsert) рядок,
змінюючи одну мітку часу, тож одна й та сама людина ніколи не дублюється між двома секціями підказок.

Потокування з ObjectBox використовує розширення `Query<T>.asFlow()`
(`data/local/obx/ObxFlow.kt:8`); його прибирання `callbackFlow` + `awaitClose` розібрано в
[[Async and Reactivity_UA|Асинхронність і реактивність]].

Маппери живуть у `data/local/obx/mappers/` (`ObxMessageMappers.kt`, `ObxConversationMappers.kt`,
`ObxUserMappers.kt`; `toObx()` / `toDomain()`), тримаючи анотації ObjectBox поза доменними
моделями.

## Репозиторії

`data/repository/` (усі `@Singleton`, прив'язані в `di/RepositoryModule.kt`):
- `MessageRepositoryImpl` (`IMessageRepository`) - локальний потік + send/delete/read + оновлення
  Firestore + завантаження старіших сторінок (`loadOlderMessages`, `:151`) + одноразові сторінки
  спільних медіа для профілю чату (`fetchSharedMediaPage`, `:171`, спирається на
  `FirestoreService.fetchMessagesOfTypes`, `data/remote/firebase/FirestoreService.kt:317`) + злиття
  зі збереженням стану прочитання (`putPreservingReadState`,
  `data/repository/MessageRepositoryImpl.kt:190`). Правила видалення пояснено в
  [[Chat and Messaging_UA|Чат і обмін повідомленнями]].
- `MediaRepositoryImpl` (`IMediaRepository`) - володіє власним scope рівня застосунку для
  завантажень/вивантажень; див. [[Chat and Messaging_UA|Чат і обмін повідомленнями]] та
  обґрунтування scope в [[Async and Reactivity_UA|Асинхронність і реактивність]].
- `ClipRepositoryImpl` (`IClipRepository`, `data/repository/ClipRepositoryImpl.kt:48`) - голосові
  кліпи: надсилання через outbox, підтверджений запис у Firestore, дедупльоване завантаження
  (`InFlight`), відновлення після смерті процесу, прибирання видалених кліпів. Див.
  [[Voice Messages_UA|Голосові повідомлення]].
- `ConversationRepositoryImpl` (`IConversationRepository`) - потік списку розмов, create/find,
  видалення для себе проти для всіх через `visibleTo`/`clearedAt`.
- `UserRepositoryImpl` (`IUserRepository`) - пошук/lookup користувачів, псевдоніми контактів,
  спостереження за аватаром.
- `AuthRepositoryImpl` (`IAuthRepository`) - обгортка над `FirebaseAuthService`.
- `SyncRepositoryImpl` (`ISyncRepository`) - осушує чергу синхронізації.
- `CallHistoryRepository` - записи журналу дзвінків, похідні від повідомлень типу дзвінка
  (`data/repository/CallHistoryRepository.kt:20`); див. [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].
- `RecentContactsRepository` (`data/repository/RecentContactsRepository.kt`) - простий
  box-орієнтований `@Singleton` (тієї ж форми, що й `CallHistoryRepository`, без інтерфейсу), що
  читає/пише `ObxRecentContact`. Надає `recordSearched` / `recordOpened` (upsert-и на IO-диспетчері)
  та два реактивні читання на базі `queryFlow` - `observeRecentSearched` / `observeRecentOpened`,
  зіставлені в `List<User>`. Живить секції "Нещодавно шукані" / "Нещодавно відкриті" в оверлеї пошуку.
- `ConversationReconciler` - узгоджує віддалені знімки розмов із локальними рядками.

`data/local/repository/LocalRepositoryImpl` (`ILocalRepository`) централізує прямий доступ до
боксів, що використовується кількома репозиторіями.

## Сервіси Firebase

BOM `34.10.0` (`gradle/libs.versions.toml:13`) керує версіями Firebase. Сервіси обгортають SDK і
повертають доменні типи / `Result`.

- `FirestoreService` (`data/remote/firebase/FirestoreService.kt`) - користувачі, розмови й
  підколекція `conversations/{id}/messages`. Читання - це `callbackFlow` зі snapshot listener
  (`getRecentMessagesStream:229`, `getAllConversations:504`); записи - suspend-методи `Result`, що
  використовують `await()` з `kotlinx-coroutines-play-services`. Він інтегрує E2EE-кодек під час
  encode/decode. Видалення для всіх встановлює поле `deleted` (`:204`); видалення для себе додає
  uid у масив `deletedFor` (`:177`); очищення розмови записує `clearedAt.{uid}` і прибирає uid з
  `visibleTo` (`:455`).
- `RealtimeDbService` (`data/remote/firebase/RealtimeDbService.kt`) - присутність, набір і
  `.info/connected`. Присутність використовує `onDisconnect()`, тож сервер позначає користувача
  офлайн, якщо сокет обірветься (`:52`). Читання - це `callbackFlow` з `ValueEventListener`.
- `FirebaseStorageService` - URL завантаження/вивантаження медіа (використовує
  `MediaTransferManager`).
- `FirebaseMessagingManager` / `MyFirebaseMessagingService` / `FcmTokenSyncer` - FCM; див.
  [[Notifications_UA|Сповіщення]].
- `FirebaseCallSignalingService` - сигналізація дзвінків поверх RTDB; вона також записує запис
  `callChannels/<channel>` (абонент, адресат, серверна позначка часу) тим самим мультишляховим
  оновленням, що й сигнал (`callSignalUpdate`,
  `data/remote/call/FirebaseCallSignalingService.kt:159`), який функція `getRtcToken` читає, щоб
  вирішити, кому можна видати токен дзвінка; див.
  [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].
- `RemoteConfigFeatureFlags` і `CrashlyticsUserSync` - клієнт Remote Config за feature flag-ами та
  хук user id Crashlytics; див. [[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]].
- `DeviceRegistry` (`data/device/DeviceRegistry.kt`) - записує рядок цього пристрою (з його
  FCM-токеном) у `users/<uid>/devices` і очищає токен при виході.

Присутність і набір тексту до доменних інтерфейсів адаптують `FirebasePresenceService` (у
`data/presence/`, поруч із `PresenceManager` і `AppLifecycleObserver`) та `FirebaseTypingService` (у
`data/service/`, де також лежать `AndroidSessionCleaner` і `InMemoryAuthFlowGate`); обидва адаптують
`RealtimeDbService` і відфільтровують поточного користувача; див.
[[Presence and Typing_UA|Присутність і набір тексту]].

## DTO та маппери

`data/remote/dto/` містить провідні (wire) типи Firestore: `RemoteMessageDto` (у `dto/message/RemoteMessageDto.kt`,
поруч із `RemoteMediaItemDto` і `RemoteClipDto`),
`ConversationDTO`, `UserDTO`, `CallSignalDto`. Розширення-маппери `toDomain()` / `toRemoteDto()`
конвертують у доменні моделі й назад і несуть поля шифрування (`enc`, `ciphertext`, `nonce`,
`senderEpoch`, `recipientEpoch`, `recipientId`), які читає/пише кодек. DTO також несе `deletedFor`
і прапорець `deleted`, що використовуються правилами видалення.

## Черга офлайн-синхронізації

Коли запис не може дійти до Firestore, репозиторій ставить у чергу `ObxSyncQueueItem`
(`entityType`, `entityId`, `action`, `timestamp`, `retryCount`) - див. `enqueueRetry`
(`data/repository/MessageRepositoryImpl.kt:80`). Чергу осушують:
- `SyncCoordinator` (`data/sync/SyncCoordinator.kt`) - `@Singleton`, запущений у
  `MessengerApplication.onCreate`; на власному `Default`-scope він запускає одноразову синхронізацію
  на старті й знову щоразу, коли повертається мережа (`collectLatest` над `INetworkObserver`).
- `SyncManager` (`data/sync/SyncManager.kt`) - планує періодичний (15-хвилинний, обмежений
  мережею, з експоненційним backoff, `KEEP`) `SyncWorker` і може запускати одноразові синхронізації.
- `SyncWorker` (`data/sync/SyncWorker.kt`) - `@HiltWorker` `CoroutineWorker`, що делегує
  `SyncAllDataUseCase` і повертає `Result.retry()` за збою, тож WorkManager відкладає повтор.
- `NetworkObserver` / `INetworkObserver` - Flow зв'язності, який використовує координатор.

`@HiltWorker` потребує `androidx.hilt:hilt-compiler` (`app/build.gradle.kts:167`) і
`HiltWorkerFactory`, зв'язаного через `MessengerApplication` `Configuration.Provider`
(`MessengerApplication.kt:150`); інакше WorkManager не може сконструювати воркер із його
залежністю `SyncAllDataUseCase`.

## Передавання й кешування медіа

`data/media/`:
- `MediaTransferManager` - виконує завантаження/вивантаження через `FirebaseStorageService`,
  відкриваючи `StateFlow<Map<String, MediaTransfer>>` прогресу, який `MediaRepositoryImpl.transfers`
  повторно відкриває нагору.
- `MediaCache` - дисковий кеш оригіналів/завантажень за ключем id елемента й типу
  (`cacheDir/chat_media`, `MediaCache.DIR`). Для голосових кліпів він також володіє двома теками в
  `filesDir`, що мають пережити очищення кешу: `clip_outbox` (`OUTBOX_DIR`, ненадіслані кліпи до
  підтвердження надсилання, потім переносяться в кеш) і `voice_drafts` (`DRAFTS_DIR`).
  `deleteStaleTemporaries` (`data/media/MediaCache.kt:62`) прибирає залишки `recording_`, `.dl`,
  `.part` і `.enc` під час запуску.
- `MediaFileTransfer` - завантаження/вивантаження одного файлу з шифруванням; завантаження йдуть
  через унікальний тимчасовий файл і атомарне переміщення, тож збій ніколи не лишає напівзаписаний
  файл під кінцевим ім'ям.
- `VoiceRecorder`, `VoiceDraftStore`, `ClipPlaybackController` - singleton-и запису голосу,
  збереження чернеток (`<conversationId>.m4a` + JSON-супровідник) і відтворення через ExoPlayer.
  Див. [[Voice Messages_UA|Голосові повідомлення]].
- `MediaMetadata` - витягує ширину/висоту/тривалість і BlurHash для обраного файлу.
- `BlurHash` / `BlurHashCache` - компактні розмиті плейсхолдери, показані до завантаження
  зображення.
- `MediaStoreGallery` - читає галерею пристрою для пікера.

`MediaCipher` (`data/crypto/`) шифрує/розшифровує медіафайли ключем на файл, коли розмова піддається
шифруванню; ключ подорожує всередині зашифрованого корисного навантаження повідомлення. Див.
[[Chat and Messaging_UA|Чат і обмін повідомленнями]] та [[End-to-End Encryption_UA|Наскрізне шифрування]].

## Пов'язане

[[Domain Layer_UA|Доменний шар]] - [[Async and Reactivity_UA|Асинхронність і реактивність]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] -
[[Chat and Messaging_UA|Чат і обмін повідомленнями]] - [[Notifications_UA|Сповіщення]] -
[[Presence and Typing_UA|Присутність і набір тексту]] -
[[End-to-End Encryption_UA|Наскрізне шифрування]] - [[Voice Messages_UA|Голосові повідомлення]] -
[[Architecture Overview_UA|Огляд архітектури]]
