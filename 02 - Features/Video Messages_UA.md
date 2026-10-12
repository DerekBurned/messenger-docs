---
tags:
  - _UA
---
# Відеоповідомлення

Круглі відеонотатки ("відеокружечки"), додані в гілці `feat/voice_messages` 12.10.2026. Вони
мають спільні з [[Voice Messages_UA|голосовими повідомленнями]] жест запису, чернетки, outbox і
плеєр: торкніться мікрофона, щоб перемкнути його на камеру, затисніть, щоб записати кліп із
фронтальної камери в круглому прев'ю, відпустіть, щоб надіслати. Одержувач бачить круглу
бульбашку, яка збільшується під час відтворення, з кільцем прогресу по краю, яке можна тягнути
для перемотування.

Посилання `path:line` відносні до `app/src/main/java/com/example/messenger/`.

## Що

- Запис: CameraX 1.4.2 (`camera-core`, `camera-camera2`, `camera-lifecycle`, `camera-video`,
  `camera-view`; вже були в `gradle/libs.versions.toml` для клітинки камери в медіапікері) -
  `ProcessCameraProvider` + `Preview` + `VideoCapture<Recorder>`, а не
  `LifecycleCameraController`, який використовує `CameraCaptureScreen`.
- Відтворення: той самий екземпляр Media3 ExoPlayer 1.10, що й для голосових нотаток, з
  виводом у `TextureView`.
- Мініатюри: Coil 3 `VideoFrameDecoder` (зареєстрований глобально в `MessengerApplication`) на
  локальному файлі.
- Формат передачі: без змін. `Clip.Video` (`domain/model/voiceMessage/Clip.kt`), тип
  повідомлення `VIDEO_CLIP`, файл `chatMedia/<conversationId>/<messageId>/clip.mp4`;
  `firestore.rules`, `storage.rules`, кодек шифрування й підпис у пуші приймали його ще до цієї
  роботи.

## Як ми це використовуємо

### Складові

- `VideoNoteRecorder` (`data/media/VideoNoteRecorder.kt:94`) - `@Singleton` рівня застосунку,
  прив'язаний до головного потоку, з `StateFlow<VideoRecordState>` (`:54`): `Idle` або
  `Recording(conversationId, elapsedMs, started, locked)`.
- `VideoNoteRecordingOverlay` (`presentation/components/chat/VideoNoteRecordingOverlay.kt:37`) -
  затемнення, кругле `PreviewView`, кільце тривалості; його малює `ChatScreenContent` між
  списком і полем введення (`presentation/screens/chat/ChatScreenContent.kt:502`).
- `VideoNoteBubble` (`presentation/screens/chat/VideoNoteBubble.kt:100`) - замінює стару
  `VideoClipPlaceholder`; викликається з `MessageBubble`
  (`presentation/screens/chat/MessageBubble.kt:211`).
- `ClipPlaybackController.attachVideo` / `detachVideo` (`data/media/ClipPlaybackController.kt:224`,
  `:229`) з `VideoViewStack` (`:73`).
- `ClipKind` (`domain/model/voiceMessage/ClipKind.kt`) і `ClipRecording`
  (`data/media/VoiceRecorder.kt:55`) - запис, який передається на надсилання або в чернетки,
  голосовий чи відео.
- `IClipRepository.cachedFile` (`data/repository/ClipRepositoryImpl.kt:132`) - локальний файл
  для мініатюри, без завантаження.

### Перемикання режиму

Жест `recordGesture` кнопки мікрофона чекає 200 мс після дотику
(`presentation/components/chat/RecordMicButton.kt:60`, `HOLD_TO_RECORD_MS` на `:121`). Відпускання
в цьому вікні - це дотик: `ToggleClipMode` перемикає `ChatUiState.clipMode`
(`presentation/viewmodel/ChatViewModel.kt:219`), іконка плавно змінюється Mic <-> Videocam, а на
2 с з'являється підказка (`presentation/screens/chat/MessageComposer.kt:263`). Утримання понад
200 мс запускає запис у поточному режимі. Режим - це стан ViewModel, тому після повторного
відкриття чату він знову голосовий.

