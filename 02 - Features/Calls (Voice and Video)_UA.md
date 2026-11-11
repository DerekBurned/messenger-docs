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

- `CallForegroundService` (`data/remote/call/CallForegroundService.kt:50`) - машина станів і
  власник життєвого циклу. `@AndroidEntryPoint`.
- `ActiveCallHolder` (`data/remote/call/ActiveCallHolder.kt:9`) - єдиний глобальний
  `StateFlow<ActiveCall?>`.
- `AgoraCallService` (`data/remote/call/AgoraCallService.kt:21`) - RTC-обгортка `ICallService`.
- `FirebaseCallSignalingService` (`ICallSignalingService`) - сигналізація дзвінків поверх
  Realtime DB.
- `AgoraTokenProvider` (`data/remote/call/AgoraTokenProvider.kt:10`; викликає Cloud Function
  `getRtcToken`), `CallUids`, `MissedCallRecorder`, `IncomingCallCoordinator`.
- Шлюзи feature flag: `CallMediaGate` (`data/remote/call/CallMediaGate.kt:12`) і `IncomingCallGate`
  (`data/remote/call/IncomingCallGate.kt:15`), див. "Feature flag" нижче.
- Інтеграція з Telecom: `data/remote/call/telecom/` (`TelecomCallManager`,
  `MessengerConnectionService`, `MessengerConnection`) для системного phone-account/PiP.
- Представлення: `CallViewModel` (`presentation/screens/call/CallViewModel.kt:40`), `CallScreen`,
  `CallsViewModel`/`CallsScreen` і компоненти в `presentation/components/call/`
  (`ActiveCallBar`, `CallBarPresenter`, `CallBubble`, `CallControls`, `IncomingCallBar`,
  `PipCallSurface`).

## Agora RTC

Залежність `io.agora.rtc:full-sdk:4.6.3`, оголошена напряму в `app/build.gradle.kts:204`
(не в каталозі версій). Повний SDK, бо відео потребує нативних відеомодулів; App ID резолвиться з
`local.properties` -> Gradle-property -> порожній рядок у `BuildConfig.AGORA_APP_ID`
(`app/build.gradle.kts:20`-`:24`, `:53`). У репозиторії App ID не зберігається. Без нього збірка все одно
проходить і застосунок запускається: `AgoraCallService` пише помилку в лог і залишає двигун `null`,
тож дзвінки просто не з'єднуються.

`AgoraCallService` створює один `RtcEngine` у своєму `init` (`AgoraCallService.kt:106`) і адаптує
колбеки `IRtcEngineEventHandler` у доменний `CallEventListener` (`AgoraCallService.kt:30`):
`onUserJoined -> onRemoteUserJoined`, `onUserOffline` розділяється на навмисний вихід
(`onRemoteUserLeft`) проти обриву мережі (`onRemoteConnectionLost`),
`onConnectionStateChanged -> onConnectionStateChanged`, `onTokenPrivilegeWillExpire ->` оновлення
токена. `joinChannel` (`:130`) встановлює `ChannelMediaOptions` (профіль комунікації, роль
broadcaster, публікація мікрофона, публікація камери коли відео увімкнено). Відео вмикається ліниво
(`ensureVideoModule`, `:238`), тож дзвінки лише-аудіо ніколи не розкручують відеоконвеєр.

## Машина станів живе у foreground-сервісі

`ActiveCallHolder` - це `object` із `MutableStateFlow<ActiveCall?>` (`ActiveCallHolder.kt:37`). Його
`set`/`update`/`clear` - `internal` (`:40`-`:48`) - пише лише сервіс; усе інше читає `state` або `snapshot()`. Це
ключове архітектурне рішення: сервіс - єдиний писар стану дзвінка, а UI - stateless-дзеркало, тож
екран, сповіщення й панель ніяк не можуть розійтися.

`CallForegroundService` виконує все на `CoroutineScope(SupervisorJob() + Dispatchers.IO)`
(`CallForegroundService.kt:58`) і керується intent-діями (`onStartCommand`, `:109`):
`START_OUTGOING`, `START_INCOMING`, `ACCEPT`, `DECLINE`, `END`, плюс перемикачі
mute/speaker/camera. Основне:

