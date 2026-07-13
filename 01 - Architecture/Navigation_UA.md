---
tags:
  - _UA
---
# Навігація

## Що і версія

Navigation 3 (`androidx.navigation3` `1.1.3`, `gradle/libs.versions.toml:23`) плюс
`androidx.lifecycle:lifecycle-viewmodel-navigation3` для ViewModel-сторів на кожен запис. Маршрути
- це `@Serializable` `NavKey` (kotlinx-serialization). Це новіша модель "стек навігації як стан", а
не старіший граф `NavHost`/`NavController`.

## Типізовані маршрути

`presentation/navigation/Routes.kt` оголошує кожен пункт призначення як серіалізовний `NavKey`.
Корені вкладок і сторінки налаштувань - це `data object` (`AuthRoute`, `ChatsRoute`, `CallsRoute`,
`SettingsRoute`, `PrivacyRoute`, ...); пункти з аргументами - це `data class` (`Routes.kt:55`):

```kotlin
@Serializable
data class ChatRoute(
    val conversationId: String,
    val partnerId: String = "",
    val partnerName: String = "",
    val partnerAvatarUrl: String? = null,
) : NavKey

@Serializable
data class CallRoute(
    val partnerId: String,
    val partnerName: String,
    val partnerPhone: String,
    val video: Boolean = false,
) : NavKey
```

Аргументи строго типізовані - жодних рядкових ключів у місці виклику, жодного ручного пакування
bundle під час навігації.

## Стек навігації та NavDisplay

`presentation/navigation/navigation.kt` - це вся навігаційна поверхня. `AppNavigation`
(`navigation.kt:77`) перемикає за станом автентифікації (зібраним через
`collectAsStateWithLifecycle`, `navigation.kt:82`) між `AuthDisplay` та `MainDisplay`.

Стек навігації - це стан: `rememberNavBackStack(ChatsRoute)` (`navigation.kt:130`). Навігація - це
просто мутування того списку - `backStack.add(ChatRoute(...))` щоб запушити,
`backStack.removeLastOrNull()` щоб виштовхнути. `NavDisplay` (`navigation.kt:205`) рендерить
верхній запис і анімує переходи:

```kotlin
NavDisplay(
    backStack = backStack,
    onBack = { backStack.removeLastOrNull() },
    entryDecorators = listOf(
        rememberSaveableStateHolderNavEntryDecorator(),
        rememberViewModelStoreNavEntryDecorator(),
    ),
    transitionSpec = { slideInHorizontally(tween(300)) { it } + fadeIn(tween(300)) togetherWith ... },
    entryProvider = entryProvider { entry<ChatsRoute> { ... }; entry<ChatRoute> { key -> ... } },
)
```

Два декоратори мають значення:
- `rememberSaveableStateHolderNavEntryDecorator` зберігає `rememberSaveable`-стан кожного запису
  між навігаціями.
- `rememberViewModelStoreNavEntryDecorator` дає кожному запису стека власний `ViewModelStore`, тож
  `hiltViewModel()` усередині запису видає ViewModel, скоуповану до цього запису й очищену при
  виштовхуванні.

`entryProvider` відображає кожен тип `NavKey` на його composable через `entry<T> { }`
(`navigation.kt:220` і далі). Перемикання вкладок (`navigateToTab`, `navigation.kt:173`) виштовхує
некореневі записи, потім міняє корінь, тож вкладки поводяться як односхідцевий перемикач, а не
стек.

## Передавання нав-аргументів у SavedStateHandle ViewModel

Записи Navigation 3 не автоматично живлять `SavedStateHandle` для `@HiltViewModel`. Застосунок
мостить це через `ProvideNavArgs` (`navigation.kt:451`), який обгортає поточний
`ViewModelStoreOwner` у `NavArgsViewModelStoreOwner` (`navigation.kt:464`), що впроваджує `Bundle`
як `DEFAULT_ARGS_KEY` у `CreationExtras` ViewModel:

```kotlin
entry<ChatRoute> { key ->
    ProvideNavArgs(
        "conversationId" to key.conversationId,
        "partnerId" to key.partnerId,
        "partnerName" to key.partnerName,
        "partnerAvatarUrl" to (key.partnerAvatarUrl ?: ""),
    ) {
        ChatScreenWithNav(...)
    }
}
```

Далі за течією `ChatViewModel` читає саме ці ключі зі свого `SavedStateHandle`
(`presentation/viewmodel/ChatViewModel.kt:62`). `CallViewModel` та `ChatUserProfileViewModel`
отримують свої аргументи так само. Оскільки значення - це серіалізовані поля маршруту, що течуть
через `SavedStateHandle`, вони переживають смерть процесу. Див.
[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] щодо аспекту смерті процесу.

## Оверлеї дзвінка й пошуку поверх навігації

`MainDisplay` накладає ненавігаційний UI поверх `NavDisplay` у тому ж `Box`:
- `FloatingTabBar` для коренів вкладок (`navigation.kt:394`),
- `IncomingCallBar`, докована `ActiveCallBar` та `CallBarOverlay` для активного дзвінка,
  показані/приховані на основі `ActiveCallHolder.state` та `CallBarPresenter.mode`
  (`navigation.kt:143`, `:412`),
- `SearchResultsOverlay` під час пошуку у вкладці (`navigation.kt:383`).

Панелі дзвінка живуть у корені навігації (не всередині екрана), тож вони зберігаються між змінами
екранів, поки дзвінок активний. Див. [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].

## Переходи спільних елементів

`MainDisplay` обгортає `NavDisplay` у `SharedTransitionLayout` і надає
`LocalSharedTransitionScope` / `LocalNavAnimatedVisibilityScope` (`navigation.kt:203`,
`ProvideNavAnimatedScope`, `:442`), тож аватари й елементи профілю анімуються між списком розмов,
чатом і екранами профілю.

## Чому такий підхід

- Стек навігації як спостережуваний стан лягає на ментальну модель Compose - навігація це
  `add`/`remove` над запам'ятованим списком, що піддається diff і тестуванню.
- Типізовані `NavKey` прибирають баги рядкових маршрутів і роблять аргументи перевіреними
  компілятором.
- ViewModel-стори на кожен запис через декоратор дають коректні життєві цикли ViewModel без
  окремого скоупу ViewModel для нав-графа.

Компроміс: живлення `SavedStateHandle` потребує кастомного мосту
`ProvideNavArgs`/`NavArgsViewModelStoreOwner` (`navigation.kt:451`-`:481`), що є більшою обв'язкою,
ніж стара обв'язка `SavedStateHandle` у Navigation-Compose. Обґрунтування вибору Navigation 3 над
Navigation-Compose не зафіксовано в коді; задокументовано як поточний стан.

## Пов'язане

[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] -
[[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]] -
[[Architecture Overview_UA|Огляд архітектури]] - [[Tech Stack and Versions_UA|Технологічний стек і версії]]
