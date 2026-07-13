# Presentation Layer (MVI)

The presentation layer is Jetpack Compose plus a Model-View-Intent contract. This note covers
the MVI base class, the two ViewModel styles that actually coexist in the code, how state and
events reach the UI, and the Root/Screen composable split. Navigation has its own note:
[[Navigation]].

## The MVI contract

Three marker interfaces define the vocabulary (`presentation/base/MviViewModel.kt:15`):

```kotlin
interface UiState
interface UiIntent
interface UiEffect
```

- State - an immutable snapshot the UI renders. One `data class` per screen in
  `presentation/state/` (for example `ChatUiState`, `presentation/state/ChatUiState.kt:16`),
  annotated `@Immutable` so Compose can skip recomposition when the reference is unchanged.
- Intent - a user action the UI sends down. A sealed interface per screen in
  `presentation/intent/` (for example `ChatIntent`, `presentation/intent/ChatIntent.kt:8`).
- Effect - a one-shot side effect the ViewModel sends up (show error, navigation trigger).
  A sealed interface per screen in `presentation/effect/` (for example `ChatEffect`,
  `presentation/effect/ChatEffect.kt:6`).

State is what things ARE; Effect is what just HAPPENED. The distinction matters for
correctness: state is conflated and replayed to new collectors, effects must fire exactly
once. That is why they use different Flow types below.

## The `MviViewModel` base class

`presentation/base/MviViewModel.kt:21`:

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

Why each choice:

- `state` is a `StateFlow` and not a bare `Flow`. A `StateFlow` is hot, conflated, and always
  holds a current value, so the first frame of a screen has something to render immediately
  and a config change re-collects the latest state with no gap. It is exposed read-only via
  `asStateFlow()`; only the ViewModel can write, through `setState`.
- `setState` takes a reducer `S.() -> S` and calls `_state.update`. `update` applies the
  transform atomically (compare-and-set loop), which is what you want when several coroutines
  call `setState { copy(...) }` concurrently - no lost updates.
- `effect` is a `SharedFlow` with `replay = 0`. Effects must not be re-delivered to a new
  collector after a rotation (you do not want the error toast to fire again), so replay is
  zero. `extraBufferCapacity = 16` with `DROP_OLDEST` means `emitEffect` never suspends the
  caller and never deadlocks if nothing is collecting yet; under a burst it drops the oldest
  effect rather than blocking. This is the standard "events, not state" configuration.
- `emitEffect` launches on `viewModelScope` because `SharedFlow.emit` is `suspend`; the launch
  ties the emission to the ViewModel lifecycle.

A trimmed sibling exists at `presentation/base/StateViewModel.kt:15` (`StateViewModel<S, E>`):
identical state/effect machinery but no `Intent` type and no `dispatch`. It is for screens
that call ViewModel methods directly rather than routing every action through a sealed intent.

## Two ViewModel styles actually in use

The codebase does not force one pattern. Both are present and both are legitimate:

1. Intent-routed MVI (extends `MviViewModel`). Example: `ChatViewModel`
   (`presentation/viewmodel/ChatViewModel.kt:60`) and `AuthViewModel`
   (`presentation/viewmodel/AuthViewModel.kt:39`). The UI only ever calls `dispatch(intent)`;
   `handleIntent` is a single `when` over the sealed intent
   (`presentation/viewmodel/ChatViewModel.kt:137`).
2. Method-and-StateFlow (extends plain `ViewModel` / `AndroidViewModel`). Example:
   `ConversationsViewModel` (`presentation/viewmodel/ConversationsViewModel.kt:38`) exposes a
   `MutableStateFlow(...).asStateFlow()` plus public methods (`deleteConversation`,
   `createConversation`, `refresh`). `CallViewModel`
   (`presentation/viewmodel/CallViewModel.kt:48`) does the same and additionally uses a
   `Channel<CallExit>` exposed as `receiveAsFlow()` for one-shot navigation events instead of
   a `SharedFlow` of effects.

The trade-off: style 1 gives a single, testable entry point and an exhaustive action list;
style 2 is lighter for screens with few actions or where a `Channel` maps more naturally to
"navigate once". Both expose an immutable `StateFlow` upward, so the UI collection code is the
same either way.

## How the UI collects state and events

State is collected with `collectAsStateWithLifecycle`, which only collects while the lifecycle
is at least STARTED, so a backgrounded screen stops doing recomposition work. Real call site
in navigation (`presentation/navigation/navigation.kt:82`):

