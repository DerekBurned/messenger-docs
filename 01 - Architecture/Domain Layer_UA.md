---
tags:
  - _UA
---
# Доменний шар

`domain/` - це незалежне від застосунку ядро: use case, інтерфейси, які реалізує шар даних, і
прості моделі, що перетинають шари. Він майже вільний від типів Android і Firebase, але не повністю:
`ICallService` приймає Android `View` (`domain/service/ICallService.kt:3`),
`IUserRepository.uploadAvatar` приймає `Uri` (`domain/repository/IUserRepository.kt:3`), репозиторій і
use case входу за телефоном передають `PhoneAuthCredential` з Firebase
(`domain/repository/IAuthRepository.kt:6`), деякі моделі несуть Compose-анотацію `@Immutable`, а
охоронець feature flag-ів логує через `android.util.Log` (`domain/featureflag/FeatureFlagGuard.kt:3`).
Шви все одно тримають ObjectBox і Firestore SDK поза ним.

## Use case

Use case - це клас із єдиною відповідальністю, конструкторно впровадженими залежностями й одним
публічним `operator fun invoke`, тож місця виклику читаються як виклик функції. Вони живуть під
`domain/usecase/`, згруповані за областями (`auth`, `conversation`, `message`, `presence`,
`sync`, `typing`, `user`) плюс новий пакет `domain/featureflag/` (див. нижче).

Імена файлів і класів тепер збігаються. У ранніх редакціях коду `GetMessageUseCase.kt` містив клас
`GetMessagesUseCase`, а `GetConversationUseCase.kt` стояв за `GetConversationsUseCase`; обидва файли
перейменовано після 0.1.13.1 на `GetMessagesUseCase.kt` і `GetConversationsUseCase.kt`. Дві схожі
назви лишаються окремими класами: `GetConversationsUseCase` обгортає local-first
`getAllConversations()` (`ChatsViewModel`, `ForwardMessageViewModel`), а `ObserveConversationsUseCase`
обгортає `observeRemoteConversations()` (`EditChatViewModel`).

Дві репрезентативні форми:

Потоковий use case - обгортає потік репозиторію й додає обрамлення завантаження/помилки.
`GetMessagesUseCase` (`domain/usecase/message/GetMessagesUseCase.kt:13`):

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
`MessageStatus.SENDING`, сніпет відповіді з `Message.replySnippet()`), викликає
`messageRepository.sendMessage` і випромінює `Resource.Success/Error` з `flow { }`.

Use case, додані в 0.1.13.1:
- `ObserveSessionStateUseCase` (`auth/`) - поєднує стан автентифікації Firebase, шлюз потоку входу
  й віддалений профіль у `SessionState` (див. [[Navigation_UA|Навігація]]).
- `DeleteAccountUseCase` (`auth/`) - викликає серверне видалення, потім очищує пристрій лише при
  успіху (див. [[Settings_UA|Налаштування]]).
- `BackfillRecentHistoryUseCase` (`sync/`) - тягне нещодавні повідомлення для останніх розмов після
  входу (див. [[Authentication_UA|Автентифікація]]).
- `DeleteMessageUseCase` (`message/`) тепер також приймає `IClipRepository` і прибирає завантажений
  файл голосового кліпу після успішного видалення для всіх.
- `MarkConversationAsUnreadUseCase` (`conversation/`) живить свайп-дію "позначити непрочитаним".

`Enum` `FeatureFlag`, інтерфейс `FeatureFlags` (`StateFlow` обчислених прапорців), значення
`FeatureFlagState` і охоронець `FeatureFlags.refuses(flag)` живуть у `domain/featureflag/`, тож
ViewModel і шлюзи шару даних можуть перевірити перемикач без залежності від Firebase. Їх пояснено в
[[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]].

Use case компонуються з інших use case (`SendMessageUseCase` залежить від
`GetCurrentUserUseCase`), що тримає оркестрацію в домені, а не у ViewModel. Їх постачає
`di/UseCaseModule.kt` (конструкторне впровадження, тож більшості не потрібен явний `@Provides`).

## Інтерфейси репозиторіїв

