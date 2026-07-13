---
tags:
  - _UA
---
# Дзвінки (голос і відео)

Голосові й відеодзвінки один-на-один поверх Agora RTC, керовані foreground-сервісом, що володіє
машиною станів дзвінка. UI - повноекранний дзвінок, докована панель активного дзвінка з жестами
свайп/бульбашка/повернення, PiP-відео - це чисте відображення єдиного глобального об'єкта стану.
Пропущені/завершені/недосяжні дзвінки стають повідомленнями чату й живлять вкладку історії дзвінків.

## Складові

- `CallForegroundService` (`data/remote/call/CallForegroundService.kt:45`) - машина станів і
  власник життєвого циклу. `@AndroidEntryPoint`.
- `ActiveCallHolder` (`data/remote/call/ActiveCallHolder.kt:9`) - єдиний глобальний
  `StateFlow<ActiveCall?>`.
- `AgoraCallService` (`data/remote/call/AgoraCallService.kt:20`) - RTC-обгортка `ICallService`.
- `FirebaseCallSignalingService` (`ICallSignalingService`) - сигналізація дзвінків поверх
  Realtime DB.
- `AgoraTokenProvider`, `CallUids`, `MissedCallRecorder`, `IncomingCallCoordinator`.
- Інтеграція з Telecom: `data/remote/call/telecom/` (`TelecomCallManager`,
  `MessengerConnectionService`, `MessengerConnection`) для системного phone-account/PiP.
- Представлення: `CallViewModel` (`presentation/viewmodel/CallViewModel.kt:48`), `CallScreen`,
  `CallsViewModel`/`CallsScreen` і компоненти в `presentation/components/call/`
  (`ActiveCallBar`, `CallBarPresenter`, `CallBubble`, `CallControls`, `IncomingCallBar`,
  `PipCallSurface`).

## Agora RTC

Залежність `io.agora.rtc:full-sdk:4.6.3`, оголошена напряму в `app/build.gradle.kts:174`
(не в каталозі версій). Повний SDK, бо відео потребує нативних відеомодулів; App ID резолвиться з
`local.properties` -> Gradle-property -> плейсхолдер у `BuildConfig.AGORA_APP_ID`
(`app/build.gradle.kts:22`).

`AgoraCallService` створює один `RtcEngine` у своєму `init` (`AgoraCallService.kt:105`) і адаптує
колбеки `IRtcEngineEventHandler` у доменний `CallEventListener` (`AgoraCallService.kt:29`):
`onUserJoined -> onRemoteUserJoined`, `onUserOffline` розділяється на навмисний вихід
(`onRemoteUserLeft`) проти обриву мережі (`onRemoteConnectionLost`),
`onConnectionStateChanged -> onConnectionStateChanged`, `onTokenPrivilegeWillExpire ->` оновлення
токена. `joinChannel` (`:129`) встановлює `ChannelMediaOptions` (профіль комунікації, роль
broadcaster, публікація мікрофона, публікація камери коли відео увімкнено). Відео вмикається ліниво
(`ensureVideoModule`, `:204`), тож дзвінки лише-аудіо ніколи не розкручують відеоконвеєр.

## Машина станів живе у foreground-сервісі

`ActiveCallHolder` - це `object` із `MutableStateFlow<ActiveCall?>` (`ActiveCallHolder.kt:37`). Його
`set`/`update`/`clear` - `internal` - пише лише сервіс; усе інше читає `state` або `snapshot()`. Це
ключове архітектурне рішення: сервіс - єдиний писар стану дзвінка, а UI - stateless-дзеркало, тож
екран, сповіщення й панель ніяк не можуть розійтися.

`CallForegroundService` виконує все на `CoroutineScope(SupervisorJob() + Dispatchers.IO)`
(`CallForegroundService.kt:53`) і керується intent-діями (`onStartCommand`, `:91`):
`START_OUTGOING`, `START_INCOMING`, `ACCEPT`, `DECLINE`, `END`, плюс перемикачі
mute/speaker/camera. Основне:

- Вихідний (`handleStartOutgoing`, `:111`): засіває `ActiveCallHolder`, стартує foreground із
  типом сервісу microphone (і camera, якщо відео), надсилає `RINGING`-`CallSignal`, здобуває
  аудіофокус, приєднується до каналу Agora, спостерігає за віддаленим статусом і ack-ом дзвінка від
  callee, і озброює таймаут "requesting".
