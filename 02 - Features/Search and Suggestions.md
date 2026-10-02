# Search and Suggestions

User search from the floating tab bar: pressing the search button expands the bar into an input
field, and a full-screen overlay shows either live user results or, when the field is empty,
Telegram-style *result* suggestions (tappable contacts) split into "Recently searched" and
"Recently opened". Search matches users to start a conversation with - it is not message search.

## Pieces

- `FloatingTabBar` / `SearchField` (`presentation/components/common/FloatingTabBar.kt:59`, `:172`)
  - the pill bar that animates a round search button into a full-width input field.
- `SearchResultsOverlay` (`presentation/components/search/SearchResultsOverlay.kt:44`) - the
  full-screen results/suggestions surface.
- `SearchUsersViewModel` (`presentation/viewmodel/SearchUsersViewModel.kt`) - deb-free query
  driver plus the two recents streams; state in `SearchUsersUiState`.
- `RecentContactsRepository` (`data/repository/RecentContactsRepository.kt`) + `ObxRecentContact`
  - the recents store (see [[Data Layer]]).
- Host wiring in `presentation/navigation/graph/MainDisplay.kt`.

## Opening search: the bar animates into an input field

Search state is hoisted in `MainDisplay.kt` as `isSearching` / `searchQuery` (`MainDisplay.kt:210`,
`:211`). The collapsed `SearchField` is a 48dp round button; tapping it calls `onSearchOpen`
(`MainDisplay.kt:570`), flipping `isSearching = true`. `FloatingTabBar` animates a `progress`
float 0->1 (`FloatingTabBar.kt:75`) and `lerp`s the field width from `RestingSearchSize` to the
full bar width (`:94`), so the button visibly grows into an input field. The leading magnifier
stays as a decorative search icon; the round control on the trailing edge (where the button was)
fades in as the close **X** (`SearchField`, `FloatingTabBar.kt:253`). Focus is requested on the
text field via `LaunchedEffect(isSearching)` (`FloatingTabBar.kt:187`).

### Layering: the tab bar must draw on top of the overlay

`SearchResultsOverlay` fills the screen with a `WallpaperBackground`. It is composed **before**
the `FloatingTabBar` inside the same inner box (`MainDisplay.kt:521` then `:564`), so the tab bar -
and its animating input field and X - draws on top of the overlay instead of being hidden behind
it. The overlay reserves top padding (72dp when the bar is at the top) so results start below the
bar. Getting this order wrong (overlay after the bar) hides the whole search field behind the
wallpaper - the bug this ordering exists to prevent.

## Closing search

- The trailing X in `SearchField` calls `onSearchClose`, which resets `isSearching`/`searchQuery`.
- System / gesture back also closes it: `BackHandler(enabled = isTabRoute && isSearching)`
  (`MainDisplay.kt:582`) runs the same reset, so back exits search instead of leaving the app.

## Results vs suggestions

`SearchResultsOverlay` debounces the query 300ms then calls `viewModel.searchUsers` and branches:
loading -> spinner; error -> message; **blank query -> two suggestion sections**; non-empty with
no hits -> "No users found"; otherwise a `LazyColumn` of `SearchResultRow`.

The blank-query branch renders "Recently searched" then "Recently opened" (the second deduplicated
against the first by user id), each a labelled `SectionHeader` over the same `SearchResultRow`.
If both lists are empty it falls back to the "Type a username to search" hint. Tapping any row -
result or suggestion - runs `createConversationWithUser`, which also records the tap as a search.

## The recents store

`RecentContactsRepository` (plain box-backed `@Singleton`, no interface - same shape as
`CallHistoryRepository`) reads/writes `ObxRecentContact` (`@Unique userId`, `lastSearchedAt` /
`lastOpenedAt`):

- `recordSearched(user)` is called from `SearchUsersViewModel.createConversationWithUser`
  (`SearchUsersViewModel.kt:60`) when a result/suggestion is tapped.
- `recordOpened(userId, name, avatar)` is called from `ChatViewModel.init`
  (`ChatViewModel.kt:124`) whenever a chat is opened, and again once the partner avatar loads
  (`ChatViewModel.kt:176`) so the stored avatar is filled in.
- `observeRecentSearched` / `observeRecentOpened` (`RecentContactsRepository.kt:39`, `:46`) are
  `queryFlow`-backed reactive reads ordered by their timestamp desc, mapped to `List<User>`, and
  collected into `SearchUsersUiState` in the ViewModel's `init` (`SearchUsersViewModel.kt:31`).

## Why this design

- Reusing the floating search button's expand animation (rather than a separate search screen)
  keeps search one tap from any tab and avoids a navigation transition.
- Result suggestions (tappable contacts) beat query suggestions here because the search target is
  always a specific person; showing recent people is a one-tap shortcut to the common case.
- A single `ObxRecentContact` row per user with two timestamps lets one store back both sections
  and guarantees a person is never listed twice, without a second entity or a join.

## Related

[[Conversations List]] - [[Chat and Messaging]] - [[Data Layer]] - [[Navigation]] -
[[Async and Reactivity]]