### Дозволи

Для відео потрібні `CAMERA` і `RECORD_AUDIO` (`recordPermissions`,
`presentation/screens/chat/ChatScreenWithNav.kt:53`). Відсутні запитуються разом через
`RequestMultiplePermissions`; діалоги пояснення й "вимкнено, відкрийте Налаштування" показують
текст про камеру, якщо запит прийшов із режиму відео (`requestedForVideo`, `:130`, `:235`).
`VideoNoteRecorder.start` перевіряє обидва ще раз і без них відмовляє.

### Запис

`ChatViewModel.startRecording` (`:227`) ставить на паузу кліп, що грає, і викликає
`videoNoteRecorder.start(conversationId)` (`VideoNoteRecorder.kt:154`). `start` відмовляє під час
дзвінка, без дозволів або без `AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE`, потім встановлює
`Recording(started = false)` і запускає `bindAndRecord` (`:209`):

1. `ProcessCameraProvider.awaitInstance(context)` усередині `withContext(NonCancellable)` (`:211`).
2. Фронтальна камера, якщо є, інакше задня.
3. `Recorder` з `QualitySelector.from(Quality.SD, lowerQualityOrHigherThan(SD))` і
   `setTargetVideoEncodingBitRate(1_200_000)`; `VideoCapture` з `MIRROR_MODE_ON_FRONT_ONLY`;
   `Preview`, провайдер поверхні якого під'єднує оверлей.
4. `bindToLifecycle(ProcessLifecycleOwner.get(), selector, preview, capture)`.
5. `prepareRecording` у тимчасовий файл `recording_*.mp4` з
   `setDurationLimitMillis(Clip.Video.MAX_DURATION_MS)` (2 хвилини) і
   `setFileSizeLimit(38 MB)`, `withAudioEnabled()`, `start(mainExecutor, listener)`.

`VideoRecordEvent.Start` встановлює `started` (оверлей ховає спінер), `Status` оновлює
`elapsedMs` з `recordedDurationNanos`, `Finalize` викликає `complete` (`:276`). Поле введення
використовує `RecordingBar` без хвилі (червона крапка, таймер, змах для скасування) і ту саму
доріжку блокування, що й голос.

### Як завершується запис

Кожен вихід проходить через `end(active, requested)` (`:266`), який записує запит через
`nextVideoEnd` (`:74`) - перемагає перший запит, але пізніше скасування перемагає завжди - і
викликає `recording.stop()` (або завершує одразу, якщо CameraX ще не стартувала). На `Finalize`
рішення приймає `videoFinalizeOutcome` (`:87`):

| Запит | Finalize | Результат |
|---|---|---|
| `DISCARD` (змах вліво, кошик, вихід з акаунта) | будь-який | файл видалено |
| будь-який | порожній файл, менше 1 с, помилка кодера/рекордера | файл видалено |
| `SEND` (відпускання, надіслати, стоп) | `NONE` | передається в `ChatViewModel.sendRecording` |
| немає (палець ще тримає) | `DURATION_LIMIT_REACHED`, `FILE_SIZE_LIMIT_REACHED`, `SOURCE_INACTIVE` | чернетка |
| `DRAFT` (фон, дзвінок, втрата фокуса, вихід з чату) | придатний | чернетка |

`stop` (`:183`) чекає результату до 5 с. Якщо очікування скасовано (користувач вийшов з чату,
поки CameraX ще писала файл) або вийшов час, `keepUnclaimedAsDraft` (`:303`) зберігає вже
отриманий запис як чернетку або перетворює очікуваний `SEND` на `DRAFT`
(`videoEndWithoutWaiter`, `:77`), тож кліп, який користувач вирішив надіслати, ніколи не
губиться мовчки. `release` (`:313`) відв'язує лише use case-и цього рекордера, віддає фокус і
повертається в `Idle`.