```kotlin
val authViewModel: AuthViewModel = hiltViewModel()
val authState by authViewModel.state.collectAsStateWithLifecycle()
```

Effects are collected with the project helper `ObserveAsEvents`
(`presentation/base/ObserveAsEvents.kt:12`):

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

The goal is "run this handler once per emitted event, only while visible, on the main thread".
`repeatOnLifecycle(STARTED)` restarts the collection when the screen returns to the
foreground and cancels it when it leaves, which prevents a backgrounded screen from
navigating or toasting. `Dispatchers.Main.immediate` avoids an unnecessary re-dispatch when
already on main, so an effect that triggers navigation runs synchronously in the same frame.
`ChatScreen` imports and uses this helper (`presentation/screens/ChatScreen.kt:102`).

## Reducing state: the `setState { copy(...) }` pattern

Every state mutation goes through the reducer. From `ChatViewModel`
(`presentation/viewmodel/ChatViewModel.kt:314`):

```kotlin
setState {
    copy(
        typingUsernames = typingUserIds.toList(),
        isPartnerTyping = typingUserIds.isNotEmpty(),
    )
}
```

Because `ChatUiState` is an immutable `data class`, `copy` produces a new instance; `StateFlow`
compares by equality and only emits when something actually changed, so identical updates do
not cause recomposition. Reads inside the ViewModel use `currentState`
(`presentation/viewmodel/ChatViewModel.kt:161`) for the latest snapshot without collecting.

## UiText: error/message strings that survive to compose time

`presentation/base/UiText.kt:10` is a sealed type with `DynamicString` (a runtime string) and
`StringResource(@StringRes id, vararg args)` (a resource id resolved later). ViewModels put a
`UiText?` in state (for example `ChatUiState.error`) instead of a resolved `String`, then the
composable resolves it with `asString()` at render time. This keeps the ViewModel free of a
`Context` and lets the resolved text follow the current locale/config. Extension helpers
`String.toUiText()` and `Int.toUiText(vararg)` (`presentation/base/UiText.kt:30`) are used
throughout the ViewModels.

## Process death and `SavedStateHandle`

Screen-scoped ViewModels read their navigation arguments from `SavedStateHandle`, which is
restored across process death. `ChatViewModel` (`presentation/viewmodel/ChatViewModel.kt:62`):

```kotlin
val conversationId: String = savedStateHandle.get<String>("conversationId") ?: ""
private val partnerId: String = savedStateHandle.get<String>("partnerId") ?: ""
```

Those keys are populated by the navigation layer's `ProvideNavArgs` wrapper - see
[[Navigation]] for how a typed route becomes `SavedStateHandle` args.

## The Root/Screen composable split

Screens are structured so the ViewModel-bound "root" is thin and the visual "screen" is a pure
function of state. The root obtains the ViewModel with `hiltViewModel()`, collects state with
`collectAsStateWithLifecycle`, wires `ObserveAsEvents` for effects, and passes `state` plus
`onIntent`/callbacks into a stateless screen composable. `ChatScreen.kt` follows this: it
imports `hiltViewModel`, `collectAsStateWithLifecycle`, `ObserveAsEvents`, `ChatIntent`,
`ChatEffect`, and `ChatUiState` (`presentation/screens/ChatScreen.kt:95`), and the visual
screen renders `ChatUiState` while forwarding user actions as `ChatIntent`s. Keeping the
stateless screen a pure function of `UiState` is what makes `@Preview` and UI tests possible
without a live ViewModel.

## Alternatives considered

- A single mandated MVI base for every screen. Not done - the code deliberately keeps the
  lighter `StateFlow`+methods style for simple screens. Documented here as an intentional
  coexistence rather than an inconsistency.
- LiveData for state. Rejected in favour of `StateFlow`/`collectAsStateWithLifecycle`, which
  is the current Compose-native idiom and needs no `Lifecycle`-aware wrapper of its own.
- A `Channel` for effects everywhere. The base class uses `SharedFlow(replay=0)`; `CallViewModel`
  uses a `Channel` for its exit event. Both guarantee single delivery; `SharedFlow` fits the
  "0..n subscribers" effect stream, `Channel` fits the strict single-consumer nav signal.

## Related

[[Async and Reactivity]] - [[Navigation]] - [[Domain Layer]] - [[Chat and Messaging]] -
[[Authentication]] - [[Architecture Overview]]
