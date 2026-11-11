---
tags:
  - _UA
---
# Feature flag-и та Remote Config

Застосунок може вимикати окремі функції з консолі Firebase без випуску нової збірки. Feature flag -
це іменований булів прапорець; Firebase Remote Config доставляє значення, невеликий доменний тип його
обчислює, а ту саму відповідь читають у трьох місцях: у дереві Compose (щоб сховати точки входу), у
ViewModel (щоб відмовити в діях) і в коді дзвінків та сповіщень, що працює без жодного UI (щоб
відмовити в роботі, яку запустив push). Поруч стоїть Crashlytics, що записує, які прапорці були
вимкнені, коли стався збій. Ця нотатка розповідає, як прапорець оголошується, як його значення
обчислюється й оновлюється, і чому сховати кнопку - ніколи не єдина перевірка.

Додано 2026-10-18 (коміт `020711a`, а Crashlytics - у `65115d0`), шаблон для консолі з'явився
2026-10-21 (PR #22), а застосування нижче за UI - 2026-11-05 - 2026-11-07 (PR #68).

Посилання `path:line` - відносно `app/src/main/java/com/example/messenger/`, якщо шлях не починається
з імені кореня репозиторію (`remoteconfig.template.json`, `app/build.gradle.kts`, `functions/...`,
`database.rules.json`).

## Що

- Remote Config: `com.google.firebase:firebase-config`, версію задає Firebase BOM
  (`gradle/libs.versions.toml:92`, підключено в `app/build.gradle.kts:159`). Див.
  [[Tech Stack and Versions_UA|Технологічний стек і версії]].
- Crashlytics: `firebase-crashlytics` (`gradle/libs.versions.toml:91`) і його Gradle-плагін
  (`app/build.gradle.kts:9`).
- Каталог прапорців: `enum class FeatureFlag` у `domain/featureflag/FeatureFlag.kt:3`, 62 записи, кожен
  має `key` Remote Config і необов'язковий `parent`.
- Тримач значень: інтерфейс `FeatureFlags` (`domain/featureflag/FeatureFlags.kt:5`), який реалізує
  `RemoteConfigFeatureFlags` (`data/remote/firebase/RemoteConfigFeatureFlags.kt:27`) і який
  прив'язано в `di/FeatureFlagModule.kt:16`.
- Шаблон для консолі: `remoteconfig.template.json` у корені репозиторію, що перелічує кожен прапорець,
  щоб консоль показувала їх як іменовані згруповані параметри.

## Як ми це використовуємо

### Оголошення прапорця

Прапорець - це один рядок в enum: ключ у snake_case (ім'я параметра Remote Config) і, якщо функція є
частиною більшої, її батько. Форма - `CALLS_VIDEO("calls_video", CALLS)` (`FeatureFlag.kt:5`). Зв'язок
із батьком робить один перемикач каскадним: вимкнення `calls` вимикає кожен прапорець `calls_*`,
вимкнення `clip_playback` вимикає перемотування, міні-плеєр і швидкість відтворення, а
`clip_playback_speed` сам є дитиною `clip_mini_player`, тож ланцюг двоступеневий
(`FeatureFlag.kt:15` - `:18`). `FeatureFlagState.isEnabled` піднімається вгору по ланцюгу батьків і
відповідає false, щойно будь-який предок вимкнений (`domain/featureflag/FeatureFlagState.kt:5`).

В enum є й три допоміжні функції для описаного нижче списку часу збірки: `fromKey` (`:79`),
`parseKeys` (`:84`) і `unknownKeys` (`:87`). Помилка в цьому списку не валить застосунок; `start()`
логує попередження для кожного невідомого ключа
(`data/remote/firebase/RemoteConfigFeatureFlags.kt:39`).

Важливо й те, чого прапорця немає взагалі: в enum нічого немає для входу, виходу, видалення акаунта,
надсилання текстового повідомлення, прийняття, відхилення, вимкнення мікрофона чи завершення дзвінка,
ані для наскрізного шифрування. Коміт, що запровадив прапорці, каже те саме ("not switchable"). Ці
шляхи не можна вимкнути віддалено.

### Значення за замовчуванням до та після отримання

Обчислення - одне правило в `FeatureFlagState.resolve` (`FeatureFlagState.kt:21`): для кожного
прапорця береться значення Remote Config, якщо воно є, інакше "увімкнено, якщо збірка не каже
вимкнено за замовчуванням" (`remoteValue(flag.key) ?: (flag !in offByDefault)`, `:26`).

- **Поки нічого не надійшло**, стан - це `FeatureFlagState.defaults(offByDefault)`
  (`RemoteConfigFeatureFlags.kt:33`). Набір "вимкнено за замовчуванням" береться з
  `BuildConfig.FEATURES_OFF_BY_DEFAULT` (`:32`), який `app/build.gradle.kts` заповнює для кожного
  типу збірки: порожній список для `debug` (а отже й для `e2e`, що його копіює) на `:62`, а для
  `release` - список `featuresOffInRelease` (`:66`, сам список `:29` - `:39`). На 0.2.2.4 цей список
  містить дев'ять недороблених функцій: `profile_edit_change_photo`, `accounts_switch`,
  `partner_profile_mute`, `partner_profile_block`, `settings_privacy_controls`,
  `settings_security_placeholders`, `settings_notification_controls`, `settings_data_storage` та
  `settings_language`. (README репозиторію каже "десять"; у build-файлі дев'ять, бо `settings_security`
  прибрали зі списку, щоб рядок Безпеки лишався доступним у release-збірках; це покриває тест
  `app/src/test/.../data/remote/firebase/RemoteConfigFeatureFlagsTest.kt:168`.) Тож у debug-збірці
  кожен перемикач увімкнений, у release-збірці дев'ять вимкнені, і обидва варіанти коректні, поки
  телефон офлайн або Remote Config ще жодного разу не відповів.
- **Після отримання** значення з консолі перемагає в обох напрямках: воно може вимкнути
  увімкнений-за-замовчуванням прапорець, а публікація `true` вмикає вимкнений у release прапорець
  (група Settings у шаблоні каже "All are off in release builds unless you publish true"). Гачок -
  `remoteValue` (`:85`): він повертає значення лише коли Remote Config повідомляє джерело
  `VALUE_SOURCE_REMOTE` (`:87`). Параметр, залишений на значенні за замовчуванням у застосунку, або
  не булів ігнорується, і діє значення збірки (тести: `an in-app default value is ignored`,
  `a console value that is not a boolean keeps the build default`).

### Життєвий цикл отримання й активації

`MessengerApplication.onCreate` першим викликає `featureFlags.start()`, ще до налаштування зображень,
синхронізації чи push (`MessengerApplication.kt:71`). `start()` ідемпотентний (захист через
`AtomicBoolean`, `:35` і `:38`) і робить чотири речі (`RemoteConfigFeatureFlags.kt:37` - `:58`):

1. Задає мінімальний інтервал отримання: нуль у debug-збірках, 900 секунд в інших (`:44`, `:94`).
2. `ensureInitialized()` і, по завершенню, `publish()`, тож значення, уже збережені на пристрої з
   попереднього запуску, використовуються одразу (`:47`).
3. `fetchAndActivate()` і `publish()` по завершенню (`:48`).
4. Слухач реального часу `addOnConfigUpdateListener`: коли консоль публікує зміну, слухач викликає
   `activate()` і `publish()` (`:49` - `:52`); помилка слухача лише логується.

`publish()` (`:79`) перераховує весь `FeatureFlagState` із поточних значень Remote Config,
записує його в `StateFlow` і записує нижче описаний користувацький ключ Crashlytics.

Код, що не може діяти на застарілому значенні, використовує `awaitFresh()` (`:60`): чекати до 1
секунди на ініціалізацію, потім до 1,5 секунди на `fetch(300)` (прийняти значення віком до п'яти
хвилин) та `activate()`, потім `publish()`. Кожен `Task` очікується через `awaitQuietly`, що логує
збій, але перекидає скасування далі (`:69`), а таймаути означають, що відсутня мережа затримує
викликача щонайбільше на мить, після чого береться те, що вже опубліковано. Цей виклик використовують
шлюз вхідного дзвінка, push-обробник і приймач дій сповіщення, нижче.

### Як UI читає прапорці

`MainActivity` збирає `featureFlags.state` через `collectAsStateWithLifecycle` і надає його як
`LocalFeatureFlags` навколо всього застосунку (`presentation/MainActivity.kt:99` - `:103`);
`BubbleActivity`, що містить чат у бульбашці сповіщення, робить те саме
(`presentation/BubbleActivity.kt:53`, `:58`). `presentation/featureflag/LocalFeatureFlags.kt` оголошує
local із `FeatureFlagState.defaults()` як запасним значенням (`:9`) і читач `featureEnabled(flag)`
(`:13`) - `@ReadOnlyComposable`, що питає поточний стан. Його викликають близько 33 composable-файлів
(72 місця виклику на 0.2.2.4), наприклад кнопки дзвінка у верхній панелі чату
(`presentation/components/chat/ChatTopBar.kt:131`), жест свайпу для відповіді
(`presentation/components/chat/ChatSwipeGestures.kt:64`) і пошук у панелі вкладок
(`presentation/navigation/graph/MainDisplay.kt:223`). Оскільки стан - це `StateFlow`, який замінює
`publish()`, змінений у консолі прапорець рекомпонує відкритий екран.

Приховування - це зручність, а не гарантія, тому існують наступні два розділи.

### Як ViewModel застосовують прапорці

Охоронець - крихітне розширення `FeatureFlags.refuses(flag)`
(`domain/featureflag/FeatureFlagGuard.kt:5`): воно повертає true, залогувавши "Refused: <key> is
switched off", коли прапорець вимкнено. ViewModel питає його на початку методу, який захищає, і
повертається раніше.

- ViewModel із маршрутизацією intent роблять це один раз, централізовано. `ChatViewModel.handleIntent`
  спершу відображає intent на потрібний прапорець (`requiredFlag`,
  `presentation/screens/chat/ChatViewModel.kt:230`): відповідь - на `message_reply`, перехід - на
  `message_reply_jump`, видалення - на `message_delete_for_me` / `message_delete_for_everyone`,
  завантаження медіа, завантаження у переглядачі, скасування - на прапорці медіа, запис - на
  `voice_messages` або `video_notes` залежно від поточного режиму кліпу тощо; потім відмовляє до
  виконання `when` (`:190` - `:192`). Відтворення кліпу, що вже грає, виняток, тож пауза працює.
- ViewModel у стилі методів захищають кожен публічний метод. Приклади: `ChatsViewModel` для
  видалення, видалення для всіх, позначення прочитаним і оновлення свайпом
  (`presentation/screens/chats/ChatsViewModel.kt:198`, `:206`, `:214`, `:248`), `AppearanceViewModel`
  для кожного налаштування
  (`presentation/screens/settings/appearance/AppearanceViewModel.kt:56` - `:115`), `ProfileViewModel`,
  `EditProfileViewModel`, `ChangeAccountViewModel`, `MediaPickerViewModel`, `EditChatViewModel`,
  `EditContactDataViewModel`, `SearchUsersViewModel`, `ClipPlayerViewModel` і
  `ChatUserProfileViewModel`.
- `CallViewModel` перевіряє дзвінки біля джерела: `outgoingAllowed` вимагає `calls`, а для відеодзвінка
  ще й `calls_video` (`presentation/screens/call/CallViewModel.kt:177`); `startOutgoing` надсилає
  `CallExit.Unavailable` замість здійснення дзвінка (`:181` - `:185`); увімкнення динаміка, камери чи
  перемикання камери відхиляється окремо (`:229`, `:235`, `:240`).

Це PR #68 (`b23b783` додає охоронця; коміти для ViewModel йдуть за ним 2026-11-05). Покрито тестами з
`FakeFeatureFlags` (`app/src/test/.../testutil/fakes/FakeFeatureFlags.kt`).

### Як сервіс дзвінків і шлюзи дзвінків застосовують прапорці

Дзвінки починаються й закінчуються у foreground-сервісі та push-обробниках, а не на екрані, тож
сховати кнопку дзвінка їх не захищає.

- `CallMediaGate` (`data/remote/call/CallMediaGate.kt`) - єдине місце, що перетворює "дзвінок попросив
  відео" на "що дозволяє ця збірка": `media(requestedVideo, cameraGranted)` (`:16`) знижує відеодзвінок
  до лише-аудіо, коли `calls_video` вимкнено, і вмикає динамік для відеодзвінка лише коли це дозволяє
  `calls_speaker`; `allowsVideo`, `allowsSpeaker` і `allowsFlipCamera` (`:25` - `:29`) відповідають на
  окремі питання, а `showsRemoteVideo` (`:31`) ховає плитку відео співрозмовника. `CallForegroundService`
  впроваджує його (`data/remote/call/CallForegroundService.kt:56`) і питає його при старті вихідного
  дзвінка (`:133`), прийнятті вхідного (`:229`, у `handleAccept`), перемиканні динаміка (`:313`),
  вмиканні відео (`:335`, `:337`), перемиканні камери (`:350`) і показі віддаленого відео (`:702`).
- `IncomingCallGate` (`data/remote/call/IncomingCallGate.kt`) стереже інші двері - вхідний дзвінок.
  `admit(calleeId, callId)` (`:22`) спершу викликає `awaitFresh()` (`:23`), тож дзвінок із холодного
  push оцінюється за актуальними прапорцями; якщо `calls` увімкнено, він записує id дзвінка й повертає
  true, що також змушує дзвінок, який прийшов двома шляхами, дзвонити один раз (`:20`, `:24`). Якщо
  `calls` вимкнено, він записує статус `DECLINED` у сервіс сигналізації з 3-секундною межею (`:40`) і
  повертає false, тож абонента повідомлено, а не залишено дзвонити. Обидва шляхи проходять через нього:
  слухач на передньому плані `IncomingCallCoordinator`
  (`data/remote/call/IncomingCallCoordinator.kt:56`) і FCM-обробник
  `MyFirebaseMessagingService.handleCallPayload`, що досягає шлюзу через свою Hilt entry point і
  `runBlocking` (`data/remote/firebase/MyFirebaseMessagingService.kt:152`); див.
  [[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]]. Вимкнення лише `calls_video`
  усе одно дозволяє дзвінку дзвонити, як голосовому (тест
  `calls still ring when only video calls are switched off`).
- `MainActivity` застосовує ті самі прапорці до picture-in-picture без Compose local: він просить PiP
  лише коли `calls_pip` увімкнено й пропонує дії PiP для відео та динаміка лише коли це дозволяють їхні
  прапорці (`presentation/MainActivity.kt:187`, `:216`, `:224`).

Серверна сторона має відповідне правило, яке не є прапорцем, але є тією самою ідеєю. PR #64 змусив
Cloud Function `getRtcToken` відмовляти в токені, якщо викликач не є `callerId` або `calleeId` цього
дзвінка: абонент записує `callChannels/<channel>` з обома id тим самим мультишляховим оновленням, що й
сигнал (`callSignalUpdate`, `data/remote/call/FirebaseCallSignalingService.kt:159`), функція читає цей
запис і відповідає `permission-denied` будь-кому іншому (`functions/src/index.ts:259`, перевірка в
`functions/src/callAccess.ts:9`), а `database.rules.json:44` дозволяє писати його лише абоненту. Тому
абонент отримує свій токен лише після запису сигналу (`signalThenJoin`,
`CallForegroundService.kt:1033`; тест `the caller fetches its token only after the call signal is
written`). Змінений клієнт, що оминає UI, усе одно не може приєднатися до чужого дзвінка.

### Сповіщення

Функції сповіщень теж є прапорцями (`notification_inline_reply`, `notification_mark_read`,
`notification_shortcuts`, `notification_bubbles`). `ChatNotifier` будує сповіщення й питає прапорець
через `UiEntryPoints.featureFlags()`; якщо цей пошук кидає виняток, він відкочується до "увімкнено"
(`presentation/notification/ChatNotifier.kt:204` - `:208`), тож збій Hilt не може заглушити
повідомлення. Приймач дій теж без UI: `NotificationReplyReceiver` викликає `awaitFresh()`, а потім
`isEnabled`, перш ніж надіслати inline-відповідь чи позначити розмову прочитаною, і в іншому разі лише
оновлює сповіщення (`presentation/notification/NotificationReplyReceiver.kt:42` - `:60`). Push-обробник
також оновлює прапорці перед побудовою мітки кліпу
(`data/remote/firebase/MyFirebaseMessagingService.kt:111`). Див. [[Notifications_UA|Сповіщення]].

### Захист у глибину

Той самий перемикач навмисно перевіряється на кількох рівнях, бо кожен рівень закриває шлях, якого не
закриває попередній:

| Рівень | Що робить | Чому одного його не досить |
|---|---|---|
| Значення Remote Config | Єдине джерело істини плюс список "вимкнено за замовчуванням" release-збірки | Воно лише вирішує; ніщо не блокується значенням, яке ніхто не читає |
| UI (`featureEnabled`) | Прибирає кнопку, рядок чи жест | Прихований елемент - це не відмова в дії: інший екран, дія сповіщення, відновлений стек назад чи стан, що вже був у русі, усе ще можуть викликати той самий метод |
| ViewModel (`refuses`) | Відмовляє intent чи метод, хто б не викликав | Не покриває роботу, що починається поза екраном |
| Сервіс і шлюзи (`CallMediaGate`, `IncomingCallGate`, приймач) | Відмовляють у медіа дзвінка, вхідних дзвінках і діях сповіщень, що починаються з push чи broadcast, після `awaitFresh()` | Лише на боці клієнта |
| Сервер (перевірка учасника в `getRtcToken`, правила бази й Firestore) | Відмовляє в доступі до ресурсів незалежно від клієнта | Не знає про feature flag-и; захищає дані, а не функції |

Перша версія (`020711a`) ховала точки входу в UI ("Gated UI hides each feature's entry points") і вже
стерегла вхідні дзвінки та дії сповіщень; охоронець у ViewModel і `CallMediaGate` з'явилися пізніше в
PR #68 саме з причини в таблиці.

### Crashlytics

- Перемикач: збір - це значення manifest meta-data `firebase_crashlytics_collection_enabled =
  ${crashlyticsCollectionEnabled}` (`app/src/main/AndroidManifest.xml:42`, `:43`). Build-файл задає
  плейсхолдер `false` для `debug` (`app/build.gradle.kts:61`, його успадковує `e2e`) і `true` для
  `release` (`:65`), тож звіти надсилають лише release-збірки. Жоден feature flag Crashlytics не
  керує.
- Екземпляр: `FirebaseCrashlytics.getInstance()` надається в Hilt через `di/FirebaseModule.kt:59`. У
  коді застосунку немає явного виклику ініціалізації.
- Ідентифікатор користувача: `CrashlyticsUserSync.start()` (викликається з
  `MessengerApplication.kt:114`) реєструє слухач стану Firebase Auth, що задає user id Crashlytics
  як uid поточного користувача або порожній рядок після виходу
  (`data/remote/firebase/CrashlyticsUserSync.kt:14` - `:16`). Це лише uid, без номера телефону чи
  імені.
- Стан прапорців: кожен `publish()` записує користувацький ключ `flags_off` зі списком ключів, що
  зараз вимкнені, через кому (`RemoteConfigFeatureFlags.kt:82`, ім'я ключа на `:93`), тож звіт про
  збій каже, які вимикачі були активні. Це перевіряє тест (`flags turned off are reported to
  Crashlytics`).
- Що надсилається: автоматичні звіти про збої, більше нічого. У коді немає `recordException`, `log`
  чи іншого ручного виклику Crashlytics. Власний обробник неперехоплених винятків застосунку
  (`MessengerApplication.kt:103` - `:107`) логує збій і передає його обробнику, встановленому до
  нього; чи це обробник Crashlytics, залежить від порядку ініціалізації, що тут не перевірялося.

### Шаблон Remote Config

`remoteconfig.template.json` застосунок не читає. Він існує, щоб консоль показувала кожен прапорець з
описом і групою. Його 62 параметри лежать у дванадцяти `parameterGroups` (Calls, Voice and video
messages, Clip playback, Media, Messages, Chat list, Search, Your profile, Partner profile,
Appearance, Settings, Notifications). Кожен параметр має `valueType` `BOOLEAN` із
`defaultValue.useInAppDefault = true` і без умовних значень, а кожен опис каже, що робить вимкнення
(наприклад `calls`: "hides every call button, blocks outgoing calls and declines incoming calls. A
call already in progress is not ended"). Оскільки шаблон завжди каже "використати значення за
замовчуванням у застосунку", його публікація нічого не змінює; далі ви редагуєте параметр у консолі,
щоб задати `false` (або `true` для вимкненого в release прапорця).

Шаблон тримається в згоді з enum юніт-тестом: `RemoteConfigTemplateTest`
(`app/src/test/.../domain/featureflag/RemoteConfigTemplateTest.kt`) стверджує, що шаблон перелічує
кожен ключ прапорця рівно один раз і більше нічого, і що кожен параметр зберігає значення за
замовчуванням у застосунку. `app/build.gradle.kts:113` оголошує шаблон входом завдання юніт-тестів,
тож його редагування перезапускає тести. Отже, додавання прапорця - це три правки: рядок в enum, запис
у шаблоні і (лише якщо він має стартувати вимкненим у release) список `featuresOffInRelease`.

## Чому такий вибір

Що фіксують код і повідомлення комітів, а що ні. Перші два пункти прямо названо в коміті, що додав
прапорці; наступні два описують, що дозволяє дизайн, а не зафіксований мотив:

- Прапорець - це вимикач: вимкнути функцію для всіх без релізу, зокрема для встановлених збірок. Коміт,
  що їх додав, називає їх "remote kill switches" і перелічує шляхи, навмисно залишені поза ними
  (вхід, надсилання, керування дзвінком, шифрування).
- Список часу збірки дає недоробленим функціям безпечне значення за замовчуванням у release-збірках,
  поки Remote Config ще не відповів, і дозволяє значенню з консолі пізніше їх увімкнути.
- `StateFlow` незмінного значення дає дереву Compose і будь-якому не-UI коду читати ту саму відповідь
  синхронно (`isEnabled`), водночас реагуючи на зміну (`state`).
- Ієрархія батьків тримає консоль короткою: один перемикач для дзвінків замість шести.
- Чому Remote Config, а не інший механізм доставки, у коді чи повідомленнях комітів не зафіксовано.

## Розглянуті альтернативи

- Лише час збірки (`BuildConfig.FEATURES_OFF_BY_DEFAULT` без Remote Config). Це досі відправна точка,
  але сама по собі вона потребує нового релізу, щоб щось змінити.
- Лише приховування точок входу. Так працювала UI-половина першої версії, і саме від неї відходять
  пізніші перевірки у ViewModel і медіа дзвінків (PR #68); див. Захист у глибину.
- Розгортання по користувачах чи за умовами. Шаблон не має умов (`"conditions": []`), а тест забороняє
  умовні значення, тож кожен прапорець поки глобальний.

## Пов'язане

[[Architecture Overview_UA|Огляд архітектури]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] -
[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] - [[Domain Layer_UA|Доменний шар]] -
[[Data Layer_UA|Шар даних]] - [[Async and Reactivity_UA|Асинхронність і реактивність]] -
[[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]] - [[Notifications_UA|Сповіщення]] -
[[Tech Stack and Versions_UA|Технологічний стек і версії]] -
[[Build, CI and Releases_UA|Збірка, CI і релізи]]
