# File Layout Refactor - `refactor/readability`

> Historical. Executed on 2026-08-11 via PRs #3 and #4 into `develop`. Single-use types stayed in
> their declaring files, so some files listed below were never created. Source paths reflect the
> plan, not the current tree.

Date: 2026-08-11
Status: **Shipped** - PR https://github.com/DerekBurned/Messenger/pull/3 into `develop`
Branch: `refactor/readability`

## Amendment: what actually shipped

Mid-implementation the rule changed. **A type is only extracted if it is referenced outside the file
that declares it.** Single-use types stay where they are.

This dropped roughly 15 of the 51 targets and, importantly, **eliminated every `private` to
`internal` promotion** listed in the Visibility section below - that section is superseded and no
longer describes the branch. The result is a 100% pure move refactor with zero API surface change.

Types that stayed put under the new rule: `CallPartnerCache`, `CaptureMode`, `GridTile`,
`StorageSegment`, `CallPartnerUi`, `EncryptedBlob`, `Spacing`, `CallsFilter`, `SafetyNumberState`,
`NotificationActionEntryPoint`, `RemoteMediaItemDto`, `FcmEntryPoint`, `TransferCancelledException`,
`ChatRow`, `CallPermissions`, `NavArgsViewModelStoreOwner`, `formatSeconds`, `mediaReadPermissions`,
`sendCallAction`, `toNavRoute`.

`PipForm` is the single deliberate exception - extracted and promoted to `internal` by explicit
request, since it was the file that prompted the whole effort.

`MessageContextMenuItem` was also extracted despite looking single-use: it is public API constructed
by `LongPressMessage` and held by `MessageContextMenuState`, and appeared single-use only because it
shared a file with its consumer, which was itself moving.

Two additions beyond the original mapping: `StateViewModel` moved to `base/mvi/` alongside
`MviViewModel` (same family), and two files were renamed to match their contents -
`MessageEnvelope.kt` to `CryptoEncoding.kt` and `MessageDTO.kt` to `RemoteMessageDto.kt`.

Final verification: `assembleDebug` succeeded, 85 unit tests passed with 0 failures,
`default.json` unchanged, no comments added.

## Problem

Opening a Kotlin file in this project often reveals several unrelated top-level declarations at once. A file named after one type holds three others. This makes navigation, debugging, and code review harder than it needs to be: the filename stops predicting the contents, and "go to file" stops being useful.

Two shapes of the problem:

1. A file holding several top-level types (`ObxEntities.kt` holds 8 entities).
2. A file whose main purpose is a screen or component, with a supporting type buried inside it (`PipCallSurface.kt` holds a `sealed interface PipForm` among its composables).

A third, related problem: three composable files have grown past 500 lines, so even with one type per file they are hard to navigate.

## Goal

After this branch, opening any file in `app/src/main` shows one subject. Filenames predict contents.

## Non-goals

Explicitly out of scope, to keep the diff reviewable and the risk at zero:

- Renaming variables, functions, or parameters.
- Deleting dead code.
- Adding comments or KDoc (project rule: zero comments).
- Any logic change whatsoever.
- `CallForegroundService.kt` - see Deferred below.
- `Routes.kt` - see Deliberate exclusions below.
- Test sources under `app/src/test`.

## Scope

51 files.

| Category | Count |
|---|---|
| Files with 2 or more top-level types | 35 |
| Files where a single type hides among top-level functions | 17 |
| Union of the two, minus the excluded `Routes.kt` | **51** |

Three of those 51 are additionally over 500 lines and get a second treatment (one file per composable, not just type extraction): `ChatScreen.kt`, `CallScreen.kt`, `navigation.kt`.

Every change is a **pure move**. Declarations are relocated verbatim; imports and package statements are the only edited lines. Correctness is proved by compilation.

## Deliberate exclusions

**`presentation/navigation/Routes.kt`** (20 declarations). Every declaration is a `NavKey` route object of 2 to 5 lines. The file is a route table and reads as one; splitting it into 20 files would make navigation worse, not better. Kept as is.

**`data/remote/call/CallForegroundService.kt`** (1003 lines). This is a single class, not a file with several classes. It carries roughly 60 private methods and 20 mutable fields (`tickerJob`, `ringTimeoutJob`, `hasAudioFocus`, `missRecorded`, `peerWasPresent`, and others) whose lifecycles are interdependent. Splitting it means extracting collaborators and rewiring shared mutable state - a behavioural refactor with real regression risk on live calls, which the unit test suite would not catch. Deferred to its own branch where it can be tested against real call flows.

**`presentation/base/UiText.kt`**, **`data/remote/dto/CallSignalDto.kt`**. Each holds one type plus its own mapper or formatter functions. Cohesive; the functions exist only to serve the type. Kept as is.

## Conventions

### Folder rule

