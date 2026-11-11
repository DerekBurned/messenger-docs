---
tags:
  - _UA
---
# Профіль чату та спільні медіа

Екран профілю співрозмовника, що відкривається з чату, і вкладки Photos / Videos / Voice / Files на
ньому, які перелічують усе, чим ви обмінялися. Додано PR #54 (злито 2026-11-03).

Посилання `path:line` - відносно `app/src/main/java/com/example/messenger/`, якщо вони не
починаються з кореневої назви репозиторію (`firestore.indexes.json`).

## Складові

- `ChatUserProfileScreen` (`presentation/screens/chatuserprofile/ChatUserProfileScreen.kt:10`) -
  тонкий екран: збирає `ChatUserProfileViewModel.uiState` і `clipPlayback` та передає колбеки.
- `ChatUserProfileScreenContent`
  (`presentation/screens/chatuserprofile/ChatUserProfileScreenContent.kt:64`) - UI без стану:
  шапка, інформаційна картка, safety number, далі вкладки спільних медіа та список; також тримає
  два повноекранні переглядачі (фото аватара та спільні медіа).
- `ChatUserProfileViewModel` (`presentation/screens/chatuserprofile/ChatUserProfileViewModel.kt:38`)
  - звичайний `ViewModel` (не базовий MVI) з `MutableStateFlow<ChatUserProfileUiState>`.
- `ChatUserProfileUiState` (`presentation/screens/chatuserprofile/ChatUserProfileUiState.kt:8`) -
  додає `mediaTab`, `sharedMedia`, `mediaLoading`, `mediaEndReached`, `mediaPagesLoaded` і
  `transfers`.
- `SharedMedia`, `SharedMediaEntry` та `sharedMediaOf`
  (`presentation/screens/chatuserprofile/SharedMedia.kt:9`, `:15`, `:21`) - чиста функція, що
  перетворює купу повідомлень на три списки для вкладок. `MediaTab`
  (`presentation/screens/chatuserprofile/MediaTab.kt:3`) - enum `PHOTO, VIDEO, VOICE, FILE`.
- `sharedMediaSection` і `MediaTabsRow`
  (`presentation/components/profile/ProfileSharedMedia.kt:62`,
  `presentation/components/profile/ChatUserProfileComponents.kt:188`) - смуга вкладок і
  список/сітка під нею. Сама комірка - це `MediaCell` чату
  (`presentation/components/media/MediaAlbum.kt:205`), використана без змін.
- Дані: `IMessageRepository.fetchSharedMediaPage` (`domain/repository/IMessageRepository.kt:20`) ->
  `MessageRepositoryImpl.fetchSharedMediaPage` (`data/repository/MessageRepositoryImpl.kt:171`) ->
  `FirestoreService.fetchMessagesOfTypes` (`data/remote/firebase/FirestoreService.kt:317`);
  складений індекс - у `firestore.indexes.json`.

## Як потрапити в профіль

`ChatUserProfileRoute` отримав необов'язковий `conversationId`
(`presentation/navigation/Routes.kt:66`-`:70`). Верхня панель чату (`ChatTopBar.kt:93`, `:106`) і
його жест свайпу вліво (див. [[Chat and Messaging_UA|Чат і обмін повідомленнями]]) додають маршрут з
id відкритої розмови (`presentation/navigation/graph/MainDisplay.kt:508`-`:518`), а запис
навігації передає його у ViewModel через `SavedStateHandle` (`MainDisplay.kt:449`-`:454`). Коли
профіль відкривають звідти, де розмови немає (екран контактів, `MainDisplay.kt:415`), id порожній, і
ViewModel шукає розмову сам: першу розмову з двома учасниками, що містить співрозмовника
(`findConversationWith`, `ChatUserProfileViewModel.kt:140`). Якщо розмови немає, медіа немає:
секція одразу переходить у порожній стан (`:122`-`:124`).

## Звідки беруться медіа: два джерела, злиті разом

Спільні медіа - це об'єднання того, що вже є на пристрої, і того, що підвантажується із сервера:

1. Локальне: `messageRepository.getMessagesStream(conversationId)` - той самий реактивний
   ObjectBox-потік, що й у чаті, тож усе вже кешоване з'являється першим, а живі оновлення (нове
   фото, що надійшло, поки профіль відкрито) з'являються без оновлення сторінки
   (`ChatUserProfileViewModel.kt:130`).