Переривання такі самі, як у голосу: активний `ActiveCallHolder`, `ProcessLifecycleOwner` нижче
`STARTED`, втрата фокуса, `ChatViewModel.onCleared` і вихід екрана чату з композиції
(`ChatScreenWithNav.kt:166`, крім змін конфігурації).

### Чернетки

`VoiceDraftStore` тепер зберігає обидва види: `VoiceDraftMeta(durationMs, waveform, kind)`
(`data/media/VoiceDraftStore.kt:20`), медіафайл `<conversationId>.<kind.extension>` у
`filesDir/voice_drafts/`. Чернетка з часів до відео не має поля `kind` і читається як голос.
`VoiceDraftBar` показує відеочернетку як круглу мініатюру 36 dp з "Video message".

### Надсилання

`sendRecording` (`ChatViewModel.kt:274`) створює `Clip.Video(durationMs, sizeBytes)` і викликає
`clipRepository.sendClip` - той самий outbox, шифрування, тайм-аут завантаження за розміром і
підтверджений запис у Firestore, що й для голосової нотатки. Максимум 38 MB вкладається в ліміт
40 MB правила Storage для `clip.*`; 2 хвилини при 1,2 Мбіт/с - це близько 18 MB.

### Бульбашка

`VideoNoteBubble`:

- 200 dp у спокої, анімується до 264 dp, поки це активний кліп.
- Мініатюра: `produceState` запитує `videoFrame(messageId)` -> `ChatViewModel.clipVideoFrame` ->
  `cachedFile` і малює перший кадр через Coil; без локального файлу (отриманий кліп, ще не
  відтворений) показує градієнт бульбашки.
- Коли кліп активний і завантажений, складається `VideoNoteSurface` - `TextureView`,
  під'єднаний до плеєра. ExoPlayer розтягує відео на весь view, тож `centerCropScale` (`:93`)
  рахує матрицю `setTransform`, яка обрізає кадр 4:3 до квадрата.
- Кільце на Canvas: білий трек 30%; під час відтворення біла дуга прогресу від 12 години й
  повзунок; під час завантаження чи вивантаження - дуга акцентного кольору з `transfer.progress`.
- Центральна кнопка використовує `voiceButton(...)` з `VoiceBubble` (грати, спінер, скасувати
  надсилання, повторити) і ховається під час відтворення.
- Чипи: тривалість (позиція під час відтворення) з крапкою непрослуханого, і час + статус.
- Жести (`pointerInput(interactive, isActive)`): під час відтворення дотик у межах 28 dp від
  кільця (`isOnRing`, `:90`) споживає жест і перемотує на `ringFraction` (`:85`, 0 на 12 годині,
  за годинниковою стрілкою) при кожному русі; будь-який інший дотик чекає відпускання, коротшого
  за тайм-аут довгого натискання, і перемикає відтворення. Перетягування не по кільцю не
  споживаються, тому `SwipeToReply` і `LongPressMessage` працюють далі.
- Прев'ю контекстного меню передає `interactive = false` і ніколи не створює поверхню (`:266`).

### Відтворення

`playClip` передає `isVideo = message.clip is Clip.Video`; `ClipPlaybackState` містить `isVideo`,
`videoWidth`, `videoHeight` (з `onVideoSizeChanged`). Контролер тримає `VideoViewStack`:
`attachVideo` кладе view нагору й виводить у нього, `detachVideo` знімає його і, якщо він був
показаний, виводить у view під ним. Коли відеонотатка закінчується, `shouldAutoAdvance` (`:88`)
повертає false: її позначено переглянутою, відтворення зупиняється (бульбашка повертається до
мініатюри) замість переходу до наступної голосової нотатки. Прокручування бульбашки за межі
екрана від'єднує її поверхню; звук грає далі, а `ClipMiniPlayer` підписує його "Video message".

