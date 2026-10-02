# File Layout Refactor Implementation Plan

> Historical. Executed on 2026-08-11 via PRs #3 and #4 into `develop`. Single-use types stayed in
> their declaring files, so some files listed below were never created and the `private` to
> `internal` promotions were dropped. Source paths reflect the plan, not the current tree.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give every file in `app/src/main` a single subject, so filenames predict contents.

**Architecture:** Pure move refactor. Declarations are relocated verbatim into new files grouped by cohesive area. The only edited lines are `package` statements and `import` statements, plus eight `private` to `internal` promotions that Kotlin's file-private visibility makes unavoidable. No logic changes, no renames, no deletions, no comments.

**Tech Stack:** Kotlin, Jetpack Compose, Hilt, ObjectBox, Gradle (Windows, `gradlew.bat`).

**Spec:** [[File Layout Refactor (refactor-readability)]]

---

## Why this plan has no red-green cycles

The standard TDD loop does not apply here. Nothing changes behaviour, so there is no failing test to write. Inventing tests for unmoved code would add noise without adding safety.

Verification is instead:

1. **The compiler.** Every missed import, stale reference, or visibility mistake becomes a compile error, not a silent runtime bug. This is genuinely strong for a move refactor.
2. **The existing 23 unit test files**, which catch accidental logic edits during the moves.
3. **An explicit `default.json` diff check** for the one task with runtime risk (ObjectBox).

Per the spec, the full build runs once at the end rather than per task.

## Standing rules for every task

- **Move declarations verbatim.** Copy the exact lines, do not retype or reformat them.
- **Never add comments or KDoc.** Project rule: zero comments.
- **Never add Claude attribution to commit messages.** Project rule.
- Each new file gets `package com.example.messenger.<target package>` as its first line, then only the imports that file actually needs.
- After moving a type out of a package, search the whole project for its name and add the new import wherever it is referenced:
  ```bash
  git grep -ln "\bTypeName\b" -- '*.kt'
  ```
- Commit at the end of each task. Do not batch tasks into one commit.

---

## Task 0: Create the branch

**Files:** none

- [ ] **Step 1: Confirm the working tree is clean**

```bash
git status --porcelain
```

Expected: only the untracked `s job: adding brief explanatory comments...` file, which is unrelated. If anything else is listed, stop and ask.

- [ ] **Step 2: Cut the branch from develop**

```bash
git checkout develop && git checkout -b refactor/readability
```

- [ ] **Step 3: Merge feature/pip_polish**

```bash
git merge feature/pip_polish
```

Expected: `Fast-forward`. `feature/pip_polish` is 3 commits directly ahead of `develop`, so no merge commit is produced and no conflicts are possible.

- [ ] **Step 4: Verify HEAD**

```bash
git log --oneline -1
```

Expected: `8cd67ea feat(call): morphing in-app floating call tile`

---

## Task 1: Domain model and service

**Files:**
- Create: `app/src/main/java/com/example/messenger/domain/model/call/CallType.kt`
- Create: `app/src/main/java/com/example/messenger/domain/model/call/CallSignal.kt`
- Create: `app/src/main/java/com/example/messenger/domain/model/call/CallStatus.kt`
- Create: `app/src/main/java/com/example/messenger/domain/model/call/CallConnectionState.kt`
- Create: `app/src/main/java/com/example/messenger/domain/model/presence/UserPresence.kt`
- Create: `app/src/main/java/com/example/messenger/domain/model/presence/PresenceState.kt`
- Create: `app/src/main/java/com/example/messenger/domain/model/sync/SyncAction.kt`
- Create: `app/src/main/java/com/example/messenger/domain/model/sync/SyncEntityType.kt`
- Create: `app/src/main/java/com/example/messenger/domain/service/CallEventListener.kt`
- Modify: `app/src/main/java/com/example/messenger/domain/model/Message.kt` (remove trailing `enum class CallType`)
- Delete: `app/src/main/java/com/example/messenger/domain/model/CallSignal.kt`, `PresenceState.kt`, `sync/SyncQueueConstants.kt`
- Modify: `app/src/main/java/com/example/messenger/domain/service/ICallService.kt`

- [ ] **Step 1: Extract `CallType` from `Message.kt`**