`domain/repository/` визначає контракти, які виконує шар даних:
`IAuthRepository`, `IClipRepository`, `IConversationRepository`, `IMessageRepository`,
`IMediaRepository`, `ISyncRepository`, `IUserRepository`. `IClipRepository`
(`domain/repository/IClipRepository.kt`) - контракт голосових кліпів: `sendClip(..., replyTo)`,
`ensureLocal`, `markPlayed`, `nextVoiceClip`, `evictCachedCopy`, `observeDeletedClipIds`,
`deleteRemoteFile`, `retry`, `cancelSend` (див. [[Voice Messages_UA|Голосові повідомлення]]).
`IMessageRepository` отримав `fetchSharedMediaPage` (сторінки спільних медіа профілю чату, PR #54),
а `IClipRepository` - `cachedFile` і `registerRemoteClip`. `ILocalRepository` (`resetDB`) тепер живе в
`domain/repository/`, а `AuthAccountException` (`AlreadyRegistered` / `NotRegistered`, повертається
всередині `Resource.Failure` з `AuthRepositoryImpl.loginWithPhone`,
`data/repository/AuthRepositoryImpl.kt:58`, після PR #53) - у `domain/model/`. `IUserRepository`
отримав `observeRemoteUser` (для шлюзу сесії) і `observeCachedUsers` (для живих
рядків співрозмовників). ViewModel чи use case залежить від інтерфейсу; Hilt
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
- `ISessionCleaner` - локальне очищення під час виходу (`clearOnLogout`), плюс `clearOnDelete`,
  що також прибирає ключ ідентичності E2EE після видалення акаунта.
- `IAuthFlowGate` - `isBusy: Flow<Boolean>` і `withAuthFlow { }`, що позначають вхід як такий, що
  триває, тож шлюз сесії чекає. Прив'язаний до `InMemoryAuthFlowGate`.

`CallEventListener` і `CallConnectionState` теж живуть під `domain/service/`, тож поверхня колбеків
Agora виражена в доменних термінах, а `AgoraCallService`
(`data/remote/call/AgoraCallService.kt`) відображає константи Agora на них.

## Доменні моделі

`domain/model/` містить типи, що перетинають шари. Центральний - `Message`
(`domain/model/Message.kt:6`), sealed-ієрархія з варіантами `Message.Text`, `Message.Media`,
`Message.Clip` (`:57`) та `Message.Call` (згадується скрізь, наприклад `when (message)` у
`MessageRepositoryImpl.previewLabel`, `data/repository/MessageRepositoryImpl.kt:221`).
`Message.replySnippet()` (`:72`) - єдине правило для цитат відповіді. Навантаження кліпу - `Clip`
(`domain/model/voicemessage/Clip.kt:6`), sealed-клас із `Voice` (хвиля, ліміт 1 година
`MAX_DURATION_MS`, мітка "Voice message", `m4a`) і `Video` (круглі відеонотатки, ліміт 2 хвилини,
`mp4`, див. [[Video Messages_UA|Відеоповідомлення]]); кожен підтип
несе власні `previewLabel`, `fileExtension` і `maxDurationMs`, тож прев'ю, пуші й UI мають одне
джерело міток; `ClipKind` (`VOICE` / `VIDEO`, той самий пакет) - простий селектор, між яким
перемикається композер. Типи сесії: `SessionState` (`Resolving` / `SignedOut(reason)` / `Active(user)`) з
`SessionEndReason` (`PROFILE_MISSING`, `ACCOUNT_DELETED`) і `RemoteUserSnapshot(user,
authoritative)`. `User` отримав `deleted` / `deletedAt` для акаунтів із надгробком. Інші:
`Conversation`, `User`, `DomainUser`,
`Profile`, `MediaItem`, `MediaTransfer`, `MessageStatus` (SENDING/SENT/READ/FAILED),
`CallHistoryEntry`, `CallSignal`, `CallStatus`, `PresenceState` (ONLINE/AWAY/OFFLINE),
`UserPresence`, `PhoneNumber`, `PhoneVisibility`, `SyncStatus`, а також константи синхронізації під
`domain/model/sync/` (`SyncAction`, `SyncEntityType`).

Оскільки це звичайні типи Kotlin, вони одночасно є станом, який рендерить UI (`ChatUiState` тримає
`List<Message>` напряму), і значеннями, які продукують маппери Firestore/ObjectBox. Дискримінатори
типу повідомлення, що використовуються на шляхах дзвінків/історії, живуть як константи на `Message`
(`TYPE_TEXT`, `TYPE_MEDIA`, `TYPE_MISSED_CALL`, `TYPE_UNREACHED_CALL`, `TYPE_ENDED_CALL`,
`TYPE_VOICE_CLIP`, `TYPE_VIDEO_CLIP`; `Message.kt:81`-`:87`; типи дзвінків використовуються в
`data/repository/CallHistoryRepository.kt:79`).

## Навіщо взагалі доменний шар

- Тестовність - use case це звичайні класи з фейковими репозиторіями; Robolectric не потрібен.
- Замінюваність - шви `I*Repository` / `I*Service` дають шару даних змінюватися (Room -> ObjectBox
  вже сталося; Agora чи Firebase можна замінити) без дотику до представлення.
- Єдине місце для оркестрації - багатокрокові потоки (валідувати, резолвити користувача, надіслати,
  повторити) живуть у use case, а не розкидані по ViewModel.

Прийнятий компроміс: багато маленьких класів (і, до перейменувань після 0.1.13.1, певний дрейф імен
файлів/класів). Проєкт тримає
патерн легким (без окремої церемонії маппінгу `Result` поза `Resource` і котлінівським `Result`).

## Пов'язане

[[Data Layer_UA|Шар даних]] - [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] -
[[Architecture Overview_UA|Огляд архітектури]] - [[Voice Messages_UA|Голосові повідомлення]] -
[[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]]
