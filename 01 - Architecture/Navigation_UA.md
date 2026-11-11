---
tags:
  - _UA
---
# Навігація

## Що і версія

Navigation 3 (`androidx.navigation3` `1.1.3`, `gradle/libs.versions.toml:24`) плюс
`androidx.lifecycle:lifecycle-viewmodel-navigation3` для ViewModel-сторів на кожен запис. Маршрути
- це `@Serializable` `NavKey` (kotlinx-serialization). Це новіша модель "стек навігації як стан", а
не старіший граф `NavHost`/`NavController`.

## Типізовані маршрути

`presentation/navigation/Routes.kt` оголошує кожен пункт призначення як серіалізовний `NavKey`.
Корені вкладок і сторінки налаштувань - це `data object` (`AuthRoute`, `ChatsRoute`, `CallsRoute`,
`SettingsRoute`, `PrivacyRoute`, `DeleteAccountRoute`, ...); пункти з аргументами - це
`data class` (`Routes.kt:58`, `:77`):

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

## Шлюз сесії

Навігаційна поверхня живе в `presentation/navigation/graph/`: `AppNavigation.kt`,
`AuthDisplay.kt`, `MainDisplay.kt`, `ProvideNavArgs.kt` і `ProvideNavAnimatedScope.kt`.
`AppNavigation` (`AppNavigation.kt:27`) збирає `AuthViewModel.state` і перемикає за
`authState.session` - це `SessionState` (`domain/model/SessionState.kt:3`):

```kotlin
when (val session = authState.session) {
    is SessionState.Active -> { /* під-шлюз E2EE, нижче: Ready -> MainDisplay, інакше KeyRestoreScreen */ }
    is SessionState.SignedOut -> AuthDisplay(authViewModel = authViewModel)
    SessionState.Resolving -> Unit
}
```

Раніше він перемикав за голим прапорцем входу Firebase. Вхід із телефоном без профілю у Firestore
тоді монтував `MainDisplay` ще до перевірки профілю й відкидав назад до щойно скомпонованого
`AuthDisplay`, втрачаючи введені номер телефону й OTP. `ObserveSessionStateUseCase`
(`domain/usecase/auth/ObserveSessionStateUseCase.kt:20`) тепер доходить до `Active` лише після
підтвердження профілю:

- немає користувача Firebase -> `SignedOut()`;
- триває реєстрація чи вхід (`IAuthFlowGate.isBusy`, `InMemoryAuthFlowGate` у `data/service/`) ->
  `Resolving`, тож шлюз не втручається посеред потоку;
- інакше стежить за `userRepository.observeRemoteUser(uid)`: профіль позначено `deleted` ->
  `SignedOut(ACCOUNT_DELETED)`; профіль є -> `Active(user)`; відсутність, підтверджена сервером
  (`snapshot.authoritative`) -> `SignedOut(PROFILE_MISSING)`; промах кешу чи помилка -> `Active` з
  ідентичністю Firebase, тож офлайн-старт ніколи не змушує вийти.

`AuthViewModel.observeSession` (`presentation/screens/auth/AuthViewModel.kt:111`) зберігає сесію в
стані й для `SignedOut` з причиною запускає `endInvalidSession` (`:120`): від'єднує присутність,
виконує use case виходу й показує відповідну помилку. Той самий слухач виводить цей пристрій, коли
акаунт видалено деінде. `Resolving` не рендерить нічого поверх шпалер. Див.
[[Authentication_UA|Автентифікація]].

### Другий шлюз: готовність шифрування

Після роботи над шифруванням у PR #79 активної сесії недостатньо, щоб дійти до головного UI.
`AppNavigation` також збирає `KeyRestoreViewModel.status` (`AppNavigation.kt:34`) і для `Active` сесії
питає `e2eeStatus.stateFor(uid)` (`data/crypto/E2eeState.kt:12`, що для іншого uid повертає
`Resolving`): лише `E2eeState.Ready` монтує `MainDisplay`; кожен інший стан (`Resolving`,
`NeedsLink`, `Behind`, `Offline`) показує `KeyRestoreScreen` (`AppNavigation.kt:45` - `:60`) із
можливістю вийти. Сенс у тому, щоб UI чату ніколи не відкривався на пристрої, який ще не може
прочитати власні повідомлення. `SecurityRoute` і `RecoveryCodeRoute` (`Routes.kt:52`, `:55`) - це
шлях у застосунку назад до коду відновлення згодом. Механіка - в
[[End-to-End Encryption_UA|Наскрізне шифрування]].

## Стек навігації та NavDisplay

Стек навігації - це стан: `rememberNavBackStack(ChatsRoute)` (`MainDisplay.kt:130`). Навігація - це
просто мутування того списку - `backStack.add(ChatRoute(...))` щоб запушити,
`backStack.removeLastOrNull()` щоб виштовхнути. `NavDisplay` (`MainDisplay.kt:279`) рендерить
верхній запис і анімує переходи:

```kotlin
NavDisplay(
    backStack = backStack,
    onBack = { popFromCall() },
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

`onBack` іде через `popFromCall()` (`MainDisplay.kt:246`): вихід із `CallRoute` з увімкненим відео
згортає дзвінок у відеоплитку, а не скидає його.

`entryProvider` відображає кожен тип `NavKey` на його composable через `entry<T> { }`
(`MainDisplay.kt:298` і далі). Перемикання вкладок (`navigateToTab`, `MainDisplay.kt:235`)
виштовхує некореневі записи, потім міняє корінь, тож вкладки поводяться як односхідцевий
перемикач, а не стек. До видалення акаунта ведуть із Privacy:
`PrivacyScreen(onDeleteAccount = { backStack.add(DeleteAccountRoute) })` (`MainDisplay.kt:357`) і
`entry<DeleteAccountRoute>` (`:360`).

## Передавання нав-аргументів у SavedStateHandle ViewModel

Записи Navigation 3 не автоматично живлять `SavedStateHandle` для `@HiltViewModel`. Застосунок
мостить це через `ProvideNavArgs` (`ProvideNavArgs.kt:18`), який обгортає поточний
`ViewModelStoreOwner` у `NavArgsViewModelStoreOwner` (`ProvideNavArgs.kt:31`), що впроваджує `Bundle`
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
(`presentation/screens/chat/ChatViewModel.kt:87`). `CallViewModel` та `ChatUserProfileViewModel`
отримують свої аргументи так само. Оскільки значення - це серіалізовані поля маршруту, що течуть
через `SavedStateHandle`, вони переживають смерть процесу. Див.
[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] щодо аспекту смерті процесу.

## Оверлеї дзвінка й пошуку поверх навігації

`MainDisplay` накладає ненавігаційний UI поверх `NavDisplay` у тому ж `Box`:
- `FloatingTabBar` для коренів вкладок (`MainDisplay.kt:589`),
- `IncomingCallBar`, докована `ActiveCallBar` та `CallBarOverlay` для активного дзвінка,
  показані/приховані на основі `ActiveCallHolder.state` та `CallBarPresenter.mode`
  (`MainDisplay.kt:152`, `:153`; `IncomingCallBar` `:612`, `ActiveCallBar` `:625`, `CallBarOverlay` `:679`),
- `SearchResultsOverlay` під час пошуку у вкладці (`MainDisplay.kt:546`), показується лише поки
  увімкнено прапорець `SEARCH` (`searchEnabled`, `MainDisplay.kt:223`).

Міні-плеєр і плаваюча бульбашка дзвінка так само вмикаються через
`featureEnabled(FeatureFlag.CLIP_MINI_PLAYER)` (`MainDisplay.kt:179`) та `CALLS_FLOATING_BUBBLE`
(`:154`); див. [[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]].

Панелі дзвінка живуть у корені навігації (не всередині екрана), тож вони зберігаються між змінами
екранів, поки дзвінок активний. Див. [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].

## Переходи спільних елементів

`MainDisplay` обгортає `NavDisplay` у `SharedTransitionLayout` і надає
`LocalSharedTransitionScope` / `LocalNavAnimatedVisibilityScope` (`MainDisplay.kt:277`,
`ProvideNavAnimatedScope`, `ProvideNavAnimatedScope.kt:9`), тож аватари й елементи профілю анімуються між списком розмов,
чатом і екранами профілю.

## Чому такий підхід

- Стек навігації як спостережуваний стан лягає на ментальну модель Compose - навігація це
  `add`/`remove` над запам'ятованим списком, що піддається diff і тестуванню.
- Типізовані `NavKey` прибирають баги рядкових маршрутів і роблять аргументи перевіреними
  компілятором.
- ViewModel-стори на кожен запис через декоратор дають коректні життєві цикли ViewModel без
  окремого скоупу ViewModel для нав-графа.
- Шлюз за розв'язаним `SessionState` замість прапорця Firebase означає, що основний UI монтується
  лише для користувача, чий профіль існує, а вхід, що триває, шлюз ніколи не руйнує.

Компроміс: живлення `SavedStateHandle` потребує кастомного мосту
`ProvideNavArgs`/`NavArgsViewModelStoreOwner` (`ProvideNavArgs.kt`, 48 рядків), що є більшою обв'язкою,
ніж стара обв'язка `SavedStateHandle` у Navigation-Compose. Обґрунтування вибору Navigation 3 над
Navigation-Compose не зафіксовано в коді; задокументовано як поточний стан.

## Пов'язане

[[Presentation Layer (MVI)_UA|Шар представлення (MVI)]] -
[[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]] -
[[Architecture Overview_UA|Огляд архітектури]] - [[Authentication_UA|Автентифікація]] - [[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]] -
[[Tech Stack and Versions_UA|Технологічний стек і версії]]