`Message.kt` ends with a single line after the sealed class:

```kotlin
enum class CallType { MISSED, UNREACHED, ENDED }
```

Delete that line from `Message.kt`. Create `domain/model/call/CallType.kt`:

```kotlin
package com.example.messenger.domain.model.call

enum class CallType { MISSED, UNREACHED, ENDED }
```

`Message.kt` keeps the `Message` sealed class and stays in `domain/model/`. Add `import com.example.messenger.domain.model.call.CallType` to it.

- [ ] **Step 2: Split `CallSignal.kt`**

Move `data class CallSignal` into `domain/model/call/CallSignal.kt` and `enum class CallStatus` into `domain/model/call/CallStatus.kt`, both with `package com.example.messenger.domain.model.call`. Delete the old `domain/model/CallSignal.kt`.

- [ ] **Step 3: Split `PresenceState.kt`**

Move `data class UserPresence` into `domain/model/presence/UserPresence.kt` and `enum class PresenceState` into `domain/model/presence/PresenceState.kt`, both with `package com.example.messenger.domain.model.presence`. Delete the old `domain/model/PresenceState.kt`.

- [ ] **Step 4: Split `SyncQueueConstants.kt`**

Move `object SyncAction` into `domain/model/sync/SyncAction.kt` and `object SyncEntityType` into `domain/model/sync/SyncEntityType.kt`. The package is unchanged (`com.example.messenger.domain.model.sync`) because `sync/` is already the cohesive area, so **no import updates are needed** for these two. Delete `SyncQueueConstants.kt`.

- [ ] **Step 5: Split `ICallService.kt`**

The file holds three declarations. Move `enum class CallConnectionState` to `domain/model/call/CallConnectionState.kt` (package `com.example.messenger.domain.model.call`) and `interface CallEventListener` to `domain/service/CallEventListener.kt` (package `com.example.messenger.domain.service`, unchanged). `ICallService.kt` keeps only `interface ICallService`, and gains an import for `CallConnectionState`.

- [ ] **Step 6: Fix every import**

```bash
git grep -ln "domain.model.CallType\|domain.model.CallSignal\|domain.model.CallStatus\|domain.model.UserPresence\|domain.model.PresenceState" -- '*.kt'
```

Rewrite each hit to the new package. Expected blast radius, measured on the current tree: `CallType` 10 files, `CallSignal` 5, `CallStatus` 5, `UserPresence` 6, `PresenceState` 6. `CallConnectionState` is imported from `domain.service` today, so search for that too:

```bash
git grep -ln "domain.service.CallConnectionState\|domain.service.CallEventListener" -- '*.kt'
```

