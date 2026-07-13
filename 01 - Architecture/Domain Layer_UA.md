---
tags:
  - _UA
---
# Доменний шар

`domain/` - це незалежне від застосунку ядро: use case, інтерфейси, які реалізує шар даних, і
прості моделі, що перетинають шари. Він не має імпортів Android чи Firebase.

## Use case

Use case - це клас із єдиною відповідальністю, конструкторно впровадженими залежностями й одним
публічним `operator fun invoke`, тож місця виклику читаються як виклик функції. Вони живуть під
`domain/usecase/`, згруповані за областями (`auth`, `conversation`, `message`, `presence`,
`sync`, `typing`, `user`).

Зауваження: ім'я файлу й ім'я класу не завжди збігаються. `GetMessageUseCase.kt` містить клас
`GetMessagesUseCase`; `ObserveConversationsUseCase.kt` / `GetConversationUseCase.kt` стоять за
`GetConversationsUseCase`, який впроваджує ViewModel. Завжди довіряйте імені класу.

Дві репрезентативні форми:

Потоковий use case - обгортає потік репозиторію й додає обрамлення завантаження/помилки.
`GetMessagesUseCase` (`domain/usecase/message/GetMessageUseCase.kt:13`):

```kotlin
operator fun invoke(conversationId: String): Flow<Resource<List<Message>>> {
    if (conversationId.isBlank()) return flow { emit(Resource.Error("Invalid conversation ID")) }
    return messageRepository.getMessagesStream(conversationId)
        .map<List<Message>, Resource<List<Message>>> { Resource.Success(it) }
        .onStart { emit(Resource.Loading) }
        .catch { emit(Resource.Error(it.message ?: "Failed to load messages")) }
}
```

Мета: віддати UI потік, що вже говорить мовою `Resource` (Loading / Success / Error), тож ViewModel
не мусить try/catch навколо collect. Він повертає холодний `Flow`, бо лише декорує холодний
ObjectBox-потік репозиторію (див. [[Async and Reactivity_UA|Асинхронність і реактивність]]);
`onStart` засіває `Loading` перед першим випромінюванням з БД, `catch` перетворює збій потоку на
термінальний `Resource.Error`, а не крах.

Suspend use case, що продукує потік результатів - `SendMessageUseCase`
(`domain/usecase/message/SendMessageUseCase.kt:17`) валідує ввід (не порожній, <= 5000 символів),
резолвить поточного користувача через `GetCurrentUserUseCase`, будує `Message.Text` (новий `UUID`,
`MessageStatus.SENDING`, сніпет відповіді, обрізаний до 200 символів), викликає
`messageRepository.sendMessage` і випромінює `Resource.Success/Error` з `flow { }`.

Use case компонуються з інших use case (`SendMessageUseCase` залежить від
`GetCurrentUserUseCase`), що тримає оркестрацію в домені, а не у ViewModel. Їх постачає
`di/UseCaseModule.kt` (конструкторне впровадження, тож більшості не потрібен явний `@Provides`).

## Інтерфейси репозиторіїв

`domain/repository/` визначає контракти, які виконує шар даних:
`IAuthRepository`, `IConversationRepository`, `IMessageRepository`, `IMediaRepository`,
`ISyncRepository`, `IUserRepository`. ViewModel чи use case залежить від інтерфейсу; Hilt
прив'язує `*Impl` (див. [[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] та
[[Data Layer_UA|Шар даних]]). Це шов, що тримає ObjectBox і Firebase поза доменом.

## Інтерфейси сервісів

`domain/service/` абстрагує сервіси, специфічні для пристрою й вендора, тож шари домену й
представлення не імпортують Agora чи Firebase напряму:
- `ICallService` - RTC-рушій (join/leave channel, mute, camera, прив'язка відео). Прив'язаний до
  `AgoraCallService`.
- `ICallSignalingService` - сигналізація дзвінків поверх Realtime Database. Прив'язаний до
  `FirebaseCallSignalingService`.
- `IPresenceService`, `ITypingService` - присутність і набір. Прив'язані до
  `FirebasePresenceService` / `FirebaseTypingService`.
- `ISessionCleaner` - локальне очищення під час виходу.

`CallEventListener` і `CallConnectionState` теж живуть під `domain/service/`, тож поверхня колбеків
Agora виражена в доменних термінах, а `AgoraCallService`
(`data/remote/call/AgoraCallService.kt`) відображає константи Agora на них.

## Доменні моделі

`domain/model/` містить типи, що перетинають шари. Центральний - `Message`, sealed-ієрархія з
варіантами `Message.Text`, `Message.Media` та `Message.Call` (згадується скрізь, наприклад
`when (message)` у `MessageRepositoryImpl.previewLabel`,
`data/repository/MessageRepositoryImpl.kt:199`). Інші: `Conversation`, `User`, `DomainUser`,
`Profile`, `MediaItem`, `MediaTransfer`, `MessageStatus` (SENDING/SENT/READ/FAILED),
`CallHistoryEntry`, `CallSignal`, `CallStatus`, `PresenceState` (ONLINE/AWAY/OFFLINE),
`UserPresence`, `PhoneNumber`, `PhoneVisibility`, `SyncStatus`, а також константи синхронізації під
`domain/model/sync/` (`SyncAction`, `SyncEntityType`).

Оскільки це звичайні типи Kotlin, вони одночасно є станом, який рендерить UI (`ChatUiState` тримає
`List<Message>` напряму), і значеннями, які продукують маппери Firestore/ObjectBox. Дискримінатори
типу повідомлення, що використовуються на шляхах дзвінків/історії, живуть як константи на `Message`
(`TYPE_TEXT`, `TYPE_MISSED_CALL`, `TYPE_UNREACHED_CALL`, `TYPE_ENDED_CALL`; див.
`data/repository/CallHistoryRepository.kt:67`).

## Навіщо взагалі доменний шар

- Тестовність - use case це звичайні класи з фейковими репозиторіями; Robolectric не потрібен.
- Замінюваність - шви `I*Repository` / `I*Service` дають шару даних змінюватися (Room -> ObjectBox
  вже сталося; Agora чи Firebase можна замінити) без дотику до представлення.
- Єдине місце для оркестрації - багатокрокові потоки (валідувати, резолвити користувача, надіслати,
  повторити) живуть у use case, а не розкидані по ViewModel.

Прийнятий компроміс: багато маленьких класів і певний дрейф імен файлів/класів. Проєкт тримає
патерн легким (без окремої церемонії маппінгу `Result` поза `Resource` і котлінівським `Result`).

## Пов'язане

[[Data Layer_UA|Шар даних]] - [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] -
[[Architecture Overview_UA|Огляд архітектури]]
