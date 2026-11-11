---
tags:
  - _UA
---
# Пошук і підказки

Пошук користувачів із плаваючого таб-бару: натискання кнопки пошуку розгортає бар у поле вводу, а
повноекранний оверлей показує або живі результати за користувачами, або, коли поле порожнє,
підказки-результати в стилі Telegram (контакти, на які можна натиснути), поділені на "Нещодавно
шукані" та "Нещодавно відкриті". Пошук знаходить користувачів, щоб почати розмову - це не пошук по
повідомленнях.

## Складові

- `FloatingTabBar` / `SearchField` (`presentation/components/common/FloatingTabBar.kt:61`, `:177`)
  - пігулко-подібний бар, що анімує круглу кнопку пошуку в повноширинне поле вводу.
- `SearchResultsOverlay` (`presentation/components/search/SearchResultsOverlay.kt:44`) -
  повноекранна поверхня результатів/підказок.
- `SearchUsersViewModel` (`presentation/components/search/SearchUsersViewModel.kt`) - драйвер запиту й два
  потоки нещодавніх; стан у `SearchUsersUiState`.
- `RecentContactsRepository` (`data/repository/RecentContactsRepository.kt`) + `ObxRecentContact`
  - сховище нещодавніх (див. [[Data Layer_UA|Шар даних]]).
- Прив'язка у `presentation/navigation/graph/MainDisplay.kt`.

## Відкриття пошуку: бар анімується в поле вводу

Стан пошуку піднято в `MainDisplay.kt` як `isSearching` / `searchQuery` (`MainDisplay.kt:221`,
`:222`). Згорнутий `SearchField` - це кругла кнопка 48dp (`RestingSearchSize`, `FloatingTabBar.kt:58`);
натискання викликає `onSearchOpen` (`MainDisplay.kt:595`), що перемикає `isSearching = true`.
`FloatingTabBar` анімує float `progress` 0->1 (`FloatingTabBar.kt:78`) і `lerp`-ить ширину поля від
`RestingSearchSize` до повної ширини бару (`:98`), тож кнопка видимо виростає в поле вводу. Ліва лупа лишається декоративною
іконкою пошуку; круглий елемент на кінцевому краї (де була кнопка) проявляється як закриваючий
**X** (`SearchField`, `FloatingTabBar.kt:177`). Фокус запитується на текстовому полі через
`LaunchedEffect(isSearching)` (`FloatingTabBar.kt:192`).

### Порядок шарів: бар має малюватися поверх оверлея

`SearchResultsOverlay` заповнює екран `WallpaperBackground`. Він компонується **перед**
`FloatingTabBar` у тому самому внутрішньому боксі (`MainDisplay.kt:546`, потім `:589`), тож таб-бар -
і його анімоване поле вводу та X - малюється поверх оверлея, а не ховається за ним. Оверлей
резервує верхній відступ (72dp, коли бар угорі), тож результати починаються під баром.
Неправильний порядок (оверлей після бару) ховає все поле пошуку за шпалерами - саме та вада, яку
цей порядок і запобігає.

## Закриття пошуку

- Кінцевий X у `SearchField` викликає `onSearchClose`, що скидає `isSearching`/`searchQuery`.
- Системне / жестове "назад" теж закриває: `BackHandler(enabled = isTabRoute && isSearching)`
  (`MainDisplay.kt:607`) виконує те саме скидання, тож "назад" виходить із пошуку, а не із застосунку.

## Результати проти підказок

`SearchResultsOverlay` дебаунсить запит на 300 мс (`SearchResultsOverlay.kt:54`), потім викликає `viewModel.searchUsers` і
розгалужується: завантаження -> спінер; помилка -> повідомлення; **порожній запит -> дві секції
підказок**; непорожній без збігів -> "No users found"; інакше `LazyColumn` з `SearchResultRow`.