`CallEventListener` keeps its package, so only `CallConnectionState` hits need rewriting.

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "refactor(domain): one type per file for call, presence and sync models"
```

---

## Task 2: Data crypto

**Files:**
- Create: `app/src/main/java/com/example/messenger/data/crypto/model/IdentityKey.kt`
- Create: `app/src/main/java/com/example/messenger/data/crypto/model/PeerKey.kt`
- Create: `app/src/main/java/com/example/messenger/data/crypto/model/EncryptedBlob.kt`
- Create: `app/src/main/java/com/example/messenger/data/crypto/model/MediaSecret.kt`
- Create: `app/src/main/java/com/example/messenger/data/crypto/model/MessagePayload.kt`
- Create: `app/src/main/java/com/example/messenger/data/crypto/KeystoreKeyWrapper.kt`
- Create: `app/src/main/java/com/example/messenger/data/crypto/ObxParticipantResolver.kt`
- Modify: `IdentityKeyStore.kt`, `PeerKeyRegistry.kt`, `ConversationCrypto.kt`, `KeyWrapper.kt`, `ParticipantResolver.kt`
- Delete: `app/src/main/java/com/example/messenger/data/crypto/MessageEnvelope.kt`

- [ ] **Step 1: Move the five model types into `data/crypto/model/`**

Each gets `package com.example.messenger.data.crypto.model`:

| New file | Moved from |
|---|---|
| `model/IdentityKey.kt` | `IdentityKeyStore.kt` (`class IdentityKey`) |
| `model/PeerKey.kt` | `PeerKeyRegistry.kt` (`class PeerKey`) |
| `model/EncryptedBlob.kt` | `ConversationCrypto.kt` (`class EncryptedBlob`) |
| `model/MediaSecret.kt` | `MessageEnvelope.kt` (`data class MediaSecret`) |
| `model/MessagePayload.kt` | `MessageEnvelope.kt` (`data class MessagePayload`) |

`MessageEnvelope.kt` also holds top-level mapper functions. Those stay behind — keep the file, now holding only the functions, and add imports for `MediaSecret` and `MessagePayload`. If removing both types leaves the file with no declarations at all, delete it instead and move the functions to `model/MessagePayload.kt`.

- [ ] **Step 2: Split the two interface/implementation pairs**

`KeyWrapper.kt` keeps `interface KeyWrapper`; `class KeystoreKeyWrapper` moves to `KeystoreKeyWrapper.kt`. `ParticipantResolver.kt` keeps `fun interface ParticipantResolver`; `class ObxParticipantResolver` moves to `ObxParticipantResolver.kt`. Both stay in `com.example.messenger.data.crypto`, so **no import updates are needed** for this step.

- [ ] **Step 3: Fix imports for the five moved model types**

```bash
git grep -ln "\bIdentityKey\b\|\bPeerKey\b\|\bEncryptedBlob\b\|\bMediaSecret\b\|\bMessagePayload\b" -- '*.kt'
```

Note `PeerKey` will also match `PeerKeyRegistry` and `ObxPeerKey`; check each hit rather than blind-replacing.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "refactor(crypto): extract key and envelope models into crypto/model"
```

---

## Task 3: ObjectBox entities

**Files:**
- Create 8 files in `app/src/main/java/com/example/messenger/data/local/obx/`
- Delete: `app/src/main/java/com/example/messenger/data/local/obx/ObxEntities.kt`

- [ ] **Step 1: Split into one file per entity**

Create `ObxMessage.kt`, `ObxUser.kt`, `ObxConversation.kt`, `ObxProfile.kt`, `ObxPeerKey.kt`, `ObxContactAlias.kt`, `ObxRecentContact.kt`, `ObxSyncQueueItem.kt`.

**The package stays `com.example.messenger.data.local.obx`.** This is the deliberate exception to the folder rule. Moving these to a subpackage would relocate the generated `MyObjectBox` class and rewrite `app/objectbox-models/default.json`, risking `DbSchemaException` on launch.

Because the package is unchanged, **no import updates are needed anywhere.** Carry each entity's `@Entity`, `@Id`, `@Index`, `@Unique` and `@Convert` annotations across exactly as written, and give each file only the imports it uses.

- [ ] **Step 2: Verify no import churn leaked in**

```bash
git diff --stat
```

Expected: only files under `data/local/obx/` appear. If any other file changed, an entity's package was altered by mistake — revert and redo.

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "refactor(obx): split ObxEntities into one file per entity"
```

---

## Task 4: ObjectBox converters

**Files:**
- Create: `app/src/main/java/com/example/messenger/data/local/obx/ObxGson.kt`
- Create: `MediaItemListConverter.kt`, `StringListConverter.kt`, `NullableStringListConverter.kt`, `PhoneNumberConverter.kt`, `PhoneVisibilityConverter.kt` in the same folder
- Delete: `app/src/main/java/com/example/messenger/data/local/obx/ObxConverters.kt`

- [ ] **Step 1: Hoist the shared Gson instance**

`ObxConverters.kt` currently declares one file-private instance shared by all five converters:

```kotlin
private val gson = Gson()
```

Splitting the file naively would give each converter its own instance, changing behaviour. Create `ObxGson.kt`:

```kotlin
package com.example.messenger.data.local.obx

import com.google.gson.Gson

internal val obxGson = Gson()
```

- [ ] **Step 2: Move each converter to its own file**

One file per converter, package unchanged (`com.example.messenger.data.local.obx`). In each moved converter, replace every `gson.` reference with `obxGson.`. This is the only body edit in the entire refactor, and it preserves the single-instance behaviour exactly.

- [ ] **Step 3: Confirm no stray Gson instances were introduced**

```bash
git grep -n "Gson()" -- 'app/src/main/java/com/example/messenger/data/local/obx/'
```

Expected: exactly one hit, in `ObxGson.kt`.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "refactor(obx): split converters into one file each with a shared Gson"
```

