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

- `FloatingTabBar` / `SearchField` (`presentation/components/common/FloatingTabBar.kt:59`, `:172`)
  - пігулко-подібний бар, що анімує круглу кнопку пошуку в повноширинне поле вводу.
- `SearchResultsOverlay` (`presentation/components/search/SearchResultsOverlay.kt:44`) -
  повноекранна поверхня результатів/підказок.
- `SearchUsersViewModel` (`presentation/viewmodel/SearchUsersViewModel.kt`) - драйвер запиту й два
  потоки нещодавніх; стан у `SearchUsersUiState`.
- `RecentContactsRepository` (`data/repository/RecentContactsRepository.kt`) + `ObxRecentContact`
  - сховище нещодавніх (див. [[Data Layer_UA|Шар даних]]).
- Прив'язка у `presentation/navigation/graph/MainDisplay.kt`.

## Відкриття пошуку: бар анімується в поле вводу

Стан пошуку піднято в `MainDisplay.kt` як `isSearching` / `searchQuery` (`MainDisplay.kt:210`,
`:211`). Згорнутий `SearchField` - це кругла кнопка 48dp; натискання викликає `onSearchOpen`
(`MainDisplay.kt:570`), що перемикає `isSearching = true`. `FloatingTabBar` анімує float
`progress` 0->1 (`FloatingTabBar.kt:75`) і `lerp`-ить ширину поля від `RestingSearchSize` до повної
ширини бару (`:94`), тож кнопка видимо виростає в поле вводу. Ліва лупа лишається декоративною
іконкою пошуку; круглий елемент на кінцевому краї (де була кнопка) проявляється як закриваючий
**X** (`SearchField`, `FloatingTabBar.kt:253`). Фокус запитується на текстовому полі через
`LaunchedEffect(isSearching)` (`FloatingTabBar.kt:187`).

### Порядок шарів: бар має малюватися поверх оверлея

`SearchResultsOverlay` заповнює екран `WallpaperBackground`. Він компонується **перед**
`FloatingTabBar` у тому самому внутрішньому боксі (`MainDisplay.kt:521`, потім `:564`), тож таб-бар -
і його анімоване поле вводу та X - малюється поверх оверлея, а не ховається за ним. Оверлей
резервує верхній відступ (72dp, коли бар угорі), тож результати починаються під баром.
Неправильний порядок (оверлей після бару) ховає все поле пошуку за шпалерами - саме та вада, яку
цей порядок і запобігає.

## Закриття пошуку

- Кінцевий X у `SearchField` викликає `onSearchClose`, що скидає `isSearching`/`searchQuery`.
- Системне / жестове "назад" теж закриває: `BackHandler(enabled = isTabRoute && isSearching)`
  (`MainDisplay.kt:582`) виконує те саме скидання, тож "назад" виходить із пошуку, а не із застосунку.

## Результати проти підказок

`SearchResultsOverlay` дебаунсить запит на 300 мс, потім викликає `viewModel.searchUsers` і
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
  (`SearchUsersViewModel.kt:60`), коли натиснуто результат/підказку.
- `recordOpened(userId, name, avatar)` викликається з `ChatViewModel.init`
  (`ChatViewModel.kt:124`) щоразу при відкритті чату, і ще раз, коли завантажиться аватар
  співрозмовника (`ChatViewModel.kt:176`), щоб зберегти аватар.
- `observeRecentSearched` / `observeRecentOpened` (`RecentContactsRepository.kt:39`, `:46`) -
  реактивні читання на базі `queryFlow`, впорядковані за міткою часу за спаданням, зіставлені в
  `List<User>` і зібрані в `SearchUsersUiState` в `init` в'юмоделі (`SearchUsersViewModel.kt:31`).

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
[[Async and Reactivity_UA|Асинхронність і реактивність]]