Гілка порожнього запиту рендерить "Recently searched", потім "Recently opened" (другу
дедупльовано проти першої за id користувача), кожна - заголовок `SectionHeader` над тим самим
`SearchResultRow`. Якщо обидва списки порожні, показується підказка "Type a username to search".
Натискання будь-якого рядка - результату чи підказки - викликає `createConversationWithUser`, що
також записує натискання як пошук.

## Сховище нещодавніх

`RecentContactsRepository` (простий box-орієнтований `@Singleton`, без інтерфейсу - тієї ж форми,
що й `CallHistoryRepository`) читає/пише `ObxRecentContact` (`@Unique userId`, `lastSearchedAt` /
`lastOpenedAt`):

- `recordSearched(user)` викликається з `SearchUsersViewModel.createConversationWithUser`
  (`SearchUsersViewModel.kt:73`-`:76`), коли натиснуто результат/підказку.
- `recordOpened(userId, name, avatar)` викликається з `ChatViewModel.init`
  (`ChatViewModel.kt:131`) щоразу при відкритті чату, і ще раз, коли завантажиться аватар
  співрозмовника (`ChatViewModel.kt:183`), щоб зберегти аватар.
- `observeRecentSearched` / `observeRecentOpened` (`RecentContactsRepository.kt:39`, `:46`) -
  реактивні читання на базі `queryFlow`, впорядковані за міткою часу за спаданням, зіставлені в
  `List<User>` і зібрані в `SearchUsersUiState` в `init` в'юмоделі (`SearchUsersViewModel.kt:36`-`:49`).

## Feature flag

Пошук - за прапорцем `search` (`FeatureFlag.SEARCH`) з дочірнім `search_recent`
(`FeatureFlag.SEARCH_RECENT`); див. [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]].

- `SEARCH`, UI: `FloatingTabBar` показує кнопку пошуку лише коли він увімкнений (`FloatingTabBar.kt:77`,
  `:89`, `:96`); `MainDisplay` скидає `isSearching`/`searchQuery`, коли прапорець вимикається або маршрут
  не є вкладкою (`MainDisplay.kt:223`-`:228`), і монтує `SearchResultsOverlay` лише з увімкненим
  прапорцем (`:545`).
- `SEARCH`, ViewModel: `SearchUsersViewModel.searchUsers` і `createConversationWithUser` одразу
  повертаються через `featureFlags.refuses(FeatureFlag.SEARCH)` (`SearchUsersViewModel.kt:58`, `:74`).
  Порожній запит усе одно очищає результати до перевірки.
- `SEARCH_RECENT`, UI: гілка порожнього запиту ховає обидві секції підказок, коли він вимкнений
  (`SearchResultsOverlay.kt:93`-`:96`).
- `SEARCH_RECENT`, ViewModel: два потоки нещодавніх `combine`-яться з живим станом прапорців, тож
  вимкнення миттєво спорожнює `recentSearched` / `recentOpened` (`SearchUsersViewModel.kt:37`-`:49`).
  Запис нещодавніх не обмежується; обмежений лише показ.

## Чому такий дизайн

- Повторне використання анімації розгортання плаваючої кнопки пошуку (замість окремого екрана
  пошуку) лишає пошук за один тап з будь-якої вкладки й уникає переходу навігації.
- Підказки-результати (контакти) тут кращі за підказки-запити, бо ціль пошуку - завжди конкретна
  людина; показ нещодавніх людей - це ярлик за один тап до типового випадку.
- Один рядок `ObxRecentContact` на користувача з двома мітками часу дозволяє одному сховищу живити
  обидві секції й гарантує, що людина ніколи не з'явиться двічі, без другої сутності чи join.

## Пов'язане

[[Conversations List_UA|Список розмов]] - [[Chat and Messaging_UA|Чат і обмін повідомленнями]] -
[[Data Layer_UA|Шар даних]] - [[Navigation_UA|Навігація]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]] - [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]]
