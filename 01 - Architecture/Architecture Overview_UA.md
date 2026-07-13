---
tags:
  - _UA
---
# Огляд архітектури

## Форма застосунку

Єдиний модуль Gradle (`:app`, `app/build.gradle.kts`), один набір джерел Kotlin, кореневий
пакет `com.example.messenger`. У цьому корені код поділено на класичні три шари плюс DI і util:

```
com.example.messenger
  presentation/   Compose UI, ViewModels, MVI State/Intent/Effect, навігація, сповіщення
  domain/         use cases, інтерфейси репозиторіїв, інтерфейси сервісів, доменні моделі
  data/           реалізації репозиторіїв, ObjectBox, сервіси Firebase, crypto, media, sync, calls
  di/             модулі Hilt
  util/           Resource, DateUtils, валідація, схожість, константи
  MessengerApplication.kt   точка входу @HiltAndroidApp
```

Кількість файлів на момент написання: `presentation` 137, `data` 67, `domain` 55, `di` 10,
`util` 10 файлів Kotlin.

## Правило залежностей

Залежності спрямовані всередину. `presentation` залежить від `domain`; `data` залежить від
`domain`; `domain` не залежить ні від чого специфічного для застосунку. Шов - це набір
інтерфейсів у `domain/repository` та `domain/service` (наприклад `IMessageRepository`,
`IConversationRepository`, `ICallService`, `IPresenceService`). Реалізації живуть у `data` і
прив'язуються до цих інтерфейсів через Hilt - див. [[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]].
ViewModel ніколи не бачить тип `*Impl`; вона залежить від інтерфейсу (або від use case, що
залежить від нього).

Доменні моделі (`domain/model/`) - це "валюта", що перетинає шари. `data` конвертує між трьома
представленнями:
- сутності ObjectBox (`data/local/obx/ObxEntities.kt`, суфікс `Obx*`) для локального сховища,
- DTO Firestore (`data/remote/dto/`, наприклад `RemoteMessageDto`) для мережі,
- доменні моделі (`domain/model/`, наприклад `Message`, `Conversation`, `User`) для use case та
  UI.

Маппери (`data/local/obx/ObxMappers.kt`, `data/remote/dto/*` `toDomain`/`toRemoteDto`) роблять
переклад, тож ані анотації ObjectBox, ані форма полів Firestore не просочуються нагору.

## Три шари одним абзацом кожен

Presentation - екрани Compose спостерігають за `StateFlow<UiState>` від ViewModel і надсилають
назад `UiIntent`; одноразові `UiEffect` (навігація, показ помилки) доставляються через
`SharedFlow`. Повні деталі в [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] та
[[Navigation_UA|Навігація]].

Domain - use case з єдиною відповідальністю (`domain/usecase/...`) надають `operator fun invoke`
і оркеструють репозиторії. Це звичайні класи з конструкторним впровадженням, без залежностей від
Android. Повні деталі в [[Domain Layer_UA|Доменний шар]].

Data - репозиторії реалізують доменні інтерфейси і є єдиним місцем, що торкається ObjectBox,
Firestore, Realtime Database, Storage чи crypto-кодека. Це точка злиття моделі offline-first:
локальний ObjectBox - джерело істини, яке читає UI, а мережа - це писар/оновлювач за ним. Повні
деталі в [[Data Layer_UA|Шар даних]].

## Наскрізний потік даних: надсилання й показ текстового повідомлення

1. UI надсилає `ChatIntent.SendMessage(text)` до `ChatViewModel`
   (`presentation/viewmodel/ChatViewModel.kt:140`).
2. ViewModel викликає `SendMessageUseCase`
   (`domain/usecase/message/SendMessageUseCase.kt:17`), який будує `Message.Text` з `UUID`-id та
   `MessageStatus.SENDING`, а потім викликає репозиторій.
3. `MessageRepositoryImpl.sendMessage` (`data/repository/MessageRepositoryImpl.kt:52`) спершу
   записує повідомлення в ObjectBox (`messageBox.put`), оновлює прев'ю розмови, потім пробує
   запис у Firestore через `FirestoreService.sendMessage`
   (`data/remote/firebase/FirestoreService.kt:95`). За збою мережі рядок переводиться у `FAILED`
   і ставиться повторна спроба в чергу синхронізації.
4. Повідомлення вже на екрані, бо UI читає холодний потік ObjectBox: `getMessagesStream`
   (`data/repository/MessageRepositoryImpl.kt:34`) повертає `Query.asFlow()`, який повторно
   випромінює весь список на кожен локальний запис. `put` з кроку 3 запускає повторне
   випромінювання; `GetMessagesUseCase` у ViewModel його збирає і викликає `setState`.
5. Окремо `observeRecentMessages` / `getRecentMessagesStream`
   (`data/remote/firebase/FirestoreService.kt:204`) тримає відкритим snapshot listener Firestore
   і записує вхідні віддалені рядки в ObjectBox, який знову випромінює в той самий UI-потік.

Головна властивість: шлях читання і шлях запису розв'язані через ObjectBox. UI ніколи не чекає
мережу, щоб показати повідомлення. Див. [[Async and Reactivity_UA|Асинхронність і реактивність]]
щодо того, чому кожен потік холодний/гарячий і на якому диспетчері він працює, та
[[Chat and Messaging_UA|Чат і обмін повідомленнями]] щодо повної функції.

## Зв'язування під час запуску застосунку

`MessengerApplication` (`MessengerApplication.kt:34`) - це `@HiltAndroidApp` і
`Configuration.Provider` (він постачає `HiltWorkerFactory`, щоб `@HiltWorker`-воркери отримали
свої залежності). Його `onCreate` (`MessengerApplication.kt:61`) - це корінь композиції для
довгоживучого машинного механізму рівня застосунку, весь `@Inject`-ований як singleton:

- Coil `SingletonImageLoader` з кешем пам'яті на 10% RAM і дисковим кешем на 30 МБ, плюс
  `VideoFrameDecoder` для мініатюр відео.
- `NotificationChannels.ensureCreated` - реєструє канали заздалегідь (див.
  [[Notifications_UA|Сповіщення]]).
- `AppLifecycleObserver` на `ProcessLifecycleOwner` - керує присутністю
  foreground/background (див. [[Presence and Typing_UA|Присутність і набір тексту]]).
- `SyncCoordinator.start()` і `SyncManager.schedulePeriodicSync()` - механізм офлайн-синхронізації
  (див. [[Data Layer_UA|Шар даних]]).
- `FcmTokenSyncer.start()`, `IncomingCallCoordinator.start()`,
  `TelecomCallManager.registerPhoneAccount()` - обв'язка push та дзвінків.
- `E2eeBootstrap.publishKeys()` - публікує публічний ключ ідентичності цього пристрою (див.
  [[End-to-End Encryption_UA|Наскрізне шифрування]]).

Оскільки це зв'язується в `Application.onCreate`, воно живе весь процес, і саме тому деякі з цих
компонентів володіють власним `CoroutineScope`, а не позичають `viewModelScope` - див. розбір
scope у [[Async and Reactivity_UA|Асинхронність і реактивність]].

## Пов'язане

[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] - [[Domain Layer_UA|Доменний шар]] -
[[Data Layer_UA|Шар даних]] - [[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]] - [[Navigation_UA|Навігація]] -
[[Tech Stack and Versions_UA|Технологічний стек і версії]]