- Вихідний (`handleStartOutgoing`, `:129`): питає `CallMediaGate`, які медіа дозволені цьому
  дзвінку, засіває `ActiveCallHolder`, стартує foreground із типом сервісу microphone (і camera,
  якщо відео), далі `signalThenJoin` (`:1033`): спершу пишеться `RINGING`-`CallSignal`, а аудіофокус
  і локальне налаштування відео/динаміка виконуються, поки запис триває, і лише після завершення
  запису викликач отримує RTC-токен і приєднується до каналу Agora (`fetchTokenAndJoin`, `:726`).
  Потім спостерігає за віддаленим статусом і ack-ом дзвінка від callee і озброює таймаут
  "requesting".
- Вхідний (`handleStartIncoming`, `:177`): якщо вже на іншому дзвінку - відхиляє новий; інакше
  постить повноекранне сповіщення дзвінка (або тихе, якщо застосунок на передньому плані), ack-ає
  дзвінок, спостерігає віддалений статус, озброює 60-секундний таймаут дзвінка.
- Прийняти (`handleAccept`, `:225`): повторно перевіряє медіа через `CallMediaGate` (відеодзвінок, чий прапорець тим часом вимкнено, деградує до аудіо), переводить в active, підвищує тип foreground-сервісу до
  включення mic/camera, пише статус `ACTIVE`, приєднується до каналу, запускає таймер дзвінка й
  відстеження присутності.
- Завершити (`terminate`, `:271`): записує завершений дзвінок, якщо він був активним, пише
  термінальний статус із таймаутом 2 с, очищає вузли сигналізації, потім `stopSelfClean`.

Таймаути й перепідключення - явні job: дзвінок (60 с), requesting (30 с), вихідний/без відповіді
(65 с), і 20-секундне вікно перепідключення (`startReconnectTimeout`, `:517`), що завершує дзвінок,
якщо він не повністю з'єднаний (`isFullyConnected` = active + remote present + Agora CONNECTED).
Присутність пірінга відстежується поверх сигналізації (`startPresence`, `:598`), тож якщо застосунок
пірінга помирає, дзвінок завершується після вікна перепідключення.

