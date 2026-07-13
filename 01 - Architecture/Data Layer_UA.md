---
tags:
  - _UA
---
# Шар даних

`data/` реалізує кожен доменний інтерфейс і є єдиним шаром, що торкається ObjectBox, Firebase,
Agora, crypto-кодека, передавання медіа та ОС. Саме тут забезпечується модель offline-first.

Підпакети: `local/` (ObjectBox + prefs), `remote/` (auth, firebase, call, dto), `crypto/`
(E2EE), `media/`, `presence/`, `service/`, `repository/`, `sync/`.

## Принцип offline-first

Локальне сховище ObjectBox - джерело істини, яке читає UI. Мережа - писар і оновлювач за ним.
Метод репозиторію зазвичай:
1. одразу записує/оновлює ObjectBox (оптимістично), що
2. повторно випромінює локальний потік `Query.asFlow()`, який збирає UI, і лише тоді
3. пушить у Firestore, відкочуючись до черги синхронізації за збою.

`MessageRepositoryImpl.sendMessage` (`data/repository/MessageRepositoryImpl.kt:52`) - зразок:
`messageBox.put(...)`, потім `updateConversationPreview`, потім `firestoreService.sendMessage`; за
збою переводить рядок у `FAILED` і викликає `enqueueRetry`. Сторона читання ніколи не блокується на
мережі - див. аналіз потоків у [[Async and Reactivity_UA|Асинхронність і реактивність]].

## Сховище ObjectBox

Версія `5.4.2` (`gradle/libs.versions.toml:16`), обробка анотацій через kapt
(`app/build.gradle.kts:10`, `:163`). Сутності оголошені в `data/local/obx/ObxEntities.kt` і
генеруються в `MyObjectBox`.

Домовленості сутностей (`data/local/obx/ObxEntities.kt:14`): `@Id var boxId: Long` - внутрішній
ключ; бізнес-ключ - це окремий `@Unique(onConflict = ConflictStrategy.REPLACE) var uid`, тож
upsert того самого віддаленого id замінює рядок, а не дублює його. Стовпці, "гарячі" для запитів,
несуть `@Index` - `ObxMessage.conversationId` та `ObxMessage.type`
(`ObxEntities.kt:23`, `:26`); індекс `type` існує, щоб запит історії дзвінків (`type.oneOf(...)`)
не був повним скануванням таблиці. Складні поля зберігаються через `@Convert` property-конвертери
(наприклад `MediaItemListConverter` для `mediaItemsJson`, `PhoneNumberConverter`,
`StringListConverter`), тож `List<MediaItem>` зберігається як JSON в одному стовпці.

Сім боксів постачаються як singleton у `di/ObjectBoxModule.kt`: `provideBoxStore`
(`di/ObjectBoxModule.kt:29`) будує `MyObjectBox` і лише в debug відновлюється після
`DbSchemaException`, витираючи й перебудовуючи сховище - пониження схеми на dev-пристрої
самовиправляється замість краху під час запуску. Кожен `Box<Obx*>` - окремий
`@Provides @Singleton`, тож репозиторії впроваджують саме ті бокси, які їм потрібні.

Потокування з ObjectBox використовує розширення `Query<T>.asFlow()`
(`data/local/obx/ObxFlow.kt:8`); його прибирання `callbackFlow` + `awaitClose` розібрано в
[[Async and Reactivity_UA|Асинхронність і реактивність]].

Маппери живуть у `data/local/obx/ObxMappers.kt` (`toObx()` / `toDomain()`), тримаючи анотації
ObjectBox поза доменними моделями.

## Репозиторії