- Вхідний (`handleStartIncoming`, `:156`): якщо вже на іншому дзвінку - відхиляє новий; інакше
  постить повноекранне сповіщення дзвінка (або тихе, якщо застосунок на передньому плані), ack-ає
  дзвінок, спостерігає віддалений статус, озброює 60-секундний таймаут дзвінка.
- Прийняти (`handleAccept`, `:203`): переводить в active, підвищує тип foreground-сервісу до
  включення mic/camera, пише статус `ACTIVE`, приєднується до каналу, запускає таймер дзвінка й
  відстеження присутності.
- Завершити (`terminate`, `:249`): записує завершений дзвінок, якщо він був активним, пише
  термінальний статус із таймаутом 2 с, очищає вузли сигналізації, потім `stopSelfClean`.

Таймаути й перепідключення - явні job: дзвінок (60 с), requesting (30 с), вихідний/без відповіді
(65 с), і 20-секундне вікно перепідключення (`startReconnectTimeout`, `:497`), що завершує дзвінок,
якщо він не повністю з'єднаний (`isFullyConnected` = active + remote present + Agora CONNECTED).
Присутність пірінга відстежується поверх сигналізації (`startPresence`, `:581`), тож якщо застосунок
пірінга помирає, дзвінок завершується після вікна перепідключення.

Дисципліна прибирання: `stopSelfClean` (`:531`) скасовує кожен job, зупиняє ringback-тон, залишає
канал Agora, звільняє аудіофокус, очищає `ActiveCallHolder`, позначає Telecom-з'єднання
роз'єднаним і зупиняє foreground-сервіс. `onDestroy` (`:905`) додатково скасовує весь scope через
`scope.coroutineContext[Job]?.cancel()`, тож жоден job таймауту не може пережити сервіс. Це той
самий приклад керування scope сервісу, згаданий у [[Async and Reactivity_UA|Асинхронність і реактивність]].

## UI віддзеркалює стан

`CallViewModel` (`presentation/viewmodel/CallViewModel.kt:48`) - звичайна `AndroidViewModel` із
`MutableStateFlow<CallUiState>`. `observeActiveCall` (`:113`) збирає `ActiveCallHolder.state` через
`collectLatest` і копіює кожне поле в `CallUiState`; коли holder стає null, він програє короткий
стан "дзвінок завершено" й маршрутизує вихід через `Channel<CallExit>` (`receiveAsFlow()`, `:62`) -
Pop, або OpenChat для прийнятого вхідного дзвінка. Дії користувача - тонкі `startService`-intent'и
до foreground-сервісу (`acceptCall`, `endCall`, `toggleMute`, `switchCamera`, ...). Відеоповерхні
прив'язуються напряму до Agora через `bindLocalVideo`/`bindRemoteVideo` (`:228`). Старт вихідного
дзвінка (`startOutgoing`, `:183`) будує `TelecomCallMeta`, стартує foreground-сервіс і розміщує
дзвінок через Telecom.

`observePartnerAvatar` (`:99`) скасовує попередній job спостереження за аватаром перед повторним
підписуванням на новий id пірінга, тож перемикання відстежуваного пірінга не протікає застарілий
збирач `observeUser`.

## Панель активного дзвінка: свайп, бульбашка, повернення

Поки дзвінок активний і не на повноекранному екрані дзвінка, докована панель показується вгорі
(змонтована в корені навігації, `presentation/navigation/navigation.kt:424`, тож зберігається між
екранами). `ActiveCallBar` (`presentation/components/call/ActiveCallBar.kt:88`) читає
`ActiveCallHolder.state` та `CallBarPresenter.mode` і опрацьовує вертикальний drag
(`ActiveCallBar.kt:110`):

```kotlin
detectVerticalDragGestures(
    onVerticalDrag = { change, delta -> change.consume(); scope.launch { dragY.snapTo(dragY.value + delta) } },
    onDragEnd = {
        when {
            offset <= -hideThresholdPx -> scope.launch { dragY.animateTo(-hideTargetPx, tween(160)); CallBarPresenter.hide() }
            offset >= bubbleThresholdPx -> { CallBarPresenter.bubbleOffset = null; CallBarPresenter.minimizeToBubble() }
            else -> scope.launch { dragY.animateTo(0f, tween(180)) }
        }
    },
    onDragCancel = { scope.launch { dragY.animateTo(0f, tween(180)) } },
)
```

