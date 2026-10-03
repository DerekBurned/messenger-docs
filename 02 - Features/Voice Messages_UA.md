---
tags:
  - _UA
---
# Голосові повідомлення

Голосові нотатки із затисканням (вийшли в 0.1.13.1): запис зі змахом вліво для скасування й
змахом угору для блокування, перервані записи зберігаються як чернетки на кожен чат, надсилання
через стійкий до збоїв outbox із підтвердженим записом у Firestore, відтворення через ExoPlayer
з тимчасовим аудіофокусом і автопереходом, бульбашка з хвилею й перемотуванням та міні-плеєр
(з 1x/1.5x/2x), що супроводжує на всіх вкладках. Відеокліпи мають ту саму модель і формат передачі,
але показують лише бульбашку-заглушку.

## Складові

- Модель: `Message.Clip` (`domain/model/Message.kt:57`) з `Clip`
  (`domain/model/voiceMessage/Clip.kt:6`): `Clip.Voice` (url, тривалість, розмір, `fileKey`,
  `waveform`) і `Clip.Video`. Кожен підтип має власні `previewLabel` ("Voice message" /
  "Video message"), `fileExtension` (`m4a` / `mp4`) і `maxDurationMs` (голос: 1 година).
- Запис: `VoiceRecorder` (`data/media/VoiceRecorder.kt:77`), `@Singleton` рівня застосунку,
  обмежений головним потоком, з `StateFlow<VoiceRecordState>`.
- Чернетки: `VoiceDraftStore` (`data/media/VoiceDraftStore.kt:33`) - одна чернетка на розмову,
  зберігається у `filesDir/voice_drafts/`.
- Надсилання, завантаження, відновлення, прибирання: `ClipRepositoryImpl`
  (`data/repository/ClipRepositoryImpl.kt:48`) за `IClipRepository`.
- Відтворення: `ClipPlaybackController` (`data/media/ClipPlaybackController.kt:80`) - один
  ExoPlayer, з `StateFlow<ClipPlaybackState>`.
- Файли: `MediaCache` (`clip_outbox`, `voice_drafts`, `chat_media`) і `MediaFileTransfer`
  (зашифроване вивантаження/завантаження), див. [[Data Layer_UA|Шар даних]].
- UI: `RecordMicButton.kt` (модифікатор `recordGesture`), `RecordingBar`, `VoiceDraftBar`,
  `VoiceWaveform`, `ReplyQuote` у `presentation/components/chat/`; `VoiceBubble` і `MessageComposer`
  у `presentation/screens/chat/`; `ClipMiniPlayer`
  (`presentation/components/common/ClipMiniPlayer.kt:64`) з `ClipPlayerViewModel`, змонтований у
  `MainDisplay` (`presentation/navigation/graph/MainDisplay.kt:620`).
- Обв'язка: голосові intent-и `ChatIntent`, які обробляє `ChatViewModel` (`StartRecording`,
  `LockRecording`, `CancelRecording`, `FinishRecording`, `InterruptRecording`, `SendVoiceDraft`,
  `DiscardVoiceDraft`, `MicPermissionDenied`, `DismissMicPrompt`, `PlayClip`, `SeekClip`,
  `RetryClip`, `CancelClipSend`).
- Бекенд: `firestore.rules` (`hasValidClip`), `storage.rules` (`chatMedia/...`) і прев'ю пушу у
  функції `onMessageCreated` (`CLIP_PREVIEWS`, `functions/src/index.ts:19`).

## Запис

Кнопка мікрофона має `Modifier.recordGesture` (`presentation/components/chat/RecordMicButton.kt`).
Дотик викликає `StartRecording`; `ChatViewModel.startRecording` відмовляє, якщо в цьому чаті вже є
чернетка, ставить на паузу кліп, що грає, і викликає `voiceRecorder.start(conversationId)`
(`VoiceRecorder.kt:130`).

`start` відмовляє під час дзвінка, потім бере `AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE` (відмовляє,
якщо не дали) і налаштовує `MediaRecorder` на AAC у MPEG-4, моно, 44.1 кГц, 64 кбіт/с із
`setMaxDuration(Clip.Voice.MAX_DURATION_MS)`. Невдалі `prepare()`/`start()` звільняють рекордер і
видаляють файл. Під час запису він кожні 100 мс опитує `maxAmplitude`: стан тримає останні 64
нормалізовані значення для живої смуги, а повний список при завершенні стискається до 50 кошиків
(`downsampleWaveform`).