2. Віддалене: `loadMoreMedia` (`:147`) просить `fetchSharedMediaPage` наступну сторінку з
   `MEDIA_PAGE_SIZE = 60` повідомлень (`:197`). Запит Firestore бере лише повідомлення
   `TYPE_MEDIA` і `TYPE_VOICE_CLIP` (`MessageRepositoryImpl.kt:249`), найновіші першими за
   `timestamp`, і продовжує після останнього повідомлення попередньої сторінки (`startAfter` на цьому
   документі, `FirestoreService.kt:335`). Якщо якоря вже немає, сторінка порожня. Сторінки
   зберігаються лише в пам'яті, у `remoteMedia` (`ChatUserProfileViewModel.kt:54`); у ObjectBox
   нічого не пишеться, тож закриття профілю їх відкидає.

Їх поєднує `combine(local, remoteMedia)`; віддалені рядки кладуться першими, а локальні
перезаписують їх за id, тож повідомлення, що є в обох, показується один раз, і перемагає локальна
копія (з реальним станом надсилання й кешованими файлами) (`:130`-`:132`). Далі `sharedMediaOf`
відкидає видалені повідомлення, сортує від новіших до старіших і розкладає результат: фото - це
`MediaItem` типу `IMAGE`, відео - елементи типу `VIDEO`, а голосові - рядки `Message.Clip`, у яких
кліп - `Clip.Voice` (`SharedMedia.kt:21`-`:30`). Альбом з кількох фото дає по одному запису на
елемент, кожен пам'ятає своє батьківське повідомлення. Відеонотатки (`TYPE_VIDEO_CLIP`) не
запитуються й тут не з'являються.

Два правила тримають серверний запит узгодженим із чатом:

- Запит передає `cutoff = cutoffFor(conversationId)` - `clearedAtForMe` розмови
  (`MessageRepositoryImpl.kt:47`, `:180`), тож медіа, очищені через "очистити для себе", не
  повертаються із сервера. Див. [[Conversations List_UA|Список розмов]].
- Кожен отриманий документ проходить `applyDeletedFor(dto, codec.decode(dto))`
  (`FirestoreService.kt:340`): повідомлення, видалені для цього користувача, позначаються видаленими
  (і відкидаються `sharedMediaOf`), а зашифровані розшифровуються так само, як у чаті.

## Пагінація

`mediaEndReached` стає true, коли сторінка повертає менше ніж 60 повідомлень (`:160`) або запит
провалюється (`:166`) - збій зупиняє подальшу пагінацію замість повторів у циклі. Одночасно працює
лише одна задача сторінки (`pageJob`, `:150`). Рахуються повідомлення, а не елементи, тож сторінка з
60 повідомлень може містити понад 60 фото.

Тригер живе в UI, а не у ViewModel: наприкінці списку `sharedMediaSection` показує за пріоритетом
спінер, поки `mediaLoading` істинне, інакше (коли є ще сторінки) невидимий елемент-сентинел, чий
`LaunchedEffect(mediaPagesLoaded, mediaTab)` викликає `onLoadMore`, інакше порожній стан
(`ProfileSharedMedia.kt:99`-`:106`). Оскільки ключ містить `mediaPagesLoaded`, сентинел спрацьовує
знову після кожної сторінки, доки він ще скомпонований, - тож вкладка з малою кількістю збігів
(скажімо, Voice, коли більшість сторінок - фото) продовжує підвантажувати, доки сентинел не вийде
з поля зору або історія не закінчиться. Усі вкладки ділять один курсор і один віддалений кеш.

## Сітка і список голосових

- Photos і Videos використовують `mediaGrid` (`ProfileSharedMedia.kt:110`): записи нарізаються на
  рядки по `GRID_COLUMNS = 4` (`:60`, `:118`), кожна комірка квадратна (`aspectRatio(1f)`) із
  закругленням 8 dp, а короткі рядки добиваються спейсерами, щоб комірки зберігали розмір. Рядки
  мають ключем id першого елемента.
- Комірка - це `MediaCell` чату: показує заглушку BlurHash, поки файл не закешовано, далі зображення
  (з бейджем відтворення для відео); незавантажена комірка показує кнопку завантаження, а активне
  передавання - скрим із прогресом і скасуванням (`MediaAlbum.kt:224`-`:247`).