---

## Task 5: Data remote, repository and media

**Files:**
- Create: `data/remote/dto/RemoteMediaItemDto.kt`
- Create: `data/remote/firebase/FcmEntryPoint.kt`
- Create: `data/repository/RemoteConversations.kt`
- Create: `data/media/TransferCancelledException.kt`
- Modify: `data/remote/dto/MessageDTO.kt`, `data/remote/firebase/MyFirebaseMessagingService.kt`, `data/repository/ConversationReconciler.kt`, `data/media/MediaTransferManager.kt`

- [ ] **Step 1: Perform four two-way splits**

| Source file | Stays | Moves to new file |
|---|---|---|
| `dto/MessageDTO.kt` | `RemoteMessageDto` + its mapper functions | `RemoteMediaItemDto` → `dto/RemoteMediaItemDto.kt` |
| `firebase/MyFirebaseMessagingService.kt` | `MyFirebaseMessagingService` | `FcmEntryPoint` → `firebase/FcmEntryPoint.kt` |
| `repository/ConversationReconciler.kt` | `ConversationReconciler` | `RemoteConversations` → `repository/RemoteConversations.kt` |
| `media/MediaTransferManager.kt` | `MediaTransferManager` | `TransferCancelledException` → `media/TransferCancelledException.kt` |

Rename `MessageDTO.kt` to `RemoteMessageDto.kt` so the filename matches the type it now holds — this is a file rename, not a type rename, so no references change.

All four keep their existing packages, so **no import updates are needed.**

- [ ] **Step 2: Commit**

```bash
git add -A && git commit -m "refactor(data): separate DTOs, entry points and exceptions from their hosts"
```

---

## Task 6: MVI base

**Files:**
- Create: `presentation/base/mvi/UiState.kt`, `UiIntent.kt`, `UiEffect.kt`, `MviViewModel.kt`
- Delete: `presentation/base/MviViewModel.kt`

- [ ] **Step 1: Split the four declarations**

Each of `interface UiState`, `interface UiIntent`, `interface UiEffect` becomes a one-line file in `presentation/base/mvi/`, package `com.example.messenger.presentation.base.mvi`. `abstract class MviViewModel` moves to `mvi/MviViewModel.kt` and imports nothing extra — the three markers are now in its own package.

- [ ] **Step 2: Fix imports across every ViewModel and state class**

```bash
git grep -ln "presentation.base.UiState\|presentation.base.UiIntent\|presentation.base.UiEffect\|presentation.base.MviViewModel" -- '*.kt'
```

This is the widest-reaching import change in the refactor — every ViewModel and every UI state class references at least one of these. Rewrite each to `presentation.base.mvi.<Name>`.

`presentation/base/UiText.kt` and `StateViewModel` are **not** moved and keep the `presentation.base` package.

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "refactor(presentation): move MVI contracts into base/mvi"
```

---

## Task 7: UI state

**Files:**
- Create: `presentation/state/chat/ChatUiState.kt`, `PendingAttachment.kt`, `ChatUserProfileUiState.kt`, `MediaTab.kt`
- Create: `presentation/state/account/ChangeAccountUiState.kt`, `AccountSummary.kt`, `ChangePhoneUiState.kt`, `ChangePhoneStep.kt`
- Create: `presentation/state/media/MediaPickerUiState.kt`, `GalleryItem.kt`, `GalleryFilter.kt`
- Delete: `presentation/state/ChatUiState.kt`, `ChatUserProfileUiState.kt`, `ChangeAccountUiState.kt`, `ChangePhoneUiState.kt`, `FilePickerUiState.kt`

- [ ] **Step 1: Move into three area packages**

Packages: `com.example.messenger.presentation.state.chat`, `.account`, `.media`.

`FilePickerUiState.kt` is the three-way split: `MediaPickerUiState`, `GalleryItem`, `GalleryFilter` all go to `state/media/`.

`ChangeAccountUiState.kt` also holds top-level functions. Keep them with `ChangeAccountUiState.kt` in the new `state/account/` package.

- [ ] **Step 2: Fix imports**

```bash
git grep -ln "presentation.state.ChatUiState\|presentation.state.PendingAttachment\|presentation.state.ChatUserProfileUiState\|presentation.state.MediaTab\|presentation.state.ChangeAccountUiState\|presentation.state.AccountSummary\|presentation.state.ChangePhoneUiState\|presentation.state.ChangePhoneStep\|presentation.state.MediaPickerUiState\|presentation.state.GalleryItem\|presentation.state.GalleryFilter" -- '*.kt'
```

Check `app/src/test` too — the ViewModel tests reference these state classes.

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "refactor(state): group UI state by feature area"
```

