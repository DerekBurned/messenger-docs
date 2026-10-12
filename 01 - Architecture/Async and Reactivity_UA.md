---
tags:
  - _UA
---
# Асинхронність і реактивність

Кожен асинхронний примітив у застосунку, з властивістю, що змусила зробити цей вибір, і реальним
місцем виклику. Версія корутин - `1.11.0` (`gradle/libs.versions.toml:28`).

## Flow (холодний) - потік, що виконує роботу на кожного збирача

Холодний `Flow` стартує заново для кожного збирача і зупиняється, коли збирання припиняється.
Використовується там, де джерело - це запит або слухач, який має бути активним лише поки хтось
дивиться.

Канонічний приклад - локальний потік повідомлень, `MessageRepositoryImpl.getMessagesStream`
(`data/repository/MessageRepositoryImpl.kt:34`):

```kotlin
override fun getMessagesStream(conversationId: String): Flow<List<Message>> {
    return queryFlow {
        messageBox.query(
            ObxMessage_.conversationId.equal(conversationId)
                .and(ObxMessage_.deleted.equal(false)),
        )
            .order(ObxMessage_.timestamp)
            .build()
    }
        .map { rows -> rows.map { it.toDomain() } }
        .flowOn(Dispatchers.Default)
}
```

Мета: віддати повідомлення розмови як живий список, що повторно випромінюється щоразу, коли
локальна база змінюється, без опитування з боку UI. Він повертає `Flow` (холодний), бо підписка
ObjectBox має існувати лише поки екран збирає; коли користувач залишає чат, збирання
завершується, а нативна підписка розбирається (див. `queryFlow` нижче).

Чому `flowOn(Dispatchers.Default)`: `map { it.toDomain() }` конвертує кожен рядок у доменну модель
- чиста робота CPU. `flowOn` переносить upstream (випромінювання запиту й маппінг) на диспетчер
`Default` (CPU), тож маппінг ніколи не виконується в головному потоці, а збирач усе одно отримує
дані в тому контексті, у якому збирає. `Default`, а не `IO`, бо це обчислення над уже отриманими
рядками, а не блокуючий IO.

## `callbackFlow` - міст із listener-API у Flow

`callbackFlow` - це інструмент для адаптації колбека register/unregister у холодний Flow. Через
нього мостяться три сімейства слухачів.

### Підписка на запит ObjectBox - `ObxFlow.asFlow`

`data/local/obx/ObxFlow.kt:8`:

```kotlin
fun <T> Query<T>.asFlow(): Flow<List<T>> = callbackFlow {
    val query = this@asFlow
    val subscription = subscribe().observer { data -> trySend(data) }
    awaitClose {
        subscription.cancel()
        query.close()
    }
}
```

Мета: перетворити `Query` ObjectBox на Flow, що випромінює повний набір результатів зараз і знову
на кожну зміну відповідних рядків. `callbackFlow` обов'язковий (а не `flow { }`), бо дані
прибувають через колбек `observer` ObjectBox, а не витягуванням.

Два прибирання в `awaitClose`, обидва критичні:
- `subscription.cancel()` зупиняє ObjectBox від виклику колбека після того, як збирач зник.
- `query.close()` звільняє нативний хендл `Query`. Рядок `val query = this@asFlow` існує саме
  щоб це резолвилось у `io.objectbox.query.Query.close()`, а не в `ProducerScope.close()`, який
  `callbackFlow` теж вносить у скоуп. Без захоплення ресивера некваліфікований `close()`
  прив'язується до каналу, а нативний хендл запиту тече на кожен відписник. `awaitClose`
  виконується, коли збирач скасовує, тож обидва прибирання прив'язані до тривалості життя збирача.

`trySend` (а не `send`), бо колбек observer - не suspend-контекст; `trySend` не блокує і за
стандартного буферизованого каналу за звичайного навантаження нічого не відкидає.

`queryFlow` (`ObxFlow.kt:17`) - той самий міст з однією відмінністю: він приймає білдер і викликає
`build()` всередині `callbackFlow`, тож кожне збирання отримує власний свіжий `Query`. З `asFlow`
`Query` створюється один раз і закривається в `awaitClose`, тож повторне збирання того самого Flow
підписалося б на закритий хендл. Репозиторії використовують `queryFlow` для потоків, які екран може
збирати більше одного разу (`getMessagesStream`, списки нещодавніх, id видалених кліпів).

### Snapshot listener Firestore - `FirestoreService.getRecentMessagesStream`

`data/remote/firebase/FirestoreService.kt:251`. `callbackFlow` обгортає
`addSnapshotListener(MetadataChanges.INCLUDE)`:

