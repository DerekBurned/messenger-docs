---
tags:
  - _UA
---
# Шар представлення (MVI)

Шар представлення - це Jetpack Compose плюс контракт Model-View-Intent. Ця нотатка охоплює
базовий клас MVI, два стилі ViewModel, що реально співіснують у коді, як стан і події доходять
до UI, і поділ на composable Root/Screen. Навігація має власну нотатку:
[[Navigation_UA|Навігація]].

## Контракт MVI

Три маркерні інтерфейси задають словник, по одному на файл у `presentation/base/mvi/`
(`UiState.kt:3`, `UiIntent.kt:3`, `UiEffect.kt:3`):

```kotlin
interface UiState
interface UiIntent
interface UiEffect
```

- State - незмінний знімок, який рендерить UI. По одному `data class` на екран у
  пакеті цього екрана (наприклад `ChatUiState`, `presentation/screens/chat/ChatUiState.kt:16`),
  анотований `@Immutable`, щоб Compose міг пропустити рекомпозицію, коли посилання не змінилося.
- Intent - дія користувача, яку UI надсилає донизу. Sealed-інтерфейс на екран
  поруч зі станом (наприклад `ChatIntent`, `presentation/screens/chat/ChatIntent.kt:8`).
- Effect - одноразовий побічний ефект, який ViewModel надсилає нагору (показати помилку, тригер
  навігації). Sealed-інтерфейс на екран у тому самому пакеті (наприклад `ChatEffect`,
  `presentation/screens/chat/ChatEffect.kt:6`). Після рефакторингу розкладки файлів колишніх тек
  `presentation/state/`, `intent/`, `effect/` і `viewmodel/` більше немає; стан, intent, effect,
  ViewModel і composable кожного екрана лежать разом у `presentation/screens/<screen>/`.

State - це те, чим речі Є; Effect - те, що щойно СТАЛОСЯ. Різниця важлива для коректності: state
конфлюється й повторюється новим підписникам, а ефекти мають спрацювати рівно один раз. Саме тому
нижче вони використовують різні типи Flow.

## Базовий клас `MviViewModel`

`presentation/base/mvi/MviViewModel.kt:15` (тіла з одного виразу зведено в один рядок):

```kotlin
abstract class MviViewModel<S : UiState, I : UiIntent, E : UiEffect>(
    initialState: S,
) : ViewModel() {

    private val _state = MutableStateFlow(initialState)
    val state: StateFlow<S> = _state.asStateFlow()

    private val _effect = MutableSharedFlow<E>(
        replay = 0,
        extraBufferCapacity = 16,
        onBufferOverflow = BufferOverflow.DROP_OLDEST,
    )
    val effect: SharedFlow<E> = _effect.asSharedFlow()

    fun dispatch(intent: I) { handleIntent(intent) }
    protected abstract fun handleIntent(intent: I)

    protected fun setState(reducer: S.() -> S) { _state.update(reducer) }
    protected val currentState: S get() = _state.value
    protected fun emitEffect(effect: E) { viewModelScope.launch { _effect.emit(effect) } }
}
```

Чому саме такий вибір:

- `state` - це `StateFlow`, а не голий `Flow`. `StateFlow` гарячий, конфлюючий і завжди тримає
  поточне значення, тож перший кадр екрана одразу має що рендерити, а зміна конфігурації
  повторно збирає останній стан без пропуску. Він відкритий лише для читання через
  `asStateFlow()`; писати може лише ViewModel через `setState`.
- `setState` приймає редьюсер `S.() -> S` і викликає `_state.update`. `update` застосовує
  трансформацію атомарно (цикл compare-and-set), а це саме те, що потрібно, коли кілька корутин
  викликають `setState { copy(...) }` одночасно - жодних втрачених оновлень.
- `effect` - це `SharedFlow` з `replay = 0`. Ефекти не мають повторно доставлятися новому
  підписнику після повороту (тост з помилкою не повинен спрацювати знову), тож replay нульовий.
  `extraBufferCapacity = 16` з `DROP_OLDEST` означає, що `emitEffect` ніколи не призупиняє
  викликача і не викликає взаємоблокування, якщо ще ніхто не збирає; за сплеску відкидається
  найстаріший ефект, а не блокується. Це стандартна конфігурація "події, а не стан".
- `emitEffect` запускається на `viewModelScope`, бо `SharedFlow.emit` - це `suspend`; цей запуск
  прив'язує випромінювання до життєвого циклу ViewModel.