---

## Task 8: Component support types

**Files:**
- Create: `presentation/components/call/model/CallPartnerUi.kt`, `CallPartnerCache.kt`, `CallBarMode.kt`, `SurfaceKind.kt`, `VideoStream.kt`, `PipForm.kt`
- Create: `presentation/components/chat/model/MessageAction.kt`, `MessageContextMenuItem.kt`, `MessageContextMenuState.kt`
- Create: `presentation/components/common/model/MainTab.kt`, `PillButtonStyle.kt`, `MotionSprings.kt`
- Create: `presentation/components/media/model/MediaSource.kt`, `CaptureMode.kt`
- Create: `presentation/components/auth/model/AuthMode.kt`, `Country.kt`, `Countries.kt`
- Modify the host files listed below
- Delete: `presentation/components/auth/Countries.kt`

- [ ] **Step 1: Extract types from their host component files**

| Type | Host file it leaves | Target |
|---|---|---|
| `CallPartnerUi`, `CallPartnerCache` | `call/ActiveCallBar.kt` | `call/model/` |
| `CallBarMode` | `call/CallBarPresenter.kt` | `call/model/` |
| `SurfaceKind`, `VideoStream` | `call/VideoSurfaceOwner.kt` | `call/model/` |
| `PipForm` | `call/PipCallSurface.kt` | `call/model/` |
| `MessageAction` | `chat/MessageContextMenu.kt` | `chat/model/` |
| `MessageContextMenuItem`, `MessageContextMenuState` | `chat/MessageContextMenuHost.kt` | `chat/model/` |
| `MainTab` | `common/FloatingTabBar.kt` | `common/model/` |
| `PillButtonStyle` | `common/PillButton.kt` | `common/model/` |
| `MotionSprings` | `common/MotionModifiers.kt` | `common/model/` |
| `MediaSource` | `media/FullscreenMediaPager.kt` | `media/model/` |
| `CaptureMode` | `media/CameraCaptureScreen.kt` | `media/model/` |
| `AuthMode` | `auth/AuthModeToggle.kt` | `auth/model/` |
| `Country`, `Countries` | `auth/Countries.kt` (delete after) | `auth/model/` |

Packages are the host package plus `.model`, e.g. `com.example.messenger.presentation.components.call.model`.

- [ ] **Step 2: Promote the two file-private types**

`CallPartnerCache` is declared `private object` and `PipForm` is declared `private sealed interface`. `CaptureMode` is declared `private enum class`. File-private visibility cannot survive the move. Change each to `internal`:

```kotlin
internal object CallPartnerCache {
internal sealed interface PipForm {
internal enum class CaptureMode { PHOTO, VIDEO }
```

Do not add any new call sites. This widens visibility from file to module and nothing else.

- [ ] **Step 3: Add imports to the host files and every other consumer**