Результати жесту, вирішені `recordGestureEnd(locked, slidToCancel, released)`:

- змах уліво більше 100 dp -> `CancelRecording`, файл видаляється;
- змах угору більше 80 dp -> `LockRecording`, палець можна прибрати, запис триває; `RecordingBar`
  показує скасування й стоп, а і стоп, і кнопка надсилання композера завершують і надсилають
  (`MessageComposer.kt:158`, `:212`);
- відпускання -> `FinishRecording` -> `stop(conversationId)` -> надіслати;
- будь-яке інше завершення жесту (система скасувала дотик) -> `InterruptRecording`, що зберігає
  чернетку.

Запис належить одному чату: `VoiceRecordState.Recording` несе `conversationId`, а
`visibleTo(conversationId)` (`VoiceRecorder.kt:46`) показує його як `Idle` усім іншим чатам, тож
заблокований запис у чаті A не можна завершити чи скасувати з чату B. Записи коротші за 500 мс
відкидаються. `MessageComposer` тримає екран увімкненим під час запису
(`KeepScreenOn(enabled = uiState.recording is VoiceRecordState.Recording)`).

## Переривання стають чернетками

`interrupt()` (`VoiceRecorder.kt:191`) завершує запис і передає його в `VoiceDraftStore.save`
замість надсилання. Спрацьовує, коли:

- застосунок іде з переднього плану (`ProcessLifecycleOwner` опускається нижче `STARTED`);
- починається дзвінок (`ActiveCallHolder.state` стає не null);
- аудіофокус втрачено назовсім чи тимчасово (`interruptsRecording`) - так звичайний мобільний
  дзвінок завершує запис;
- досягнуто ліміту в 1 годину (`MEDIA_RECORDER_INFO_MAX_DURATION_REACHED`);
- жест завершився без відпускання (див. вище);
- ViewModel чату очищено (`onCleared` -> `interrupt(conversationId)`).

`VoiceDraftStore.save` переносить файл у `filesDir/voice_drafts/<conversationId>.m4a` і пише
JSON-супровідник `{durationMs, waveform}`. `load` виконується при відкритті чату й прибирає
осиротілі чи пошкоджені чернетки, тож чернетка переживає смерть процесу. Поки чернетка є, композер
показує `VoiceDraftBar` (тривалість, хвиля, видалення) замість текстового поля, а кнопка дії
надсилає її (`SendVoiceDraft` -> `take` -> звичайний шлях надсилання). Викидають запис лише змах
уліво і явне видалення.

Відома поведінка: відкриття шторки сповіщень під час утримання мікрофона не на кожному пристрої
скасовує дотик, тож запис просто триває.

## Надсилання через outbox

`sendVoice` будує `Clip.Voice(durationMs, sizeBytes, waveform)` і викликає
`clipRepository.sendClip(conversationId, file, clip, replyTo = replyingTo)`. `sendClip`
(`ClipRepositoryImpl.kt:82`):

1. відхиляє кліп, довший за свій ліміт;
2. переносить файл у `filesDir/clip_outbox/<messageId>.m4a`, пише рядок `Message.Clip` зі статусом
   `SENDING` (з цитатою відповіді з `replyTo.replySnippet()`) і оновлює прев'ю розмови на
   "Voice message";
3. `launchSend` реєструє лінивий `Job` у `sendJobs` і запускає його.

`uploadAndSend` (`:236`):

- вивантажує файл з outbox у `chatMedia/<conversationId>/<messageId>/clip.m4a`, шифруючи ключем
  на файл, якщо розмова шифрується, під `withTimeoutOrNull(clipUploadTimeout(size))` - 30 с плюс
  одна секунда на кожні 8000 байтів (`:381`);
- при збої: скасувати й очистити передачу, позначити `FAILED`;
- при успіху: `updateClip` пише URL і ключ у транзакції ObjectBox, що заново читає рядок і
  повертає `null`, якщо кліп тим часом видалено, - тоді завантажений файл видаляється зі Storage і
  нічого не надсилається;
- надсилає документ Firestore з `confirmWithin = 20.seconds`, тож запис, що так і не дійшов до
  сервера, стає `FAILED`, а не висить у `SENDING`;
- позначає `SENT` і переносить файл з outbox у кеш.