`data/repository/` (усі `@Singleton`, прив'язані в `di/RepositoryModule.kt`):
- `MessageRepositoryImpl` (`IMessageRepository`) - локальний потік + send/delete/read + оновлення
  Firestore + пагінація + злиття зі збереженням стану прочитання (`putPreservingReadState`,
  `data/repository/MessageRepositoryImpl.kt:169`). Правила видалення пояснено в
  [[Chat and Messaging_UA|Чат і обмін повідомленнями]].
- `MediaRepositoryImpl` (`IMediaRepository`) - володіє власним scope рівня застосунку для
  завантажень/вивантажень; див. [[Chat and Messaging_UA|Чат і обмін повідомленнями]] та
  обґрунтування scope в [[Async and Reactivity_UA|Асинхронність і реактивність]].
- `ConversationRepositoryImpl` (`IConversationRepository`) - потік списку розмов, create/find,
  видалення для себе проти для всіх через `visibleTo`/`clearedAt`.
- `UserRepositoryImpl` (`IUserRepository`) - пошук/lookup користувачів, псевдоніми контактів,
  спостереження за аватаром.
- `AuthRepositoryImpl` (`IAuthRepository`) - обгортка над `FirebaseAuthService`.
- `SyncRepositoryImpl` (`ISyncRepository`) - осушує чергу синхронізації.
- `CallHistoryRepository` - записи журналу дзвінків, похідні від повідомлень типу дзвінка
  (`data/repository/CallHistoryRepository.kt:24`); див. [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].
- `ConversationReconciler` - узгоджує віддалені знімки розмов із локальними рядками.

`data/local/repository/LocalRepositoryImpl` (`ILocalRepository`) централізує прямий доступ до
боксів, що використовується кількома репозиторіями.

## Сервіси Firebase

BOM `34.10.0` (`gradle/libs.versions.toml:13`) керує версіями Firebase. Сервіси обгортають SDK і
повертають доменні типи / `Result`.

- `FirestoreService` (`data/remote/firebase/FirestoreService.kt`) - користувачі, розмови й
  підколекція `conversations/{id}/messages`. Читання - це `callbackFlow` зі snapshot listener
  (`getRecentMessagesStream:204`, `getAllConversations:422`); записи - suspend-методи `Result`, що
  використовують `await()` з `kotlinx-coroutines-play-services`. Він інтегрує E2EE-кодек під час
  encode/decode. Видалення для всіх встановлює поле `deleted` (`:173`); видалення для себе додає
  uid у масив `deletedFor` (`:145`); очищення розмови записує `clearedAt.{uid}` і прибирає uid з
  `visibleTo` (`:373`).
- `RealtimeDbService` (`data/remote/firebase/RealtimeDbService.kt`) - присутність, набір і
  `.info/connected`. Присутність використовує `onDisconnect()`, тож сервер позначає користувача
  офлайн, якщо сокет обірветься (`:46`). Читання - це `callbackFlow` з `ValueEventListener`.
- `FirebaseStorageService` - URL завантаження/вивантаження медіа (використовує
  `MediaTransferManager`).
- `FirebaseMessagingManager` / `MyFirebaseMessagingService` / `FcmTokenSyncer` - FCM; див.
  [[Notifications_UA|Сповіщення]].
- `FirebaseCallSignalingService` - сигналізація дзвінків поверх RTDB; див.
  [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].

Пакет `data/service/` містить реалізації сервісів присутності/набору
(`FirebasePresenceService`, `FirebaseTypingService`), що адаптують `RealtimeDbService` до доменних
інтерфейсів і відфільтровують поточного користувача; див.
[[Presence and Typing_UA|Присутність і набір тексту]].

## DTO та маппери

`data/remote/dto/` містить провідні (wire) типи Firestore: `RemoteMessageDto` (з `MessageDTO.kt`),
`ConversationDTO`, `UserDTO`, `CallSignalDto`. Розширення-маппери `toDomain()` / `toRemoteDto()`
конвертують у доменні моделі й назад і несуть поля шифрування (`enc`, `ciphertext`, `nonce`,
`senderEpoch`, `recipientEpoch`, `recipientId`), які читає/пише кодек. DTO також несе `deletedFor`
і прапорець `deleted`, що використовуються правилами видалення.

## Черга офлайн-синхронізації

Коли запис не може дійти до Firestore, репозиторій ставить у чергу `ObxSyncQueueItem`
(`entityType`, `entityId`, `action`, `timestamp`, `retryCount`) - див. `enqueueRetry`
(`data/repository/MessageRepositoryImpl.kt:79`). Чергу осушують:
- `SyncCoordinator` (`data/sync/SyncCoordinator.kt`) - `@Singleton`, запущений у
  `MessengerApplication.onCreate`; на власному `Default`-scope він запускає одноразову синхронізацію
  на старті й знову щоразу, коли повертається мережа (`collectLatest` над `INetworkObserver`).
- `SyncManager` (`data/sync/SyncManager.kt`) - планує періодичний (15-хвилинний, обмежений
  мережею, з експоненційним backoff, `KEEP`) `SyncWorker` і може запускати одноразові синхронізації.
- `SyncWorker` (`data/sync/SyncWorker.kt`) - `@HiltWorker` `CoroutineWorker`, що делегує
  `SyncAllDataUseCase` і повертає `Result.retry()` за збою, тож WorkManager відкладає повтор.
- `NetworkObserver` / `INetworkObserver` - Flow зв'язності, який використовує координатор.

`@HiltWorker` потребує `androidx.hilt:hilt-compiler` (`app/build.gradle.kts:142`) і
`HiltWorkerFactory`, зв'язаного через `MessengerApplication` `Configuration.Provider`
(`MessengerApplication.kt:134`); інакше WorkManager не може сконструювати воркер із його
залежністю `SyncAllDataUseCase`.

## Передавання й кешування медіа

`data/media/`:
- `MediaTransferManager` - виконує завантаження/вивантаження через `FirebaseStorageService`,
  відкриваючи `StateFlow<Map<String, MediaTransfer>>` прогресу, який `MediaRepositoryImpl.transfers`
  повторно відкриває нагору.
- `MediaCache` - дисковий кеш оригіналів/завантажень за ключем id елемента й типу.
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
[[End-to-End Encryption_UA|Наскрізне шифрування]] - [[Architecture Overview_UA|Огляд архітектури]]