`MainTab` in particular is referenced by navigation code, so check outside `components/`.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "refactor(components): extract component support types into model packages"
```

---

## Task 9: Theme, settings and remaining singles

**Files:**
- Create: `presentation/screens/ui/theme/model/AppFont.kt`, `AppTextSize.kt`, `BarPosition.kt`, `AppBackground.kt`, `SavedWallpaper.kt`
- Create: `presentation/screens/ui/theme/Spacing.kt`
- Create: `presentation/screens/settings/model/GridTile.kt`, `StorageSegment.kt`
- Create: `presentation/screens/CallsFilter.kt`
- Create: `presentation/viewmodel/CallExit.kt`, `SafetyNumberState.kt`
- Create: `presentation/notification/NotificationActionEntryPoint.kt`
- Modify: `theme/Customization.kt`, `theme/AppBackgrounds.kt`, `theme/Dimens.kt`, `settings/AppearanceScreen.kt`, `settings/DataStorageScreen.kt`, `screens/CallsScreen.kt`, `viewmodel/CallViewModel.kt`, `viewmodel/SafetyNumberViewModel.kt`, `notification/NotificationReplyReceiver.kt`

- [ ] **Step 1: Theme**

`Customization.kt` holds three enums (`AppFont`, `AppTextSize`, `BarPosition`) and `AppBackgrounds.kt` holds two data classes (`AppBackground`, `SavedWallpaper`). All five move to `theme/model/`, package `com.example.messenger.presentation.screens.ui.theme.model`. Both source files also hold top-level functions; those stay behind.

`Dimens.kt` holds `object Dimens` and `object Spacing`. `Spacing` moves to `theme/Spacing.kt`, package unchanged — no import updates needed.

- [ ] **Step 2: Settings**

`GridTile` (`private sealed interface`, `AppearanceScreen.kt:268`) and `StorageSegment` (`private data class`, `DataStorageScreen.kt:36`) move to `settings/model/`, package `com.example.messenger.presentation.screens.settings.model`. Promote both from `private` to `internal`.

- [ ] **Step 3: Remaining single extractions**

| Type | Leaves | Target file | Package change |
|---|---|---|---|
| `CallsFilter` | `screens/CallsScreen.kt` | `screens/CallsFilter.kt` | none |
| `CallExit` | `viewmodel/CallViewModel.kt` | `viewmodel/CallExit.kt` | none |
| `SafetyNumberState` | `viewmodel/SafetyNumberViewModel.kt` | `viewmodel/SafetyNumberState.kt` | none |
| `NotificationActionEntryPoint` | `notification/NotificationReplyReceiver.kt` | `notification/NotificationActionEntryPoint.kt` | none |

Only the theme and settings moves change packages, so import fixes are limited to those.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "refactor(presentation): extract theme, settings and viewmodel support types"
```

---

## Task 10: Split ChatScreen

**Files:**
- Create in `app/src/main/java/com/example/messenger/presentation/screens/chat/`: `ChatScreenWithNav.kt`, `ChatScreenContent.kt`, `MessageComposer.kt`, `MessageBubble.kt`, `MessageAppearance.kt`, `ChatRow.kt`, `MediaReadPermissions.kt`, `ChatScreenPreview.kt`
- Delete: `app/src/main/java/com/example/messenger/presentation/screens/ChatScreen.kt`

- [ ] **Step 1: Split by composable, using these line boundaries**

Line numbers are from `ChatScreen.kt` at branch HEAD (1129 lines). Verify each boundary before cutting — do not trust the numbers blindly if earlier tasks touched the file.

| Lines | Declaration | Target file |
|---|---|---|
| 147 | `private fun mediaReadPermissions` | `MediaReadPermissions.kt` |
| 155-298 | `fun ChatScreenWithNav` | `ChatScreenWithNav.kt` |
| 299-731 | `private fun ChatScreenContent` | `ChatScreenContent.kt` |
| 732-871 | `private fun MessageComposer` | `MessageComposer.kt` |
| 872-1020 | `private fun MessageBubble` | `MessageBubble.kt` |
| 1021-1082 | `private fun MessageAppearance` | `MessageAppearance.kt` |
| 1083-1089 | `private sealed interface ChatRow` | `ChatRow.kt` |
| 1090-1129 | `private fun ChatScreenPreview` | `ChatScreenPreview.kt` |

All get `package com.example.messenger.presentation.screens.chat`.

- [ ] **Step 2: Promote cross-file references**

`ChatScreenContent` and the other composables are `private` but are called from each other. Once they live in separate files, every one that is called from another file must become `internal`. `ChatRow` must become `internal` for the same reason. `ChatScreenWithNav` is already public and stays public.

Work out which ones actually need promoting by letting the compiler tell you in Task 13 — but promote at minimum: `ChatScreenContent`, `MessageComposer`, `MessageBubble`, `MessageAppearance`, `ChatRow`, `mediaReadPermissions`.