Повтор (`RetryClip` -> `retry`, `:187`) атомарно забирає лише `FAILED`, невидалений рядок без живого
job. Скасування під час вивантаження (`CancelClipSend` -> `cancelSend`, `:205`) скасовує job і
передачу та, якщо кліп ще не пішов, видаляє рядок і файл з outbox.

## Шифрування

Голосовий кліп шифрується як медіа: аудіофайл - ключем AES-GCM на файл, а ключ і хвиля
подорожують усередині зашифрованого навантаження повідомлення як `ClipSecret(key, waveform)`
(`data/crypto/model/ClipSecret.kt`, будується `Clip.toSecret()` у
`data/crypto/E2eeMessageCodec.kt:119`). Відкрита частина документа зберігає URL, тривалість і
розмір, але порожню хвилю. Див. [[End-to-End Encryption_UA|Наскрізне шифрування]].

## Відновлення після смерті процесу

`ClipRepositoryImpl.init` запускає `recoverAfterRestart` (`ClipRepositoryImpl.kt:283`):

- видаляє застарілі тимчасові файли (`recording_*`, `*.dl`, `*.part`, `*.enc`), старші за цей
  процес;
- переносить у outbox файли ненадісланих кліпів зі старого кешу;
- позначає `FAILED` кожен кліп, що лишився `SENDING` з попереднього процесу без живого job
  (`strandedClipIds`, `ClipRepositoryImpl.kt:358`), тож він показує кнопку повтору;
- узгоджує outbox (`outboxFate`, `ClipRepositoryImpl.kt:371`): тримає файли рядків
  `SENDING`/`FAILED`, переносить у кеш файли надісланих рядків, прибирає файли без рядка чи з
  видаленим рядком.

Самі вивантаження переживають лише живий процес; убите вивантаження стає повтором при наступному
запуску.

## Отримання й відтворення

Кнопка відтворення в бульбашці викликає `PlayClip` -> `playbackController.toggle(ClipQueueItem(...))`.
`clipToggle` (`ClipPlaybackController.kt:60`) вирішує PLAY / PAUSE / RESUME / IGNORE за станом і за
кліпом, який плеєр справді тримає, тож натискання B під час A чисто перемикає, а натискання під час
завантаження ігнорується.

`play` (`:208`) відмовляє під час дзвінка, вивантажує попередній кліп, показує стан завантаження й
просить у `clipRepository.ensureLocal(id)` (`ClipRepositoryImpl.kt:112`) файл: копію з outbox, потім
з кешу, інакше завантаження. Завантаження йдуть через `InFlight`, тож паралельні запити того самого
кліпу ділять одне завантаження; файл розшифровується `fileKey` і атомарно переміщується на місце.
Якщо файлу немає, стан фіксує `failedMessageId`, а бульбашка показує значок повтору; помилка плеєра
також прибирає кешовану копію. Плеєр будується один раз з атрибутами мовлення та
`setHandleAudioBecomingNoisy(true)` (від'єднання навушників ставить на паузу), а наступний голосовий
кліп завантажується наперед.

Аудіофокус - `AUDIOFOCUS_GAIN_TRANSIENT`, тож музичний застосунок продовжує після нотатки.
`focusReaction` (`:69`): постійна втрата ставить на паузу й віддає фокус; тимчасова (включно з
"можна приглушити" - мовлення ставиться на паузу, а не приглушується) ставить на паузу й
відновлює при поверненні, але лише якщо грало. Дзвінок у застосунку ставить відтворення на паузу
без автовідновлення.

Коли кліп закінчується (`onCompleted`, `:272`), він позначається прослуханим (зникає крапка), і
грає наступний голосовий кліп у тому ж чаті за часом, потім за id - від будь-якого відправника,
включно з власними й уже прослуханими, і включно з тими, що прийшли під час прослуховування
(`nextVoiceClipAfter`, `ClipRepositoryImpl.kt:362`). Інакше відтворення зупиняється.

`VoiceBubble` (`presentation/screens/chat/VoiceBubble.kt`) показує хвилю з перемотуванням,
тривалість, крапку непрослуханого, цитату відповіді й одну кнопку дії, обрану `voiceButton(...)`
(`:60`): скасування надсилання з кільцем прогресу під час вивантаження, кільце під час
завантаження, повтор надсилання для власного `FAILED`-кліпу, спінер під час підготовки, повтор
відтворення після збою, інакше пауза/відтворення.

`ClipMiniPlayer` живе в корені навігації, тож відтворення триває на всіх вкладках. Має
відтворення/паузу, закриття, цикл швидкості 1x -> 1.5x -> 2x (`nextClipSpeed`,
`ClipMiniPlayer.kt:51`) і натискання, що відкриває чат кліпу (`ClipPlayerViewModel.resolveTarget`).

## Видалення й вихід

Видалення голосового кліпу для всіх також прибирає його файл зі Storage
(`DeleteMessageUseCase` -> `clipRepository.deleteRemoteFile`). Кожен пристрій стежить за
`observeDeletedClipIds` і прибирає свою локальну копію, файл з outbox, job надсилання й передачу
(`purgeLocal`, `ClipRepositoryImpl.kt:176`); контролер відтворення зупиняється, якщо поточний кліп
видалено. Вихід зупиняє відтворення, скасовує запис, очищає чернетки й стирає outbox, чернетки й
медіакеш (див. [[Authentication_UA|Автентифікація]]).

## Дозвіл на мікрофон

Натискання мікрофона без `RECORD_AUDIO` показує діалог пояснення, коли Android це дозволяє, інакше
запитує дозвіл. Разова відмова показує тост; постійна відмова встановлює `micDenied`, і
`MicPermissionDialog` (`ChatScreenWithNav.kt:267`) пропонує "Open settings"
(`ACTION_APPLICATION_DETAILS_SETTINGS`).

## Мітки, пуші й правила бекенду

- Одне джерело міток: `Clip.previewLabel` живить прев'ю у списку чатів (локально й у документі
  розмови, також для зашифрованих чатів) і цитати відповіді. Пуші FCM відображають тип повідомлення
  через `clipLabelFor` на `R.string.clip_voice_message`, а `onMessageCreated` використовує
  `CLIP_PREVIEWS`, тож пуш голосової нотатки звучить "Voice message" навіть при зашифрованому тексті.