```kotlin
val listener = messagesRef.addSnapshotListener(MetadataChanges.INCLUDE) { snapshot, error ->
    if (error != null) { close(error); return@addSnapshotListener }
    if (snapshot != null) {
        val rows = snapshot.documents.mapNotNull { doc ->
            doc.toObject(RemoteMessageDto::class.java)?.copy(id = doc.id)
                ?.let { it to doc.metadata.hasPendingWrites() }
        }.reversed()
        trySend(rows)
    }
}
awaitClose { listener.remove() }
```

`MetadataChanges.INCLUDE` змушує слухача спрацьовувати також для локальних, ще не підтверджених
записів, а `doc.metadata.hasPendingWrites()` позначає кожен рядок як "очікуваний", тож оптимістичні
надсилання рендеряться до підтвердження сервера. `close(error)` пропагує збій прослуховування як
виняток Flow; `awaitClose { listener.remove() }` від'єднує слухача Firestore, коли збирання
завершується. Потік сирих DTO потім `map`-иться через E2EE-кодек
(`data/remote/firebase/FirestoreService.kt:281`) - див. [[End-to-End Encryption_UA|Наскрізне шифрування]].

`getAllConversations` (`data/remote/firebase/FirestoreService.kt:493`) має ту саму форму і
додатково виносить id видалених документів та `snapshot.metadata.isFromCache` у значення
`ConversationSync`.

### ValueEventListener Realtime Database - присутність/набір/з'єднання

`data/remote/firebase/RealtimeDbService.kt` мостить три посилання RTDB через `callbackFlow` +
`ValueEventListener`. Стан з'єднання (`RealtimeDbService.kt:23`):

```kotlin
fun observeConnectionState(): Flow<Boolean> = callbackFlow {
    val listener = object : ValueEventListener {
        override fun onDataChange(snapshot: DataSnapshot) {
            trySend(snapshot.getValue(Boolean::class.java) ?: false)
        }
        override fun onCancelled(error: DatabaseError) { close(error.toException()) }
    }
    connectedRef.addValueEventListener(listener)
    awaitClose { connectedRef.removeEventListener(listener) }
}
```

`observePresence` (`RealtimeDbService.kt:59`) і `observeTyping` (`RealtimeDbService.kt:97`)
дотримуються ідентичного патерну. У кожному разі `awaitClose` знімає слухача, тож спостерігач у
фоні припиняє отримувати пуші RTDB.

## StateFlow (гарячий, конфлюючий, завжди має значення)

Використовується там, де завжди має бути поточне значення для читання, а нові підписники мають
одразу отримувати останнє.

- Стан UI у ViewModel - `MutableStateFlow(initialState)`, відкритий через `asStateFlow()`
  (`presentation/base/mvi/MviViewModel.kt:19`). UI потрібне значення на першому кадрі; голий `Flow`
  залишив би екран порожнім до першого випромінювання. Див.
  [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]].
- Глобальний стан дзвінка - `ActiveCallHolder` (`data/remote/call/ActiveCallHolder.kt:37`) - це
  `object` із `MutableStateFlow<ActiveCall?>`. Foreground-сервіс дзвінка - єдиний писар; UI
  (`presentation/navigation/graph/MainDisplay.kt:147`, `ActiveCallBar`, `CallViewModel`) - читач.
  `StateFlow` пасує, бо "поточний дзвінок" - це стан із визначеним поточним значенням (можливо
  `null`), який читають з багатьох місць. Див. [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].
- Режим показу панелі дзвінка - `CallBarPresenter.mode`
  (`presentation/components/call/CallBarPresenter.kt:11`) - це `MutableStateFlow<CallBarMode>`
  (BAR / HIDDEN / BUBBLE).
- Прогрес передавання медіа - `MediaTransferManager.transfers`, повторно відкритий як
  `IMediaRepository.transfers: StateFlow<Map<String, MediaTransfer>>`
  (`data/repository/MediaRepositoryImpl.kt:60`).
- Запис і відтворення голосу - `VoiceRecorder.state` (`data/media/VoiceRecorder.kt:109`) та
  `ClipPlaybackController.state` (`data/media/ClipPlaybackController.kt:111`) - singleton-и рівня
  застосунку, тож композер чату, голосові бульбашки й міні-плеєр на всіх вкладках читають одне
  поточне значення. Див. [[Voice Messages_UA|Голосові повідомлення]].

`_state.update { }` (`presentation/base/mvi/MviViewModel.kt:36`; `ChatsViewModel` робить те саме
зі своїм `_uiState`, `presentation/viewmodel/ChatsViewModel.kt:67`) - це атомарне оновлення
compare-and-set, що робить конкурентні редьюсери безпечними.

## SharedFlow проти Channel для одноразових подій

Два механізми для "події, а не стан", обрані по місцю:

- `SharedFlow(replay = 0, extraBufferCapacity = 16, onBufferOverflow = DROP_OLDEST)` для ефектів
  MVI (`presentation/base/mvi/MviViewModel.kt:22`). Нульовий replay, щоб повернутий екран не спрацював
  ефект знову; буферизований + non-suspending emit, тож `emitEffect` ніколи не блокує.
- `Channel(Channel.BUFFERED).receiveAsFlow()` для сигналу виходу з екрана дзвінка в `CallViewModel`
  (`presentation/viewmodel/CallViewModel.kt:51`). `Channel` односпоживчий із гарантованою
  доставкою, що відповідає "навігувати геть рівно один раз".
- `Channel(Channel.CONFLATED)` для переходів присутності в `PresenceManager`
  (`data/presence/PresenceManager.kt:24`). CONFLATED тримає лише останній запитаний стан, тож
  швидкий сплеск online->away->online згортається до фінального значення перед застосуванням.

## Оператори мосту холодний/гарячий

- Білдери `flow { }` для одноразової suspend-роботи, обгорнутої як потік: `SendMessageUseCase`
  (`domain/usecase/message/SendMessageUseCase.kt:21`) випромінює `Resource.Loading/Success/Error`;
  `MessageRepositoryImpl.observeRecentMessages` (`data/repository/MessageRepositoryImpl.kt:140`)
  обгортає collect у `flow { }.flowOn(Dispatchers.IO)` (тут IO, бо це керує мережевим
  snapshot-потоком і записує рядки).
- `combine` для злиття потоків: `ChatViewModel.observePartnerPresence`
  (`presentation/viewmodel/ChatViewModel.kt:455`) комбінує потік присутності з 60-секундним
  `tickerFlow`, тож напис "останній раз у мережі N хвилин тому" оновлюється за таймером, навіть
  коли сама присутність не змінилася; `FirebasePresenceService.observeMultiplePresence`
  (`data/service/FirebasePresenceService.kt:30`) комбінує список потоків присутності по
  користувачах в єдиний `Map`.
- `onEach { }.launchIn(scope)` для збирання "запустив і забув" у стан:
  `ChatViewModel.observePartnerAlias` (`presentation/viewmodel/ChatViewModel.kt:151`) і
  `observeTransfers` (`presentation/viewmodel/ChatViewModel.kt:479`).
- `collectLatest` там, де нове випромінювання має скасувати роботу, розпочату для попереднього:
  `CallForegroundService.observeRemoteStatus`
  (`data/remote/call/CallForegroundService.kt:352`), `CallViewModel.observeActiveCall`
  (`presentation/viewmodel/CallViewModel.kt:103`), `SyncCoordinator`
  (`data/sync/SyncCoordinator.kt:40`).
- `onStart` / `catch` для обрамлення потоку станом завантаження та обробкою помилок:
  `ChatsViewModel.loadConversations`
  (`presentation/viewmodel/ChatsViewModel.kt:81`), `GetMessagesUseCase`
  (`domain/usecase/message/GetMessageUseCase.kt:16`).
- `withTimeoutOrNull` для обмеження очікування: `MediaRepositoryImpl.uploadAndSend`
  (`data/repository/MediaRepositoryImpl.kt:180`) обмежує завантаження кількох файлів 35 секундами;
  `ChatViewModel.jumpToMessage` (`presentation/viewmodel/ChatViewModel.kt:357`) обмежує кожне
  завантаження сторінки під час пошуку назад. Голосові кліпи натомість масштабують очікування з
  розміром файлу: `clipUploadTimeout` (`data/repository/ClipRepositoryImpl.kt:389`) - це 30 с плюс
  одна секунда на кожні 8000 байтів, а наступний запис у Firestore підтверджується через
  `FirestoreService.sendMessage(..., confirmWithin = 20.seconds)`, що обгортає `write.await()` у
  `withTimeoutOrNull` (`data/remote/firebase/FirestoreService.kt:152`).
- Спільний `Deferred` на ключ для дедуплікації паралельної роботи: `InFlight`
  (`data/repository/ClipRepositoryImpl.kt:392`) тримає один `async(start = CoroutineStart.LAZY)` на
  id повідомлення, тож дві бульбашки, що просять той самий кліп, завантажують його один раз і обидві
  чекають той самий результат. Запис прибирає себе в `invokeOnCompletion`, тож збій не кешується.