Extracted types are grouped by **cohesive area**, not one folder per source file. A `model/` subpackage collects the supporting types of a component group; a named subpackage collects a screen's composables.

This yields roughly 18 new packages. The rejected alternative - one folder per split source file - would have produced roughly 45 packages, most holding exactly 2 files, with `data/crypto/` alone gaining 6.

### Visibility

Eight declarations are `private` at file level. Kotlin's file-private visibility cannot survive a move to a new file, so these are promoted to `internal`:

| Declaration | Current file |
|---|---|
| `CallPartnerCache` | `components/call/ActiveCallBar.kt` |
| `PipForm` | `components/call/PipCallSurface.kt` |
| `CaptureMode` | `components/media/CameraCaptureScreen.kt` |
| `NavArgsViewModelStoreOwner` | `navigation/navigation.kt` |
| `CallPermissions` | `screens/CallScreen.kt` |
| `ChatRow` | `screens/ChatScreen.kt` |
| `GridTile` | `screens/settings/AppearanceScreen.kt` |
| `StorageSegment` | `screens/settings/DataStorageScreen.kt` |

This is the one place the refactor is not purely a move. It widens visibility from file to module. No new call sites are added.

### ObjectBox exception

`data/local/obx/` files are split but **stay in their current package**. Moving entities to a new package would relocate the generated `MyObjectBox` class and rewrite `app/objectbox-models/default.json`. Given this project's history with `DbSchemaException` on schema changes, that risk is not worth taking for a cosmetic gain.

`ObxConverters.kt` currently shares one file-private `private val gson = Gson()` across its 5 converters. Splitting the file would give each converter its own `Gson` instance. To preserve current behaviour exactly, the instance is hoisted to `internal val obxGson` in `data/local/obx/ObxGson.kt`, keeping a single shared instance.

## File mapping

### Domain

| Target package | Contents | Source |
|---|---|---|
| `domain/model/call/` | `CallType`, `CallSignal`, `CallStatus`, `CallConnectionState` | `Message.kt`, `CallSignal.kt`, `ICallService.kt` |
| `domain/model/presence/` | `UserPresence`, `PresenceState` | `PresenceState.kt` |
| `domain/model/sync/` (flat) | `SyncAction`, `SyncEntityType` | `SyncQueueConstants.kt` |
| `domain/service/` (flat) | `ICallService`, `CallEventListener` | `ICallService.kt` |

`domain/model/Message.kt` keeps the `Message` sealed class and stays in place.

### Data

| Target package | Contents | Source |
|---|---|---|
| `data/crypto/model/` | `IdentityKey`, `PeerKey`, `EncryptedBlob`, `MediaSecret`, `MessagePayload` | `IdentityKeyStore.kt`, `PeerKeyRegistry.kt`, `ConversationCrypto.kt`, `MessageEnvelope.kt` |
| `data/crypto/` (flat) | `KeyWrapper` / `KeystoreKeyWrapper`, `ParticipantResolver` / `ObxParticipantResolver` | `KeyWrapper.kt`, `ParticipantResolver.kt` |
| `data/local/obx/` (flat, no new package) | 8 entity files, 5 converter files, `ObxGson.kt` | `ObxEntities.kt`, `ObxConverters.kt` |
| `data/remote/dto/` (flat) | `RemoteMessageDto`, `RemoteMediaItemDto` | `MessageDTO.kt` |
| `data/remote/firebase/` (flat) | `MyFirebaseMessagingService`, `FcmEntryPoint` | `MyFirebaseMessagingService.kt` |
| `data/repository/` (flat) | `ConversationReconciler`, `RemoteConversations` | `ConversationReconciler.kt` |
| `data/media/` (flat) | `MediaTransferManager`, `TransferCancelledException` | `MediaTransferManager.kt` |

### Presentation - types

