# Navigation

## What and version

Navigation 3 (`androidx.navigation3` `1.1.3`, `gradle/libs.versions.toml:23`) plus
`androidx.lifecycle:lifecycle-viewmodel-navigation3` for per-entry ViewModel stores. Routes are
`@Serializable` `NavKey`s (kotlinx-serialization). This is the newer back-stack-as-state model,
not the older `NavHost`/`NavController` graph.

## Typed routes

`presentation/navigation/Routes.kt` declares every destination as a serializable `NavKey`.
Tab roots and settings pages are `data object`s (`AuthRoute`, `ChatsRoute`, `CallsRoute`,
`SettingsRoute`, `PrivacyRoute`, `DeleteAccountRoute`, ...); destinations with arguments are
`data class`es (`Routes.kt:55`, `:73`):

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

## The session gate

The navigation surface lives in `presentation/navigation/graph/`: `AppNavigation.kt`,
`AuthDisplay.kt`, `MainDisplay.kt`, `ProvideNavArgs.kt` and `ProvideNavAnimatedScope.kt`.
`AppNavigation` (`AppNavigation.kt:23`) collects `AuthViewModel.state` and switches on
`authState.session`, a `SessionState` (`domain/model/SessionState.kt:3`):

```kotlin
when (authState.session) {
    is SessionState.Active -> MainDisplay(
        authViewModel = authViewModel,
        pendingRoute = pendingRoute,
        onRouteConsumed = onRouteConsumed,
    )

    is SessionState.SignedOut -> AuthDisplay(authViewModel = authViewModel)

    SessionState.Resolving -> Unit
}
```

It used to switch on the bare Firebase sign-in flag. Signing in with a phone that has no
Firestore profile then mounted `MainDisplay` before the profile check ran and bounced back to a
freshly composed `AuthDisplay`, losing the phone number and OTP the user had typed.
`ObserveSessionStateUseCase` (`domain/usecase/auth/ObserveSessionStateUseCase.kt:20`) now reaches
`Active` only once the profile is confirmed:

- no Firebase user -> `SignedOut()`;
- a register or login is in flight (`IAuthFlowGate.isBusy`, `InMemoryAuthFlowGate` in
  `data/service/`) -> `Resolving`, so the gate does not interfere mid-flow;
- otherwise it follows `userRepository.observeRemoteUser(uid)`: a profile marked `deleted` ->
  `SignedOut(ACCOUNT_DELETED)`; a profile present -> `Active(user)`; a server-confirmed absence
  (`snapshot.authoritative`) -> `SignedOut(PROFILE_MISSING)`; a cache miss or an error ->
  `Active` with the Firebase identity, so an offline start never forces a logout.

`AuthViewModel.observeSession` (`presentation/viewmodel/AuthViewModel.kt:78`) stores the session
in state and, for a `SignedOut` with a reason, runs `endInvalidSession` (`:87`): disconnect
presence, run the logout use case and show the matching error. The same listener signs this
device out when the account is deleted elsewhere. `Resolving` renders nothing over the wallpaper.
See [[Authentication]].

## The back stack and NavDisplay

The back stack is state: `rememberNavBackStack(ChatsRoute)` (`MainDisplay.kt:125`). Navigating is
just mutating that list - `backStack.add(ChatRoute(...))` to push, `backStack.removeLastOrNull()`
to pop. `NavDisplay` (`MainDisplay.kt:267`) renders the top entry and animates transitions:

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

`entryProvider` maps each `NavKey` type to its composable via `entry<T> { }` (`MainDisplay.kt:286`
onward). Tab switching (`navigateToTab`, `MainDisplay.kt:223`) pops non-root entries then swaps
the root, so tabs behave like a single-level switcher rather than stacking. Account deletion is
reached from Privacy: `PrivacyScreen(onDeleteAccount = { backStack.add(DeleteAccountRoute) })`
(`MainDisplay.kt:343`) and `entry<DeleteAccountRoute>` (`:346`).

## Passing nav args into a ViewModel's SavedStateHandle

Navigation 3 entries do not automatically feed a `@HiltViewModel`'s `SavedStateHandle`. The app
bridges this with `ProvideNavArgs` (`ProvideNavArgs.kt:19`), which wraps the current
`ViewModelStoreOwner` in a `NavArgsViewModelStoreOwner` (`ProvideNavArgs.kt:32`) that injects a
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
(`presentation/viewmodel/ChatViewModel.kt:80`). `CallViewModel` and `ChatUserProfileViewModel`
receive their args the same way. Because the values are serialized route fields flowing through
`SavedStateHandle`, they survive process death. See [[Presentation Layer (MVI)]] for the
process-death angle.

## Call and search overlays on top of navigation

`MainDisplay` layers non-navigation UI over the `NavDisplay` in the same `Box`:
- the `FloatingTabBar` for tab roots (`MainDisplay.kt:564`),
- the `IncomingCallBar`, docked `ActiveCallBar`, and `CallBarOverlay` for the active call,
  shown/hidden based on `ActiveCallHolder.state` and `CallBarPresenter.mode`
  (`MainDisplay.kt:147`, `:587`),
- the `SearchResultsOverlay` when searching a tab (`MainDisplay.kt:521`).

The call bars live at the navigation root (not inside a screen) so they persist across screen
changes while a call is active. See [[Calls (Voice and Video)]].

## Shared-element transitions

`MainDisplay` wraps the `NavDisplay` in a `SharedTransitionLayout` and provides
`LocalSharedTransitionScope` / `LocalNavAnimatedVisibilityScope` (`MainDisplay.kt:264`,
`ProvideNavAnimatedScope`, `ProvideNavAnimatedScope.kt:10`), so avatars and profile elements animate between the
conversation list, chat, and profile screens.

## Why this approach

- Back stack as observable state fits the Compose mental model - navigation is `add`/`remove`
  on a remembered list, diffable and testable.
- Typed `NavKey`s remove string-route bugs and make arguments compiler-checked.
- Per-entry ViewModel stores via the decorator give correct ViewModel lifecycles without a
  separate nav-graph ViewModel scope.
- Gating on a resolved `SessionState` instead of the Firebase flag means the main UI only mounts
  for a user whose profile exists, and an in-progress sign-in is never torn down by the gate.

Trade-off: feeding `SavedStateHandle` needs the custom `ProvideNavArgs`/`NavArgsViewModelStoreOwner`
bridge (`ProvideNavArgs.kt`, 49 lines), which is more plumbing than the old Navigation-Compose
`SavedStateHandle` wiring. Rationale for choosing Navigation 3 over Navigation-Compose was not
captured in source; documented as the current state.

## Related

[[Presentation Layer (MVI)]] - [[Calls (Voice and Video)]] - [[Architecture Overview]] -
[[Authentication]] - [[Tech Stack and Versions]]