Обрізаний "родич" є в `presentation/base/mvi/StateViewModel.kt:15` (`StateViewModel<S, E>`):
ідентичний механізм state/effect, але без типу `Intent` і без `dispatch`. Він для екранів, що
викликають методи ViewModel напряму, а не пропускають кожну дію через sealed-intent
(`EditChatViewModel`, `EditContactDataViewModel`, `SearchUsersViewModel`).

## Два стилі ViewModel, що реально використовуються

Кодова база не нав'язує один патерн. Обидва присутні й обидва легітимні:

1. MVI з маршрутизацією intent (розширює `MviViewModel`). Приклади: `ChatViewModel`
   (`presentation/screens/chat/ChatViewModel.kt:63`), `AuthViewModel`
   (`presentation/screens/auth/AuthViewModel.kt:37`), `ChangePhoneViewModel`,
   `DeleteAccountViewModel`, `EditProfileViewModel`. UI лише викликає `dispatch(intent)`;
   `handleIntent` - це один `when` по sealed-intent
   (`presentation/screens/chat/ChatViewModel.kt:190`). Його перші два рядки питають у
   `requiredFlag(intent)`, який feature flag потрібен intent-у, і повертаються раніше, коли
   `featureFlags.refuses(flag)` - перемикач застосовується тут, нижче за UI (див. [[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]]).
2. Метод-і-StateFlow (розширює звичайний `ViewModel` / `AndroidViewModel`). Приклад:
   `ChatsViewModel` (`presentation/screens/chats/ChatsViewModel.kt:37`) надає
   `MutableStateFlow(...).asStateFlow()` плюс публічні методи (`deleteConversation`,
   `createConversation`, `refresh`). `CallViewModel`
   (`presentation/screens/call/CallViewModel.kt:40`) робить те саме і додатково використовує
   `Channel<CallExit>`, відкритий як `receiveAsFlow()`, для одноразових навігаційних подій замість
   `SharedFlow` ефектів. `ClipPlayerViewModel` (`presentation/components/common/ClipPlayerViewModel.kt`),
   що стоїть за голосовим міні-плеєром на всіх вкладках, - найтонший випадок: його `state` - це
   прямо переданий `ClipPlaybackController.state` рівня застосунку, а методи переадресовуються
   контролеру (див. [[Voice Messages_UA|Голосові повідомлення]]). Новіші ViewModel профілю, вигляду,
   вибору медіа та профілю чату (наприклад `AppearanceViewModel`, `ChatUserProfileViewModel`) теж цього
   другого стилю; кожен публічний метод починається з перевірки `featureFlags.refuses(...)`.

Компроміс: стиль 1 дає єдину, тестовну точку входу й вичерпний перелік дій; стиль 2 легший для
екранів із небагатьма діями або де `Channel` природніше лягає на "навігувати один раз". Обидва
відкривають нагору незмінний `StateFlow`, тож код збирання в UI однаковий.

## Як UI збирає стан і події

Стан збирається через `collectAsStateWithLifecycle`, який збирає лише поки життєвий цикл
щонайменше STARTED, тож екран у фоні припиняє роботу з рекомпозицією. Реальне місце виклику в
навігації (`presentation/navigation/graph/AppNavigation.kt:31`):

```kotlin
val authViewModel: AuthViewModel = hiltViewModel()
val authState by authViewModel.state.collectAsStateWithLifecycle()
```

Ефекти збираються через проєктний хелпер `ObserveAsEvents`
(`presentation/base/ObserveAsEvents.kt:13`):

```kotlin
@Composable
fun <T> ObserveAsEvents(events: Flow<T>, onEvent: (T) -> Unit) {
    val lifecycleOwner = LocalLifecycleOwner.current
    LaunchedEffect(events, lifecycleOwner.lifecycle) {
        lifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
            withContext(Dispatchers.Main.immediate) {
                events.collect(onEvent)
            }
        }
    }
}
```

Мета - "виконати цей обробник один раз на кожну випромінену подію, лише поки видно, у головному
потоці". `repeatOnLifecycle(STARTED)` перезапускає збирання, коли екран повертається на передній
план, і скасовує його, коли той іде, що не дає фоновому екрану навігувати чи показувати тости.
`Dispatchers.Main.immediate` уникає зайвого перерозподілу, коли ми вже в main, тож ефект, що
запускає навігацію, виконується синхронно в тому ж кадрі. `ChatScreenWithNav` використовує цей
хелпер (`presentation/screens/chat/ChatScreenWithNav.kt:81`).

## Редукування стану: патерн `setState { copy(...) }`