- `CompletableDeferred`, щоб перетворити колбек слухача на один очікуваний результат:
  `VideoNoteRecorder.Session.result` (`data/media/VideoNoteRecorder.kt:109`) завершується рівно
  один раз асинхронним `VideoRecordEvent.Finalize` від CameraX. `stop` чекає його під
  `withTimeoutOrNull(5_000)` у `try`/`finally` (`:183`): якщо корутину, що чекає, скасовано
  (ViewModel очищено) або вийшов час, `finally` зберігає кліп як чернетку замість того, щоб його
  загубити. Див. [[Video Messages_UA|Відеоповідомлення]].
- `withContext(NonCancellable)`, щоб захистити спільний стан від скасування:
  `ProcessCameraProvider.awaitInstance` обгорнуто в нього (`VideoNoteRecorder.kt:211`), бо
  скасування корутини, що чекає, скасувало б глобальний для процесу future ініціалізації CameraX і
  зламало б камеру до завершення процесу. Після цього корутина перевіряє, чи її сесія ще поточна,
  і відкидає результат, якщо ні.

## Скоупи й диспетчери - який і чому

- `viewModelScope` - уся робота ViewModel (`ChatViewModel`, `ChatsViewModel`,
  `AuthViewModel`, `CallViewModel`). Скасовується автоматично в `onCleared`, тож ніщо не переживає
  екран. Ручні хендли `Job` (наприклад `typingHeartbeatJob` / `typingStopJob`,
  `presentation/viewmodel/ChatViewModel.kt:93`) скасовуються явно для дебаунсу.
- `@Singleton`, що володіє власним `CoroutineScope(SupervisorJob() + Dispatchers.IO)` -
  використовується там, де робота має пережити будь-який окремий екран. `MediaRepositoryImpl`
  (`data/repository/MediaRepositoryImpl.kt:57`): завантаження, розпочате з екрана чату, має
  завершитися навіть після того, як користувач пішов, тож воно не може використовувати
  `viewModelScope`. `SupervisorJob` означає, що одне невдале завантаження не скасовує інші;
  `Dispatchers.IO`, бо завантаження - це блокуючий IO. Scope ніколи не скасовується - він живе
  весь застосунок, відповідно до тривалості `@Singleton`. `ClipRepositoryImpl`
  (`data/repository/ClipRepositoryImpl.kt:59`) має ту саму форму плюс `CoroutineExceptionHandler`,
  що логує, тож збій задачі голосового кліпу ніколи не обвалить процес; кожне надсилання працює як
  зареєстрований `Job` у `sendJobs`, тож повтор і скасування можуть його знайти.
- `CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)` - `VoiceRecorder`
  (`data/media/VoiceRecorder.kt:106`) і `ClipPlaybackController`
  (`data/media/ClipPlaybackController.kt:108`). Обидва керують платформним об'єктом (`MediaRecorder`,
  ExoPlayer) і слухачем аудіофокусу; обмеження кожного виклику й оновлення стану головним потоком
  серіалізує start/stop/interrupt та колбеки плеєра без блокувань, а ExoPlayer треба
  використовувати з того потоку, на якому його створено. `VideoNoteRecorder`
  (`data/media/VideoNoteRecorder.kt:112`) використовує той самий scope з тієї ж причини; події
  CameraX приходять на головний executor (`ContextCompat.getMainExecutor`), тож кожен перехід
  сесії відбувається в одному потоці.
- `CoroutineScope(SupervisorJob() + Dispatchers.IO)` всередині `Service` - `CallForegroundService`
  (`data/remote/call/CallForegroundService.kt:57`) виконує машину станів дзвінка (записи
  сигналізації, рінгтони, таймаути) на цьому scope і скасовує його в `onDestroy`
  (`data/remote/call/CallForegroundService.kt:941`) через `scope.coroutineContext[Job]?.cancel()`,
  тож жоден job таймауту не переживає сервіс. Див. [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].
- `CoroutineScope(SupervisorJob() + Dispatchers.Default)` - `SyncCoordinator`
  (`data/sync/SyncCoordinator.kt:23`) спостерігає за потоком мережі на `Default` і запускає
  WorkManager; легка координація, а не блокуючий IO, звідси `Default`.
- Ефемерний `CoroutineScope(Dispatchers.IO)` на кожен колбек - `MyFirebaseMessagingService`
  (`data/remote/firebase/MyFirebaseMessagingService.kt:47`, `:92`). У колбека FCM немає під рукою
  ViewModel чи довгоживучого scope, тож кожне повідомлення розкручує короткоживучий IO-scope, щоб
  зберегти й сповістити. Див. [[Notifications_UA|Сповіщення]].

## Пов'язане

[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] - [[Data Layer_UA|Шар даних]] -
[[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]] -
[[Presence and Typing_UA|Присутність і набір тексту]] -
[[End-to-End Encryption_UA|Наскрізне шифрування]] - [[Architecture Overview_UA|Огляд архітектури]] -
[[Voice Messages_UA|Голосові повідомлення]]