- Voice використовує `SharedVoiceRow` (`ProfileSharedMedia.kt:147`): кнопку
  відтворення/паузи/спінера/повтору, відправника ("You" або відображуване ім'я співрозмовника) і
  дату з тривалістю, що замінюються живою хвилею, поки цей кліп грає.
- Files завжди порожній стан (`MediaTab.FILE -> true`, `:76`): застосунок поки не вміє надсилати
  звичайні файли. Кожна вкладка має власний текст порожнього стану, наприклад "No photos yet"
  (`emptyTextFor`, `:270`).

## Відкриття фото чи відео

Дотик до комірки встановлює `openMediaId` (`ChatUserProfileScreenContent.kt:182`); тоді екран
будує джерела переглядача із записів поточної вкладки (відео на вкладці Videos, інакше фото,
`:198`-`:218`) і показує `FullscreenMediaPager` з тапнутого елемента. Сторінками стають лише елементи,
чий файл уже є в кеші медіа й має ненульову довжину (`:202`-`:203`), а комірка клікабельна лише коли
завантажена й не передається (`MediaAlbum.kt:227`), тож пейджер гортає лише завантажені медіа;
незавантажений елемент треба спершу завантажити з його комірки.

## Відтворення голосових із профілю

Дотик до голосового рядка викликає `toggleVoice` (`ChatUserProfileViewModel.kt:171`). Він реєструє
кожен голосовий кліп зі списку в репозиторії кліпів (`registerRemoteClip`,
`data/repository/ClipRepositoryImpl.kt:114`), який тримає кліпи, що є лише на сервері, у мапі в
пам'яті, до якої звертається `loadClip` (`:338`-`:339`), тож `ensureLocal` може завантажити кліп,
якого ніколи не було в ObjectBox (`:118`-`:132`), а `nextVoiceClip` - автоматично переходити далі по
кешованих і віддалених кліпах (`:153`-`:160`). Далі він передає `ClipQueueItem` спільному
`ClipPlaybackController` (`data/media/ClipPlaybackController.kt:192`) - тому самому плеєру, що й
бульбашки чату та міні-плеєр, тож відтворення з профілю ставить на паузу будь-який інший кліп. Див.
[[Voice Messages_UA|Голосові повідомлення]].

## Зашифровані медіа

Жоден код профілю сам нічого не розшифровує. Зашифровані медіа працюють, бо розшифрування
відбувається рівнем нижче:

- Метадані та ключ на файл подорожують усередині зашифрованого корисного навантаження повідомлення.
  `E2eeMessageCodec.decode` повертає `fileKey` і `blurHash` кожного елемента на `MediaItem`
  (`data/crypto/E2eeMessageCodec.kt:103`-`:111`) і так само для кліпів (`:113`-`:114`). І локальний
  потік, і шлях сторінок Firestore декодують повідомлення раніше, ніж його побачить ViewModel.
- Завантаження викликає `mediaRepository.downloadMedia(item)` (`MediaRepositoryImpl.kt:72`), що
  передає `item.fileKey` у `MediaFileTransfer.download`; з ключем він завантажує шифротекст і
  розшифровує його у файл кешу, без ключа - просто переміщує відкритий файл на місце
  (`data/media/MediaFileTransfer.kt:46`-`:57`).
- Повідомлення, яке не вдається розшифрувати на цьому пристрої, повертається як заглушка (див.
  [[End-to-End Encryption_UA|Наскрізне шифрування]]) без придатного файлу, тож відкривати нічого.

Файл кешу - єдина відкрита копія; переглядач читає саме його.

## Порожні та завантажувальні стани

- Перше завантаження: `mediaLoading = true` від моменту старту `loadSharedMedia` (`:119`), тож
  вкладка показує спінер (`MediaLoading`, `ProfileSharedMedia.kt:241`), поки визначається розмова й
  завантажується перша сторінка.
- Нічого немає: коли завантаження завершилося без записів для вкладки, `MediaEmptyState` (`:257`)
  показує текст вкладки.
- Межа сторінки: кінцевий спінер, поки вантажиться наступна сторінка; без спінера після
  `mediaEndReached`.
- Немає розмови, перша сторінка провалилась або прапорець вимкнено: без спінера, лише порожній
  стан (або взагалі без секції, коли прапорець вимкнено).

## Feature flag

`partner_profile_shared_media` (`FeatureFlag.PARTNER_PROFILE_SHARED_MEDIA`) керує всією секцією; див.
[[Feature Flags and Remote Config_UA|Feature flags і Remote Config]]. Він застосовується у двох місцях:

- UI: `ChatUserProfileScreenContent.kt:81` читає його, а рядок вкладок і `sharedMediaSection`
  випускаються лише коли він увімкнений (`:177`-`:189`).
- ViewModel: `init` пропускає `loadSharedMedia`, коли `featureFlags.refuses(...)` (`:76`), а
  `loadMoreMedia` одразу повертається (`:148`), тож вимкнена секція не робить читань Firestore, навіть
  якщо щось її викличе.

Пов'язані дочірні можливості обмежуються окремо: відтворення голосових потребує `CLIP_PLAYBACK`
(`ProfileSharedMedia.kt:163`, `:170`; `ChatUserProfileViewModel.kt:176`, що все ж дозволяє
поставити на паузу кліп, який уже грає); кнопка завантаження комірки потребує `MEDIA_DOWNLOAD`
(`MediaAlbum.kt:244`; `ChatUserProfileViewModel.kt:182`); а скасування передавання потребує
`MEDIA_CANCEL_TRANSFER` (`MediaAlbum.kt:294`; `ChatUserProfileViewModel.kt:187`, `:192`). Опис
прапорця в Remote Config: "Off: hides the Photos, Videos, Voice and Files tabs on the chat profile"
(`remoteconfig.template.json:268`-`:271`).

## Чому такий дизайн

- Спершу локальне, потім серверний хвіст: ObjectBox уже тримає свіжі медіа чату, тож вкладки миттєві
  й живі, а Firestore потрібен лише для старішої історії.
- Віддалені сторінки в пам'яті не роздувають локальну базу медіа, які користувач може ніколи не
  відкрити, ціною повторного завантаження при кожному візиті.
- Повторне використання `MediaCell`, `FullscreenMediaPager` і `ClipPlaybackController` дає ту саму
  поведінку прогресу передавання, скасування, шифрування й аудіофокусу, що й у чаті.
- Пагінація від сентинела в UI (ключ `mediaPagesLoaded`) означає, що ViewModel не потребує стану
  прокрутки: він знає лише "завантажити наступну сторінку".

## Альтернативи, які розглядалися / відхилені

- Запис отриманих сторінок в ObjectBox: не зроблено; у повідомленні коміту сказано, що старіші медіа
  "held in memory only" (причина там не названа).
- Вкладка Files зі справжнім вмістом: вкладка існує, але це постійний порожній стан, бо застосунок не
  може надсилати файли.

## Відомі обмеження

- Віддалені сторінки втрачаються при закритті профілю; другий візит знову пагінує із сервера.
- Індексуються лише фото, відео й голосові; відеонотатки та картки дзвінків не перелічуються.
- Переглядач гортає лише завантажені медіа.
- Один курсор сторінок спільний для всіх вкладок, тож рідко вживана вкладка (Voice) може підтягнути
  багато сторінок інших типів, перш ніж заповниться.
- Збій сторінки завершує пагінацію для цього візиту (`:166`), без кнопки повтору.
- Firestore потребує складеного індексу `messages(type ASC, timestamp DESC)`
  (`firestore.indexes.json:20`-`:25`); без нього віддалений запит провалюється, і вкладка показує
  лише локально кешовані медіа.

## Пов'язане

[[Chat and Messaging_UA|Чат і обмін повідомленнями]] - [[Voice Messages_UA|Голосові повідомлення]] -
[[Video Messages_UA|Відеоповідомлення]] - [[End-to-End Encryption_UA|Наскрізне шифрування]] -
[[Conversations List_UA|Список розмов]] - [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]] -
[[Data Layer_UA|Шар даних]] - [[Async and Reactivity_UA|Асинхронність і реактивність]] -
[[Navigation_UA|Навігація]] - [[Chat Profile and Shared Media - Code]] -
[[Chat Profile and Shared Media - Explained Simply]] - [[Chat Profile and Shared Media - Basic Example]]