- Свайп вгору за поріг - анімувати геть і `CallBarPresenter.hide()` (режим HIDDEN).
- Свайп вниз за поріг - `minimizeToBubble()` (режим BUBBLE), відрендерене як перетягувана
  `CallBubble`, що плаває над контентом.
- Інакше - `animateTo(0f)` повертає панель на її доковану позицію.

`CallBarPresenter` (`presentation/components/call/CallBarPresenter.kt:10`) - глобальний `object`, що
тримає `MutableStateFlow<CallBarMode>` (BAR / HIDDEN / BUBBLE) плюс останній зсув бульбашки; він
скидається до BAR, коли дзвінок завершується (`navigation.kt:148`). Панель показує живу
хвильову діаграму й таймер дзвінка при з'єднанні, або напис статусу
(`Requesting...`/`Ringing...`/`Connecting...`) інакше (`callBarStatus`, `ActiveCallBar.kt:72`), із
вбудованими кнопками mute і завершення, що постять дії сервісу.

`IncomingCallBar` показується для вхідного дзвінка, що дзвонить, а `CallBarOverlay` хостить плаваючу
бульбашку; усі три зв'язані в `MainDisplay` (`navigation.kt:412`-`:437`). Див.
[[Navigation_UA|Навігація]].

## PiP і Telecom

Пакет `telecom/` реєструє self-managed phone account (`TelecomCallManager`, реєструється в
`MessengerApplication.onCreate`), тож дзвінки інтегруються з системою (`MessengerConnectionService`,
`MessengerConnection`), що й вмикає системний PiP/поведінку для поверхні дзвінка (`PipCallSurface`).
Вхідні FCM-навантаження дзвінка додають вхідне Telecom-з'єднання (див. [[Notifications_UA|Сповіщення]]).

## Історія дзвінків

Завершені/пропущені/недосяжні дзвінки записуються як повідомлення чату типу дзвінка
(`MissedCallRecorder`, викликається із сервісу, наприклад `recordEndedCall`,
`CallForegroundService.kt:400`). Вкладка дзвінків виводить свій журнал з тих повідомлень:
`CallHistoryRepository.getCallHistory` (`data/repository/CallHistoryRepository.kt:24`) запитує
`ObxMessage.type.oneOf(CALL_TYPES)` (індексовано - `ObxEntities.kt:26`), сортує за timestamp
спадно, відфільтровує видалені рядки, батчить потрібні розмови одним запитом `uid.oneOf(...)` і
маппить кожен рядок у `CallHistoryEntry` (пірінг, напрям, тривалість, прапорець відео). Повернення
`Query.asFlow().map` тримає історію живою. `CallsViewModel` віддає її `CallsScreen`; бульбашки
повідомлень-дзвінків у чаті рендеряться через `CallMessageCards`.

## Чому такий дизайн

- Foreground-сервіс - єдиний компонент, що надійно переживає перехід застосунку у фон/зметання під
  час дзвінка, тож він володіє машиною станів; UI не може стати джерелом істини для того, що має
  його пережити.
- Єдиний глобальний `StateFlow` як межа один-писар/багато-читачів прибирає цілий клас багів "панель
  каже з'єднано, а екран каже дзвонить".
- Явні, іменовані job таймаутів плюс вікно перепідключення роблять численні крайові випадки дзвінка
  (без відповіді, застосунок пірінга вбито, обрив мережі) детермінованими, а не емерджентними.
- Показ bar/bubble/hidden відв'язаний у `CallBarPresenter`, тож опрацювання жестів і рендеринг
  лишаються простими, а режим зберігається між навігаціями.

## Пов'язане

[[Async and Reactivity_UA|Асинхронність і реактивність]] - [[Navigation_UA|Навігація]] -
[[Notifications_UA|Сповіщення]] - [[Presence and Typing_UA|Присутність і набір тексту]] -
[[Data Layer_UA|Шар даних]] - [[Architecture Overview_UA|Огляд архітектури]]