## Чому такий вибір

- `ProcessCameraProvider` замість `LifecycleCameraController`: контролер не дає задати бітрейт
  кодера, а правило Storage обмежує кліпи 40 MB. Прив'язка до `ProcessLifecycleOwner` тримає
  камеру поза Activity: синглтон не зберігає посилання на Activity, а камера закривається разом
  із видимістю застосунку - саме тоді, коли має статися переривання.
- `withContext(NonCancellable)` навколо `awaitInstance`: у CameraX 1.4.2 скасування корутини, що
  чекає, скасовує глобальний для процесу future ініціалізації CameraX (перевірено у вихідному
  коді), і всі функції CameraX ламаються до завершення процесу. Ранній вихід не повинен цього
  робити.
- Рекордер-синглтон з тими ж правилами власності `conversationId` і чернеток, що й
  `VoiceRecorder`, робить обидва види взаємозамінними для поля введення й ViewModel.
- `TextureView` замість `SurfaceView`: лише `TextureView` можна обрізати до кола засобами
  Compose і масштабувати матрицею.
- Один `ExoPlayer` для обох видів: одночасно все одно грає лише один кліп, а аудіофокус, пауза
  на дзвінки, швидкість і міні-плеєр дістаються безкоштовно.
- Стек view замість одного поля: дві композиції можуть законно тримати поверхню для одного
  кліпу під час рекомпозиції чи прокручування; стек робить власність незалежною від порядку.
- Дзеркалення фронтальної камери відповідає тому, що користувач бачив у прев'ю.

## Розглянуті й відхилені альтернативи

- `LifecycleCameraController` (як у `CameraCaptureScreen`): відхилено через відсутність
  керування бітрейтом (бітрейт SD за замовчуванням може перевищити 40 MB за 2 хвилини).
- Квадратне обрізання самого запису через 1:1 `ViewPort`: не використано; кадр записується 4:3 і
  обрізається під час показу, що працює на будь-якому пристрої без підтримки обрізання кодером.
- Media3 `PlayerView` / `media3-ui-compose`: відхилено, щоб не інфлейтити XML і не додавати
  залежність заради однієї круглої поверхні.
- Перемикання камер під час запису: стійкий запис у CameraX 1.4 експериментальний; не робили.
- Автовідтворення круглих відео без звуку в списку (як у Telegram): не зроблено.
- Збільшення ліміту понад 2 хвилини: обґрунтування ще не зафіксоване - уточнити в мейнтейнера.

## Тести

- Модульні: `VideoFinalizeOutcomeTest` (результати, `nextVideoEnd`, `videoEndWithoutWaiter`),
  `VideoNoteGeometryTest` (`ringFraction`, `isOnRing`, `centerCropScale`), `VideoViewStackTest`
  (передача стеком, `shouldAutoAdvance`), `VoiceDraftMetaTest` (kind туди й назад, старі
  чернетки).
- Інструментальні (запускати через `adb shell am instrument`, ніколи не
  `connectedDebugAndroidTest`): `RecordGestureTest` (дотик перемикає й ніколи не стартує;
  утримання стартує й надсилає; змах угору блокує; змах вліво скасовує),
  `VideoNoteBubbleGestureTest` (дотик у центрі перемикає, перетягування по кільцю перемотує,
  довге натискання в обгортках чату відкриває меню, змах не по кільцю відповідає, перемотування
  за годинниковою стрілкою від 12 години ніколи не відповідає) і два випадки `VideoNoteBubble` у
  `ChatComponentContrastTest`.

## Пов'язане

[[Voice Messages_UA|Голосові повідомлення]] - [[Chat and Messaging_UA|Чат і повідомлення]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]] - [[Video Messages]] (англійською) -
[[Video Messages - Code]] - [[Video Messages - Explained Simply]] -
[[Video Messages - Basic Example]]