Дисципліна прибирання: `stopSelfClean` (`:551`) скасовує кожен job, зупиняє ringback-тон, залишає
канал Agora, звільняє аудіофокус, очищає `ActiveCallHolder`, роз'єднує Telecom-з'єднання через
`ActiveConnectionHolder.disconnect` (див. [[#PiP і Telecom]]) і зупиняє foreground-сервіс.
`onDestroy` (`:941`) - це запобіжник: він роз'єднує *усі* залишкові Telecom-з'єднання
(`disconnectAll`), очищає `ActiveCallHolder` і скасовує весь scope через
`scope.coroutineContext[Job]?.cancel()`, тож жоден job таймауту - і жоден застряглий системний
"активний дзвінок" - не може пережити сервіс. Це той самий приклад керування scope сервісу,
згаданий у [[Async and Reactivity_UA|Асинхронність і реактивність]].

## UI віддзеркалює стан

`CallViewModel` (`presentation/screens/call/CallViewModel.kt:40`) - звичайна `AndroidViewModel` із
`MutableStateFlow<CallUiState>`. `observeActiveCall` (`:106`) збирає `ActiveCallHolder.state` через
`collectLatest` і копіює кожне поле в `CallUiState`; коли holder стає null, він програє короткий
стан "дзвінок завершено" й маршрутизує вихід через `Channel<CallExit>` (`receiveAsFlow()`, `:55`) -
Pop, або OpenChat для прийнятого вхідного дзвінка. Дії користувача - тонкі `startService`-intent'и
до foreground-сервісу (`acceptCall`, `endCall`, `toggleMute`, `switchCamera`, ...). Відеоповерхні
прив'язуються напряму до Agora через `bindLocalVideo`/`bindRemoteVideo` (`AgoraCallService.kt:198`, `:204`). Старт
вихідного дзвінка (`startOutgoing`, `CallViewModel.kt:181`) будує `TelecomCallMeta`, стартує foreground-сервіс і розміщує
дзвінок через Telecom.

`observePartnerAvatar` (`CallViewModel.kt:92`) скасовує попередній job спостереження за аватаром перед повторним
підписуванням на новий id пірінга, тож перемикання відстежуваного пірінга не протікає застарілий
збирач `observeUser`.

## Панель активного дзвінка: свайп, бульбашка, повернення

Поки дзвінок активний і не на повноекранному екрані дзвінка, докована панель показується вгорі
(змонтована в корені навігації, `presentation/navigation/graph/MainDisplay.kt:625`, тож
зберігається між екранами). `ActiveCallBar` (`presentation/components/call/ActiveCallBar.kt:148`)
читає `ActiveCallHolder.state` та `CallBarPresenter.mode` і опрацьовує вертикальний drag
(`detectVerticalDragGestures`, `ActiveCallBar.kt:190`). Після відпускання зсув вирішує:

- Свайп вгору за поріг - анімувати геть і `CallBarPresenter.hide()` (режим HIDDEN).
- Свайп вниз за поріг, і лише коли `CallBarPresenter.floatingEnabled` - `minimizeToBubble()` (режим
  BUBBLE), відрендерене як перетягувана `CallBubble`, що плаває над контентом і з'являється там, де
  була панель (`bubbleOffset`).
- Інакше - `animateTo(0f)` повертає панель на її доковану позицію.

`CallBarPresenter` (`presentation/components/call/CallBarPresenter.kt:9`) - глобальний `object`, що
тримає `MutableStateFlow<CallBarMode>` (`BAR`, `HIDDEN`, `BUBBLE`, `VIDEO`; enum у
`presentation/components/call/model/CallBarMode.kt:3`) плюс останній зсув бульбашки та запам'ятований
режим відновлення. `VIDEO` - плаваюча відеоплитка: поки дзвінок має відео, а панель - бульбашка,
`MainDisplay` підвищує її до плитки (`showVideoTile`) і згортає назад, коли відео припиняється
(`MainDisplay.kt:202`-`:213`); закриття плитки запам'ятовується (`videoDismissed`) до відновлення.
Презентер скидається до BAR, коли дзвінок завершується (`MainDisplay.kt:194`). Панель показує живу
хвильову діаграму й таймер дзвінка при з'єднанні, або напис статусу
(`Requesting...`/`Ringing...`/`Connecting...`) інакше (`callBarStatus`, `ActiveCallBar.kt:93`), із
вбудованими кнопками mute і завершення, що постять дії сервісу.

`IncomingCallBar` (`MainDisplay.kt:612`) показується для вхідного дзвінка, що дзвонить, а
`CallBarOverlay` (`MainDisplay.kt:679`) хостить плаваючу бульбашку й відеоплитку. Див.
[[Navigation_UA|Навігація]].

## PiP і Telecom

Пакет `telecom/` реєструє self-managed phone account (`TelecomCallManager`, реєструється в
`MessengerApplication.onCreate`), тож дзвінки інтегруються з системою (`MessengerConnectionService`,
`MessengerConnection`), що й вмикає системний PiP/поведінку для поверхні дзвінка (`PipCallSurface`).
Вхідні FCM-навантаження дзвінка додають вхідне Telecom-з'єднання (див. [[Notifications_UA|Сповіщення]]).

Саме системне з'єднання змушує Android показувати дзвінок як "активний", тож надійне його
прибирання має значення. Telecom створює `Connection` *асинхронно* (`onCreateOutgoingConnection` /
`onCreateIncomingConnection` спрацьовують після `placeCall`/`addNewIncomingCall`), що може
випередити швидке завершення й лишити фантомний активний дзвінок. Тому `ActiveConnectionHolder`
(`data/remote/call/telecom/ActiveConnectionHolder.kt`) тепер ідемпотентний і стійкий до гонок:

- `disconnect(callId, cause)` (`:28`) роз'єднує з'єднання, якщо воно зареєстроване, інакше
  записує callId в обмежену мапу `pendingDisconnects`.
- `put(callId, connection)` (`:12`) звіряється з цією мапою, тож з'єднання, що приходить *після*
  прибирання, роз'єднує себе одразу, а не зависає.
- `disconnectAll(cause)` (`:38`) скидає кожне залишкове з'єднання - запобіжник у `onDestroy`.
- Гілка "зайнято" в `handleStartIncoming` (`CallForegroundService.kt:187`) роз'єднує з'єднання
  відхиленого паралельного дзвінка, а `stopSelfClean` веде прибирання через `disconnect`, а не
  через одноразовий пошук.

`MessengerConnection.markDisconnected` (`MessengerConnection.kt:60`) робить власне
`setDisconnected` + `destroy`.

## Екран під час дзвінка

До 0.1.13.1 дисплей гас посеред дзвінка, бо ніщо не тримало його увімкненим. Тепер:

- `KeepScreenOn(enabled)` (`presentation/components/common/KeepScreenOn.kt:9`) - це
  `DisposableEffect`, що додає `FLAG_KEEP_SCREEN_ON` вікну-хосту, поки `enabled`, і знімає його при
  dispose. `CallScreenContent` використовує його з `enabled = !state.callEnded` (`CallScreenContent.kt:26`); композер голосового
  запису перевикористовує його під час запису (див. [[Voice Messages_UA|Голосові повідомлення]]).
- `IncomingCallActivity` ставить той самий прапорець поруч із `setShowWhenLocked(true)` при
  створенні (`presentation/IncomingCallActivity.kt:56`, `:58`).
- Затемнення від датчика наближення: `CallForegroundService` збирає `ActiveCallHolder.state` в
  `onCreate` (`CallForegroundService.kt:95`-`:105`) і хоче блокування, поки дзвінок активний, не
  завершений, з вимкненою локальною камерою й вимкненим динаміком - тобто аудіодзвінок через
  розмовний динамік. `acquireProximityLock` (`:731`) бере `PROXIMITY_SCREEN_OFF_WAKE_LOCK` (якщо
  пристрій підтримує), тож екран гасне біля вуха; перемикання на відео чи гучний зв'язок або
  завершення дзвінка звільняє його з `RELEASE_FLAG_WAIT_FOR_NO_PROXIMITY` (`:748`), а
  `stopSelfClean` теж його звільняє. Блокування береться з таймаутом 10 хвилин і повторно береться
  лише при зміні стану дзвінка. Потрібен дозвіл `WAKE_LOCK`.

`MainActivity` оголошено з `android:screenOrientation="fullSensor"`, тож застосунок вільно
обертається, включно з екраном дзвінка.

## Історія дзвінків

Завершені/пропущені/недосяжні дзвінки записуються як повідомлення чату типу дзвінка
(`MissedCallRecorder`, викликається із сервісу, наприклад `recordEndedCall`,
`CallForegroundService.kt:420`). Вкладка дзвінків виводить свій журнал з тих повідомлень:
`CallHistoryRepository.getCallHistory` (`data/repository/CallHistoryRepository.kt:25`) запитує
`ObxMessage.type.oneOf(CALL_TYPES)` (індексовано - `ObxMessage.kt:24`), сортує за timestamp
спадно, відфільтровує видалені рядки, батчить потрібні розмови одним запитом `uid.oneOf(...)` і
маппить кожен рядок у `CallHistoryEntry` (пірінг, напрям, тривалість, прапорець відео). Повернення
`Query.asFlow().map` тримає історію живою. `CallsViewModel` віддає її `CallsScreen`; бульбашки
повідомлень-дзвінків у чаті рендеряться через `CallMessageCards`.

## RTC-токени видаються лише викликачу й адресату

PR #64 (злито 2026-11-07, виправляє #33) закрив діру в `getRtcToken` (`functions/src/index.ts:259`):
раніше він перевіряв лише, що запитувач ввійшов і що назва каналу схожа на `call-<uuid>`, тож будь-хто,
хто дізнався назву каналу, міг приєднатися до дзвінка. Тепер:

- Викликач пише сигнал дзвінка і запис `callChannels/{channel}` лише для створення (`callerId`,
  `calleeId`, `createdAt`) одним багатошляховим оновленням (`callSignalUpdate`,
  `data/remote/call/FirebaseCallSignalingService.kt:159`, використовується `sendCallSignal`,
  `:28`-`:30`). `database.rules.json` дозволяє створити цей запис лише викликачу, вимагає ключ
  `call-<36 hex/дефіс-символів>` і забороняє пізніше змінювати учасників.
- `getRtcToken` відхиляє неавтентифікованих запитувачів, некоректні назви каналів
  (`isCallChannelName`, `functions/src/callAccess.ts:5`-`:7`) і будь-який uid, що не є ні `callerId`,
  ні `calleeId` запису (`isCallParticipant`, `callAccess.ts:9`-`:12`), з `permission-denied`
  (`functions/src/index.ts:259`-`:274`); лише потім будує токен за обліковими даними Agora, що
  зберігаються як секрети Cloud Functions. Нічого з цих секретів у цьому репозиторії немає.
- Оскільки запис має існувати до того, як запит токена вдасться, викликач отримує токен лише після
  завершення запису сигналу (`signalThenJoin`, див. "Вихідний" вище); адресат питає після прийняття,
  коли сигнал уже існує. Якщо токен отримати не вдається, `AgoraTokenProvider` повертає `null`, і клієнт
  приєднується без нього (`AgoraTokenProvider.kt:14`-`:26`), що не з'єднається в проєкті, який вимагає
  токени.

## Feature flag

Дзвінками керують `calls` та його дочірні прапорці (`FeatureFlag.CALLS`, `CALLS_VIDEO`,
`CALLS_FLIP_CAMERA`, `CALLS_SPEAKER`, `CALLS_PIP`, `CALLS_FLOATING_BUBBLE`; вимкнення `calls`
вимикає всіх дочірніх, `FeatureFlagState.isEnabled`). Вони застосовуються у чотирьох шарах (PR #68,
злито 2026-11-07); див. [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]]:

- Сервіс/шлюз: `CallMediaGate` (`data/remote/call/CallMediaGate.kt:16`-`:31`) визначає медіа дзвінка:
  відео лише коли увімкнено `CALLS_VIDEO`, камера лише за наявності відео й дозволу, динамік на
  старті лише за відео й `CALLS_SPEAKER`. `CallForegroundService` питає його при старті вихідного
  дзвінка (`:133`), прийнятті (`:229`), перемиканні динаміка (`:313`), камери (`:335`), зміні камери
  (`:350`) і показі віддаленого відео (`:702`). `IncomingCallGate.admit` (`IncomingCallGate.kt:22`)
  чекає свіжої конфігурації (`awaitFresh`) і, коли `CALLS` вимкнено, пише `DECLINED` у сигнал і
  відмовляє; його викликають і для RTDB-сигналів (`IncomingCallCoordinator.kt:56`), і для FCM-payload
  (`MyFirebaseMessagingService.kt:151`-`:152`), а його набір `startedCallIds` також дедуплікує той
  самий дзвінок, що надійшов обома шляхами. Дзвінок, що вже триває, не завершується.
- ViewModel: `CallViewModel.outgoingAllowed` (`CallViewModel.kt:177`) вимагає `CALLS` і, для відео,
  `CALLS_VIDEO`; інакше `startOutgoing` випромінює `CallExit.Unavailable` (`:181`-`:186`).
  `toggleSpeaker`, `toggleCamera` і `switchCamera` відмовляють за своїми прапорцями (`:229`, `:235`,
  `:240`), хоча вимкнення функції все одно дозволене.
- UI: кнопки дзвінків верхньої панелі чату (`ChatTopBar.kt:131`, `:146`), кнопки профілю
  (`ChatUserProfileComponents.kt:98`, `:108`), кнопки перетелефонувати в картках дзвінків та історії
  (`CallMessageCards.kt:63`, `CallHistoryRow.kt:45`) і елементи керування в дзвінку
  (`CallScreen.kt:58`, `ActiveCallControls.kt:28`, `:38`, `:48`); елемент лишається видимим, коли його
  стан уже увімкнений, щоб його можна було вимкнути. `CALLS_FLOATING_BUBBLE` застосовується до
  `CallBarPresenter.setFloatingEnabled` з `MainDisplay.kt:154`-`:155`.
- Activity: `MainActivity.updateCallPipParams` поєднує запит PiP з `CALLS_PIP`
  (`MainActivity.kt:186`-`:187`), а кнопки дій PiP для камери й динаміка йдуть за `CALLS_VIDEO` /
  `CALLS_SPEAKER` (`:216`, `:224`).

## Чому такий дизайн

- Foreground-сервіс - єдиний компонент, що надійно переживає перехід застосунку у фон/зметання під
  час дзвінка, тож він володіє машиною станів; UI не може стати джерелом істини для того, що має
  його пережити.
- Єдиний глобальний `StateFlow` як межа один-писар/багато-читачів прибирає цілий клас багів "панель
  каже з'єднано, а екран каже дзвонить".
- Явні, іменовані job таймаутів плюс вікно перепідключення роблять численні крайові випадки дзвінка
  (без відповіді, застосунок пірінга вбито, обрив мережі) детермінованими, а не емерджентними.
- Гейтинг у сервісі та шлюзах, а не лише кнопках, означає, що push, застарілий екран чи дія PiP не
  можуть запустити дзвінок або медіа, які вимкнув прапорець.
- Видача RTC-токенів лише учасникам, записаним викликачем, робить назву каналу можливістю, якої
  вже недостатньо самої по собі.
- Показ bar/bubble/hidden відв'язаний у `CallBarPresenter`, тож опрацювання жестів і рендеринг
  лишаються простими, а режим зберігається між навігаціями.

## Пов'язане

[[Async and Reactivity_UA|Асинхронність і реактивність]] - [[Navigation_UA|Навігація]] -
[[Notifications_UA|Сповіщення]] - [[Presence and Typing_UA|Присутність і набір тексту]] -
[[Feature Flags and Remote Config_UA|Feature flags і Remote Config]] - [[Data Layer_UA|Шар даних]] - [[Architecture Overview_UA|Огляд архітектури]]