| Target package | Contents | Source |
|---|---|---|
| `presentation/base/mvi/` | `UiState`, `UiIntent`, `UiEffect`, `MviViewModel` | `MviViewModel.kt` |
| `presentation/state/chat/` | `ChatUiState`, `PendingAttachment`, `ChatUserProfileUiState`, `MediaTab` | `ChatUiState.kt`, `ChatUserProfileUiState.kt` |
| `presentation/state/account/` | `ChangeAccountUiState`, `AccountSummary`, `ChangePhoneUiState`, `ChangePhoneStep` | `ChangeAccountUiState.kt`, `ChangePhoneUiState.kt` |
| `presentation/state/media/` | `MediaPickerUiState`, `GalleryItem`, `GalleryFilter` | `FilePickerUiState.kt` |
| `presentation/components/call/model/` | `CallPartnerUi`, `CallPartnerCache`, `CallBarMode`, `SurfaceKind`, `VideoStream`, `PipForm` | `ActiveCallBar.kt`, `CallBarPresenter.kt`, `VideoSurfaceOwner.kt`, `PipCallSurface.kt` |
| `presentation/components/chat/model/` | `MessageAction`, `MessageContextMenuItem`, `MessageContextMenuState` | `MessageContextMenu.kt`, `MessageContextMenuHost.kt` |
| `presentation/components/common/model/` | `MainTab`, `PillButtonStyle`, `MotionSprings` | `FloatingTabBar.kt`, `PillButton.kt`, `MotionModifiers.kt` |
| `presentation/components/media/model/` | `MediaSource`, `CaptureMode` | `FullscreenMediaPager.kt`, `CameraCaptureScreen.kt` |
| `presentation/components/auth/model/` | `AuthMode`, `Country`, `Countries` | `AuthModeToggle.kt`, `Countries.kt` |
| `presentation/screens/ui/theme/model/` | `AppFont`, `AppTextSize`, `BarPosition`, `AppBackground`, `SavedWallpaper` | `Customization.kt`, `AppBackgrounds.kt` |
| `presentation/screens/ui/theme/` (flat) | `Dimens`, `Spacing` | `Dimens.kt` |
| `presentation/screens/settings/model/` | `GridTile`, `StorageSegment` | `AppearanceScreen.kt`, `DataStorageScreen.kt` |
| `presentation/screens/` (flat) | `CallsFilter` | `CallsScreen.kt` |
| `presentation/viewmodel/` (flat) | `CallExit`, `SafetyNumberState` | `CallViewModel.kt`, `SafetyNumberViewModel.kt` |
| `presentation/notification/` (flat) | `NotificationReplyReceiver`, `NotificationActionEntryPoint` | `NotificationReplyReceiver.kt` |

### Presentation - large composable files

One file per top-level composable.

**`presentation/screens/chat/`** from `ChatScreen.kt` (1129 lines):
`ChatScreenWithNav.kt`, `ChatScreenContent.kt`, `MessageComposer.kt`, `MessageBubble.kt`, `MessageAppearance.kt`, `ChatRow.kt`, `MediaReadPermissions.kt`, `ChatScreenPreview.kt`

**`presentation/screens/call/`** from `CallScreen.kt` (719 lines):
`CallScreen.kt`, `CallScreenContent.kt`, `CallBackButton.kt`, `AudioCallContent.kt`, `VideoCallContent.kt`, `DraggableLocalPreview.kt`, `CallControlsRow.kt`, `IncomingCallControls.kt`, `ActiveCallControls.kt`, `CallAvatar.kt`, `CallPermissions.kt`, `CallStatusText.kt`, `CallScreenPreviews.kt`

`callStatusText` is `internal` and covered by `app/src/test/.../CallStatusTextTest.kt`. That test's import is updated; the test itself is unchanged.

**`presentation/navigation/graph/`** from `navigation.kt` (565 lines):
`AppNavigation.kt`, `AuthDisplay.kt`, `MainDisplay.kt`, `ProvideNavAnimatedScope.kt`, `ProvideNavArgs.kt`, `NavArgsViewModelStoreOwner.kt`, `SendCallAction.kt`, `MainTabToNavRoute.kt`

## Branch and delivery

Base: `refactor/readability` cut from `develop`, then `feature/pip_polish` merged in.

`feature/pip_polish` is 3 commits directly ahead of `develop` (`651c128`, `008162f`, `8cd67ea`), so this merge is a fast-forward. **Consequence: the pull request into `develop` will contain those 3 pip_polish feature commits alongside the refactor commits.** This effectively ships the PiP work as part of this PR.

One commit per area group, each independently revertable. Example: `refactor: split ObxEntities into one file per entity`.

Delivery: push `refactor/readability` to `origin`, open a pull request into `develop`.

## Verification

A single verification pass at the end of the branch:

```
./gradlew assembleDebug
./gradlew testDebugUnitTest
```

Compilation is the meaningful check here. Because every change is a move, any missed import, stale reference, or visibility mistake is a compile error rather than a silent runtime bug. The 23 existing unit test files guard against accidental logic edits.

The ObjectBox split carries the only residual runtime risk. Since entities keep their package, `app/objectbox-models/default.json` should be unchanged after the build - this is checked explicitly with `git diff` before the PR is opened.

## Risks

| Risk | Mitigation |
|---|---|
| ObjectBox model regeneration triggers `DbSchemaException` | Entities keep their package; `default.json` diff verified as empty post-build |
| Converter behaviour changes from multiple `Gson` instances | Single instance hoisted to `internal val obxGson` |
| `private` to `internal` widening on 8 declarations | Listed explicitly above; no new call sites introduced |
| PR carries unmerged pip_polish feature work | Stated above; accepted as a consequence of the chosen base |
| Import churn across the codebase | `Message` and `MediaItem` are each imported by 20 files; all churn is compiler-verified |
