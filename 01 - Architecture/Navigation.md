# Navigation

## What and version

Navigation 3 (`androidx.navigation3` `1.1.3`, `gradle/libs.versions.toml:23`) plus
`androidx.lifecycle:lifecycle-viewmodel-navigation3` for per-entry ViewModel stores. Routes are
`@Serializable` `NavKey`s (kotlinx-serialization). This is the newer back-stack-as-state model,
not the older `NavHost`/`NavController` graph.

## Typed routes

`presentation/navigation/Routes.kt` declares every destination as a serializable `NavKey`.
Tab roots and settings pages are `data object`s (`AuthRoute`, `ChatsRoute`, `CallsRoute`,
`SettingsRoute`, `PrivacyRoute`, ...); destinations with arguments are `data class`es
(`Routes.kt:55`):

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

Arguments are strongly typed - no string keys at the call site, no manual bundle packing when
navigating.

## The back stack and NavDisplay

`presentation/navigation/navigation.kt` is the whole navigation surface. `AppNavigation`
(`navigation.kt:77`) switches on auth state (collected via `collectAsStateWithLifecycle`,
`navigation.kt:82`) between `AuthDisplay` and `MainDisplay`.

The back stack is state: `rememberNavBackStack(ChatsRoute)` (`navigation.kt:130`). Navigating is
just mutating that list - `backStack.add(ChatRoute(...))` to push, `backStack.removeLastOrNull()`
to pop. `NavDisplay` (`navigation.kt:205`) renders the top entry and animates transitions:

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

Two decorators matter:
- `rememberSaveableStateHolderNavEntryDecorator` preserves each entry's `rememberSaveable`
  state across navigation.
- `rememberViewModelStoreNavEntryDecorator` gives each back-stack entry its own
  `ViewModelStore`, so `hiltViewModel()` inside an entry yields a ViewModel scoped to that
  entry and cleared when it is popped.

`entryProvider` maps each `NavKey` type to its composable via `entry<T> { }` (`navigation.kt:220`
onward). Tab switching (`navigateToTab`, `navigation.kt:173`) pops non-root entries then swaps
the root, so tabs behave like a single-level switcher rather than stacking.

## Passing nav args into a ViewModel's SavedStateHandle

Navigation 3 entries do not automatically feed a `@HiltViewModel`'s `SavedStateHandle`. The app
bridges this with `ProvideNavArgs` (`navigation.kt:451`), which wraps the current
`ViewModelStoreOwner` in a `NavArgsViewModelStoreOwner` (`navigation.kt:464`) that injects a
`Bundle` as the `DEFAULT_ARGS_KEY` of the ViewModel's `CreationExtras`:

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

Downstream, `ChatViewModel` reads those exact keys from its `SavedStateHandle`
(`presentation/viewmodel/ChatViewModel.kt:62`). `CallViewModel` and `ChatUserProfileViewModel`
receive their args the same way. Because the values are serialized route fields flowing through
`SavedStateHandle`, they survive process death. See [[Presentation Layer (MVI)]] for the
process-death angle.

## Call and search overlays on top of navigation

`MainDisplay` layers non-navigation UI over the `NavDisplay` in the same `Box`:
- the `FloatingTabBar` for tab roots (`navigation.kt:394`),
- the `IncomingCallBar`, docked `ActiveCallBar`, and `CallBarOverlay` for the active call,
  shown/hidden based on `ActiveCallHolder.state` and `CallBarPresenter.mode`
  (`navigation.kt:143`, `:412`),
- the `SearchResultsOverlay` when searching a tab (`navigation.kt:383`).

The call bars live at the navigation root (not inside a screen) so they persist across screen
changes while a call is active. See [[Calls (Voice and Video)]].

## Shared-element transitions

`MainDisplay` wraps the `NavDisplay` in a `SharedTransitionLayout` and provides
`LocalSharedTransitionScope` / `LocalNavAnimatedVisibilityScope` (`navigation.kt:203`,
`ProvideNavAnimatedScope`, `:442`), so avatars and profile elements animate between the
conversation list, chat, and profile screens.

## Why this approach

- Back stack as observable state fits the Compose mental model - navigation is `add`/`remove`
  on a remembered list, diffable and testable.
- Typed `NavKey`s remove string-route bugs and make arguments compiler-checked.
- Per-entry ViewModel stores via the decorator give correct ViewModel lifecycles without a
  separate nav-graph ViewModel scope.

Trade-off: feeding `SavedStateHandle` needs the custom `ProvideNavArgs`/`NavArgsViewModelStoreOwner`
bridge (`navigation.kt:451`-`:481`), which is more plumbing than the old Navigation-Compose
`SavedStateHandle` wiring. Rationale for choosing Navigation 3 over Navigation-Compose was not
captured in source; documented as the current state.

## Related

[[Presentation Layer (MVI)]] - [[Calls (Voice and Video)]] - [[Architecture Overview]] -
[[Tech Stack and Versions]]