- [ ] **Step 3: Fix the navigation import**

`ChatScreenWithNav` is referenced from `presentation/navigation/navigation.kt`. Add `import com.example.messenger.presentation.screens.chat.ChatScreenWithNav`.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "refactor(chat): split ChatScreen into one file per composable"
```

---

## Task 11: Split CallScreen

**Files:**
- Create in `app/src/main/java/com/example/messenger/presentation/screens/call/`: `CallScreen.kt`, `CallScreenContent.kt`, `CallBackButton.kt`, `AudioCallContent.kt`, `VideoCallContent.kt`, `DraggableLocalPreview.kt`, `CallControlsRow.kt`, `IncomingCallControls.kt`, `ActiveCallControls.kt`, `CallAvatar.kt`, `CallPermissions.kt`, `CallFormatting.kt`, `CallScreenPreviews.kt`
- Delete: `app/src/main/java/com/example/messenger/presentation/screens/CallScreen.kt`
- Modify: `app/src/test/java/com/example/messenger/presentation/screens/CallStatusTextTest.kt`

- [ ] **Step 1: Split by composable, using these line boundaries**

Line numbers from `CallScreen.kt` at branch HEAD (719 lines).

| Lines | Declaration | Target file |
|---|---|---|
| 74-116 | `fun CallScreen` | `CallScreen.kt` |
| 117-183 | `private fun rememberCallPermissions` + `private class CallPermissions` | `CallPermissions.kt` |
| 184-227 | `private fun CallScreenContent` | `CallScreenContent.kt` |
| 228-249 | `private fun CallBackButton` | `CallBackButton.kt` |
| 250-323 | `private fun AudioCallContent` | `AudioCallContent.kt` |
| 324-466 | `private fun VideoCallContent` | `VideoCallContent.kt` |
| 467-508 | `private fun DraggableLocalPreview` | `DraggableLocalPreview.kt` |
| 509-548 | `private fun CallControlsRow` | `CallControlsRow.kt` |
| 549-564 | `private fun RowScope.IncomingCallControls` | `IncomingCallControls.kt` |
| 565-611 | `private fun RowScope.ActiveCallControls` | `ActiveCallControls.kt` |
| 612-624, 631-637 | `internal fun callStatusText`, `private fun formatSeconds` | `CallFormatting.kt` |
| 625-630 | `private fun CallAvatar` | `CallAvatar.kt` |
| 638-719 | the five `@Preview` composables | `CallScreenPreviews.kt` |

`rememberCallPermissions` and `CallPermissions` stay together — the function exists only to build the class. All files get `package com.example.messenger.presentation.screens.call`.

- [ ] **Step 2: Promote visibility as needed**

Promote `CallPermissions` from `private` to `internal`. Promote every composable called across the new file boundaries to `internal`. `callStatusText` is already `internal` and stays so.

- [ ] **Step 3: Update the existing test's import**

`app/src/test/java/com/example/messenger/presentation/screens/CallStatusTextTest.kt` imports `callStatusText`. Change the import to `com.example.messenger.presentation.screens.call.callStatusText`. **Do not change any assertion in the test** — it must keep passing unmodified, which is what proves the move was clean.

- [ ] **Step 4: Fix the navigation import**

`CallScreen` is referenced from `navigation.kt`. Add `import com.example.messenger.presentation.screens.call.CallScreen`.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "refactor(call): split CallScreen into one file per composable"
```

---

## Task 12: Split navigation

**Files:**
- Create in `app/src/main/java/com/example/messenger/presentation/navigation/graph/`: `AppNavigation.kt`, `AuthDisplay.kt`, `MainDisplay.kt`, `ProvideNavAnimatedScope.kt`, `ProvideNavArgs.kt`, `NavArgsViewModelStoreOwner.kt`, `SendCallAction.kt`, `MainTabToNavRoute.kt`
- Delete: `app/src/main/java/com/example/messenger/presentation/navigation/navigation.kt`
- Modify: `app/src/main/java/com/example/messenger/presentation/MainActivity.kt`

- [ ] **Step 1: Split by declaration, using these line boundaries**

Line numbers from `navigation.kt` at branch HEAD (565 lines).