Кожна мутація стану проходить через редьюсер. З `ChatViewModel`
(`presentation/screens/chat/ChatViewModel.kt:519`):

```kotlin
setState {
    copy(
        typingUsernames = typingUserIds.toList(),
        isPartnerTyping = typingUserIds.isNotEmpty(),
    )
}
```

Оскільки `ChatUiState` - незмінний `data class`, `copy` створює новий екземпляр; `StateFlow`
порівнює за рівністю і випромінює лише коли щось справді змінилося, тож ідентичні оновлення не
спричиняють рекомпозицію. Читання всередині ViewModel використовує `currentState`
(`presentation/screens/chat/ChatViewModel.kt:240`) для останнього знімка без збирання.

## UiText: рядки помилок/повідомлень, що доживають до часу composе

`presentation/base/UiText.kt:10` - це sealed-тип з `DynamicString` (рядок часу виконання) та
`StringResource(@StringRes id, vararg args)` (id ресурсу, що резолвиться пізніше). ViewModel
кладуть у стан `UiText?` замість вже резолвленого `String`, а composable резолвить його через
`asString()` під час рендеру. Це тримає ViewModel вільним від `Context` і дає резолвленому тексту
слідувати за поточною локаллю/конфігурацією. Хелпери-розширення `String.toUiText()` і
`Int.toUiText(vararg)` (`presentation/base/UiText.kt:30`) використовуються скрізь у ViewModel.

## Смерть процесу і `SavedStateHandle`

ViewModel зі скоупом екрана читають свої навігаційні аргументи з `SavedStateHandle`, який
відновлюється після смерті процесу. `ChatViewModel` (`presentation/screens/chat/ChatViewModel.kt:87`):

```kotlin
val conversationId: String = savedStateHandle.get<String>("conversationId") ?: ""
private val partnerId: String = savedStateHandle.get<String>("partnerId") ?: ""
```

Ці ключі заповнює навігаційний шар через обгортку `ProvideNavArgs` - див.
[[Navigation_UA|Навігація]], як типізований маршрут стає аргументами `SavedStateHandle`.

## Поділ на composable Root/Screen

Екрани структуровані так, що прив'язаний до ViewModel "root" тонкий, а візуальний "screen" - чиста
функція від стану. Root отримує ViewModel через `hiltViewModel()`, збирає стан через
`collectAsStateWithLifecycle`, під'єднує `ObserveAsEvents` для ефектів і передає `state` плюс
`onIntent`/колбеки в stateless-composable екрана. Екран чату дотримується цього у файлах
`presentation/screens/chat/`: `ChatScreenWithNav` отримує ViewModel (`ChatScreenWithNav.kt:68`),
збирає стан (`:76`) і ефекти (`:81`) та передає стан плюс колбеки `ChatIntent` у
`ChatScreenContent`, що рендерить `ChatUiState`. У пакеті чату на 0.2.2.4 немає власного `@Preview`
(колишнього файлу `ChatScreenPreview` більше немає); інші екрани мають такий самий поділ
`Screen` / `ScreenContent` і тримають свої `@Preview` у файлі `ScreenContent` (наприклад
автентифікація, налаштування, видалення акаунта, профіль). Те, що stateless-екран є чистою функцією від
`UiState`, і робить можливими `@Preview` та UI-тести без живого ViewModel.

## Розглянуті альтернативи

- Єдиний обов'язковий MVI-базис для кожного екрана. Не зроблено - код навмисне тримає легший
  стиль `StateFlow`+методи для простих екранів. Задокументовано тут як навмисне співіснування, а
  не непослідовність.
- LiveData для стану. Відкинуто на користь `StateFlow`/`collectAsStateWithLifecycle`, що є
  поточним Compose-нативним ідіомом і не потребує власної обгортки з урахуванням `Lifecycle`.
- `Channel` для ефектів усюди. Базовий клас використовує `SharedFlow(replay=0)`; `CallViewModel`
  використовує `Channel` для своєї події виходу. Обидва гарантують одноразову доставку;
  `SharedFlow` пасує потоку ефектів "0..n підписників", `Channel` - строгому односпоживчому
  сигналу навігації.

## Пов'язане

[[Async and Reactivity_UA|Асинхронність і реактивність]] - [[Navigation_UA|Навігація]] -
[[Domain Layer_UA|Доменний шар]] - [[Chat and Messaging_UA|Чат і обмін повідомленнями]] -
[[Authentication_UA|Автентифікація]] - [[Voice Messages_UA|Голосові повідомлення]] -
[[Architecture Overview_UA|Огляд архітектури]]