- `firestore.rules`: `hasValidClip` приймає повідомлення `VOICE_CLIP` / `VIDEO_CLIP` з мапою `clip`
  і URL до 2048 символів.
- `storage.rules`: `chatMedia/{conversationId}/{messageId}/{fileName}` - читання, запис і видалення
  лише для учасників розмови (крос-сервісний `firestore.get` на розмову), файли кліпів до 40 МБ,
  інші медіа до 100 МБ, `application/octet-stream` або image/video/audio.

## Чому такий дизайн

- Outbox у `filesDir`, окремо від кешу, означає, що ненадісланий запис ніколи не витісняється й
  може бути узгоджений після збою; перенесення в кеш при успіху лишає одну копію.
- Підтвердження запису Firestore і повторне читання рядка в транзакції зупиняють два найгірші
  сценарії: кліп, що назавжди застряг у `SENDING`, і видалений кліп, воскреслий через пізнє
  вивантаження.
- Чернетки замість мовчазного викидання: переривання (фон, дзвінки, ліміт) - не вибір користувача,
  тож запис зберігається; викидають його лише явні жести.
- Обмеження рекордера й плеєра головним потоком робить переходи станів простими без блокувань;
  `ExoPlayer` треба використовувати з потоку, де його створено.
- Тимчасовий фокус і відсутність автовідновлення після дзвінків відповідають тому, як користувач
  очікує поведінки голосової нотатки поруч із музикою й дзвінками.
- Таймаути, що масштабуються з розміром, дають одногодинній нотатці (близько 29 МБ при 64 кбіт/с)
  вивантажитися на повільному каналі, на відміну від фіксованих 35 с для медіа.

## Відомі обмеження

- Вивантаження переживає лише живий процес (без WorkManager); убите вивантаження потребує повтору.
- Шифрування - AES-GCM усього файлу, тож одногодинний кліп - це близько 29 МБ у пам'яті.
- Ще не зроблено: підтвердження прослуховування, керування з екрана блокування (media session),
  перегляд перед надсиланням.

## Пов'язане

[[Chat and Messaging_UA|Чат і обмін повідомленнями]] - [[End-to-End Encryption_UA|Наскрізне шифрування]] -
[[Data Layer_UA|Шар даних]] - [[Async and Reactivity_UA|Асинхронність і реактивність]] -
[[Notifications_UA|Сповіщення]] - [[Authentication_UA|Автентифікація]] -
[[Voice Messages - Code]] - [[Voice Messages - Explained Simply]] - [[Voice Messages - Basic Example]]