| Lines | Declaration | Target file |
|---|---|---|
| 84-112 | `fun AppNavigation` | `AppNavigation.kt` |
| 113-134 | `private fun AuthDisplay` | `AuthDisplay.kt` |
| 135-513 | `private fun MainDisplay` | `MainDisplay.kt` |
| 514-522 | `private fun ProvideNavAnimatedScope` | `ProvideNavAnimatedScope.kt` |
| 523-536 | `internal fun ProvideNavArgs` | `ProvideNavArgs.kt` |
| 537-555 | `private class NavArgsViewModelStoreOwner` | `NavArgsViewModelStoreOwner.kt` |
| 556-560 | `private fun sendCallAction` | `SendCallAction.kt` |
| 561-565 | `private fun MainTab.toNavRoute` | `MainTabToNavRoute.kt` |

All get `package com.example.messenger.presentation.navigation.graph`.

`MainDisplay` is 379 lines on its own and remains the largest file after this refactor. That is acceptable — it is a single coherent composable, and breaking it up would be a behavioural refactor of the kind explicitly deferred for `CallForegroundService`.

- [ ] **Step 2: Promote visibility**

`NavArgsViewModelStoreOwner` goes from `private` to `internal`. Promote `AuthDisplay`, `MainDisplay`, `ProvideNavAnimatedScope`, `sendCallAction` and `toNavRoute` to `internal` since they are now called across file boundaries.

- [ ] **Step 3: Fix the MainActivity import**

`AppNavigation` is called from `MainActivity.kt`. Add `import com.example.messenger.presentation.navigation.graph.AppNavigation`.

`Routes.kt` stays in `presentation/navigation/` untouched, so route imports do not change.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "refactor(navigation): split navigation graph into one file per composable"
```

---

## Task 13: Verify and open the pull request

**Files:** none

- [ ] **Step 1: Build**

```bash
./gradlew assembleDebug
```

Expected: `BUILD SUCCESSFUL`. Any failure here is a missed import, a stale reference, or a visibility promotion that was skipped. Fix and rerun before continuing — do not proceed with a red build.

- [ ] **Step 2: Run the unit tests**

```bash
./gradlew testDebugUnitTest
```

Expected: `BUILD SUCCESSFUL`, all tests passing, including `CallStatusTextTest` whose assertions were never touched.

- [ ] **Step 3: Confirm the ObjectBox model did not change**

```bash
git diff --stat app/objectbox-models/default.json
```

Expected: **empty output.** If `default.json` changed, an entity's package or name was altered during Task 3. Stop, revert that change, and redo it — shipping a modified model risks `DbSchemaException` on launch for anyone with existing data.

- [ ] **Step 4: Sanity-check the shape of the diff**

```bash
git diff develop --stat | tail -5
git diff develop --shortstat
```

Sanity check: insertions and deletions should be roughly balanced, because this is a move refactor. A large net gain in lines means something was duplicated rather than moved.

- [ ] **Step 5: Confirm no comments were added**

```bash
git diff develop -- '*.kt' | grep -E "^\+\s*(//|/\*|\*)" | head
```

Expected: empty output. Project rule is zero comments.

- [ ] **Step 6: Push**

```bash
git push -u origin refactor/readability
```

- [ ] **Step 7: Open the pull request**

```bash
gh pr create --base develop --title "refactor: one subject per file across app/src/main" --body "See branch commits. Pure move refactor: no logic changes, no renames, no deletions.

Splits 51 files so each holds a single subject, and breaks ChatScreen, CallScreen and navigation.kt into one file per composable.

Note: this branch was cut from develop with feature/pip_polish merged in, so it also carries commits 651c128, 008162f and 8cd67ea.

Deliberately excluded: Routes.kt (a route table that reads better whole) and CallForegroundService.kt (a single 1003-line class whose split is a behavioural refactor, deferred to its own branch).

Eight declarations were promoted from private to internal because Kotlin file-private visibility cannot survive a move to a new file."
```

Do not merge the PR. Review is the user's call.

---

## Follow-up work, not in this plan

`CallForegroundService.kt` still needs its 1003 lines broken up. That is a behavioural refactor requiring testing against real call flows, and belongs on a dedicated branch.
