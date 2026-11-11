# Test Coverage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give every layer of the app behaviour-level tests: integration tests for the data layer,
unit tests for the untested ViewModels and logic-bearing use cases, and screen tests that run in CI.

**Architecture:** All new tests live in `app/src/test` so CI's `testDebugUnitTest` runs them. CI
has no emulator, so device tests are manual-only. Integration tests run repositories against a
real ObjectBox store in a temp directory, with the Firebase boundary (`FirestoreService`,
`FirebaseAuthService`) stubbed by MockK. ViewModel tests follow `EditContactDataViewModelTest`:
real use cases over the fakes in `testutil/fakes`, with MockK only for concrete Firebase and
Android classes. Screen tests render the stateless `*ScreenContent` composables under Robolectric.

**Tech Stack:** JUnit 5, AssertK, Turbine, MockK, kotlinx-coroutines-test, ObjectBox 5.4.2 (JVM
native library), Robolectric with the JUnit Vintage engine, Compose `ui-test-junit4`, Kover.

**Spec:** Coverage audit of 2026-10-21 (develop `c34d740`), tracked in GitHub issue #21. Baseline:
- 346 JVM tests in 57 classes.
- 61 device Compose tests: 51 contrast checks, 10 gesture tests, and 21 known contrast failures.
- 0 integration tests.
- 3 of 19 ViewModels, 8 of 33 use cases and 0 of 10 repository implementations tested.

---

## Global Constraints

- No code comments of any kind (no `//`, no KDoc).
- No wildcard imports.
- Fakes over mocks for domain interfaces (`I*Repository`, `I*Service`). MockK only for concrete
  Firebase or Android classes (`FirestoreService`, `FirebaseAuthService`, `FirebaseAuth`,
  `PresenceManager`, `SyncCoordinator`, recorders, `Activity`).
- Test names are backticked sentences, as in the existing suite.
- Assertions use AssertK. Flows are asserted with Turbine.
- JUnit 5 everywhere except Robolectric classes, which are JUnit 4 and run via the Vintage engine.
- Every Gradle call uses the low-memory flags, with Android Studio closed:
  `./gradlew -Pkotlin.compiler.execution.strategy=in-process -Dorg.gradle.jvmargs="-Xmx3072m -XX:MaxMetaspaceSize=768m -Dfile.encoding=UTF-8" --max-workers=2 --console=plain <tasks>`.
  Below, `GRADLE` stands for that prefix.
- Never run `connectedDebugAndroidTest`, because it uninstalls the app and wipes the E2EE keys.
- Commit as `Derek Burned <danillolka2@gmail.com>`. Never add a Co-Authored-By or any Claude
  trailer.
- One logical change per commit. Commit when its tests pass.
- All phases share one branch, `test/coverage`, cut from `develop` (owner's choice, 2026-10-21).
  After the first push, open one draft PR into `develop` (`gh pr create --draft --base develop`)
  that references #21. Merge with `gh pr merge --merge`.
- Commits that change production code (Tasks 2 and 10 only) run `/verify` first. Test-only
  commits skip it.
- No docs in the app repo.
- A new test file holds one class. Fakes go in `testutil/fakes`. JUnit extensions go in
  `testutil`.
- Production code changes only where a task names the bug it fixes.

## Review Focus

1. **Over-long or blank chat text is still sent after the error (#17).** Pinned in Task 2:
   `text over 5000 characters emits an error and sends nothing`.
2. **Edit chats re-selects everything after the user deselects all (#18).** Pinned in Task 10:
   `deselecting every chat survives a list update`.
3. **A message that failed offline must reach the server on the next sync, and stop retrying after
   5 failures.** Pinned in Task 5: `a message that failed offline is delivered by the next sync`
   and `fifth failed resend drops the item and leaves the message FAILED`.
4. **Re-fetching history must not un-read, un-delete or un-play local messages.** Pinned in Task 3:
   `re-fetched history keeps local read, deleted and played flags`.
5. **History cleared "for me" must stay cleared when older pages load.** Pinned in Task 3:
   `loading older messages passes the clear-for-me cutoff`.

Not pinned here because each needs a product decision first: #19 (Edit chats delete is
local-only) and #20 (change-phone ignores a failed profile write). Task 10 adds no test for #19's
behaviour. Task 9 adds no test for #20's failure path.

---

## Phase 0: Test infrastructure

### Task 1: Kover report and the shared Log extension

**Files:**
- Modify: `gradle/libs.versions.toml` (add a `kover` version and plugin)
- Modify: `build.gradle.kts` (add `alias(libs.plugins.kover) apply false`)
- Modify: `app/build.gradle.kts` (apply Kover, exclude generated code)
- Modify: `.github/workflows/blank.yml` (run `koverXmlReportDebug` and upload the report)
- Modify: `.gitignore` (ignore ObjectBox's extracted native libraries)
- Create: `app/src/test/java/com/example/messenger/testutil/AndroidLogExtension.kt`
- Modify: the six test classes that stub `Log` by hand (`CallViewModelTest`,
  `NotificationReplyReceiverTest`, `RemoteConfigFeatureFlagsTest`,
  `MyFirebaseMessagingServiceTest`, `IncomingCallGateTest`, `IncomingCallCoordinatorTest`)

**Interfaces:**
- Produces: `class AndroidLogExtension : BeforeEachCallback, AfterEachCallback`. It runs
  `mockkStatic(Log::class)` and stubs every `Log.v/d/i/w/e` overload (with and without a
  `Throwable`, and `Log.w(String, Throwable)`) to return 0. Afterwards it calls
  `unmockkStatic(Log::class)`. Register it with `@JvmField @RegisterExtension val log = AndroidLogExtension()`.

- [ ] **Step 1: Add Kover.** Use plugin id `org.jetbrains.kotlinx.kover` at the newest release on
  plugins.gradle.org, recorded in the version catalog. Exclude these from reports:
  - classes matching `*_`, `MyObjectBox`, `*Cursor`, `*_Factory`, `*_HiltModules*`, `Hilt_*`,
    `*ComposableSingletons*`, `BuildConfig`
  - the package `com.example.messenger.di`
- [ ] **Step 2: Run** `GRADLE :app:koverHtmlReportDebug`. Expected: BUILD SUCCESSFUL, and
  `app/build/reports/kover/htmlDebug/index.html` exists. Record the line and branch coverage
  percentages. They are the baseline for Task 14 and the PR description.
- [ ] **Step 3: Add** `/app/objectbox-jni-*` and `/app/libobjectbox-jni-*` to `.gitignore`. The
  untracked `app/objectbox-jni-windows-x64.dll` must disappear from `git status`.
- [ ] **Step 4: Write `AndroidLogExtension`, then use it in the six classes.** Remove their
  hand-written `mockkStatic(Log::class)`, `every { Log... }` and `unmockkStatic(Log::class)`
  lines, and leave their other static mocks untouched.
- [ ] **Step 5: Run** `GRADLE :app:testDebugUnitTest`. Expected: all 346 existing tests pass.
- [ ] **Step 6: Add a CI step** to `blank.yml` after the unit tests: `./gradlew koverXmlReportDebug`,
  then upload `app/build/reports/kover/` as artifact `coverage-report`.
- [ ] **Step 7: Commit** in two commits:
  - `build: add Kover coverage reports and ignore ObjectBox native libs`
  - `test: share one android.util.Log stub across unit tests`

Phases run in order on the one branch. Phase 3 uses the `FakeMessageRepository` members that
Task 2 adds.

---

## Phase 1: Use cases

### Task 2: Logic-bearing use cases, fixing #17

**Files:**
- Modify: `app/src/main/java/com/example/messenger/domain/usecase/message/SendMessageUseCase.kt`
- Create: `app/src/test/java/com/example/messenger/domain/usecase/message/SendMessageUseCaseTest.kt`
- Create: `app/src/test/java/com/example/messenger/domain/usecase/message/GetMessagesUseCaseTest.kt`
- Create: `app/src/test/java/com/example/messenger/domain/usecase/message/SyncMessagesUseCaseTest.kt`
- Create: `app/src/test/java/com/example/messenger/domain/usecase/message/MarkMessageAsReadUseCaseTest.kt`
- Modify: `app/src/test/java/com/example/messenger/testutil/fakes/FakeMessageRepository.kt`

**Interfaces:**
- Consumes: `AndroidLogExtension` (Task 1); `FakeAuthRepository`, which already backs
  `GetCurrentUserUseCase` in the existing auth tests.
- Produces: new members on `FakeMessageRepository`, used again by Tasks 7 and 8:
  - `val sent: MutableList<Message>` (recorded by `sendMessage`)
  - `var sendResult: Result<Unit>`
  - `val markedRead: MutableList<Message>` (recorded by `markMessageAsRead`)
  - `var markReadResult: Result<Unit>`
  - `var recentResults: Flow<Result<Unit>>` (returned by `observeRecentMessages`)
  - `var loadOlderResult: Result<Int>`
  - `var messagesError: Throwable?` (when set, `getMessagesStream` returns
    `flow { throw messagesError }`)

Pure one-line delegates are deliberately left untested. The ViewModel tests in Phase 3 cover them
through real use-case instances. They are: GetCurrentUser, LinkPhone, LoginWithPhoneNumber,
ObserveAuthState, the conversation mark/delete/clear/sync use cases, GetConversations,
ObserveConversations, CancelMediaTransfer, DownloadMedia, LoadOlderMessages, SendMediaMessage,
ObserveUserPresence, ObserveNetwork, ProcessSyncQueue, SyncAllData and ObserveTyping.

- [ ] **Step 1: Write `SendMessageUseCaseTest`.**
  - `blank text emits an error and sends nothing`: the flow emits exactly
    `[Resource.Error("Message is empty")]`, and `sent` is empty.
  - `text over 5000 characters emits an error and sends nothing`: input `"a".repeat(5001)`; the
    flow emits exactly `[Resource.Error("Message is too long")]`, and `sent` is empty.
  - `exactly 5000 characters is sent`: `sent.single().text.length == 5000`.
  - `signed-out user gets an error`: emits `Resource.Error("User not logged in")`, and `sent` is
    empty.
  - `message is trimmed, SENDING, and carries the reply snippet`: input `"  hi  "` with
    `replyTo = Message.Text(id = "r1", senderId = "bob", text = "orig", ...)`. Expect:
    - text is `"hi"` and status is `MessageStatus.SENDING`
    - `replyToMessageId == "r1"` and `replyToSenderId == "bob"`
    - `replyToText == replyTo.replySnippet()`
  - `repository failure emits its message`: `sendResult = Result.failure(Exception("offline"))`;
    emits `Resource.Error("offline")`.
- [ ] **Step 2: Run** `GRADLE :app:testDebugUnitTest --tests "*SendMessageUseCaseTest"`.
  Expected: FAIL in the blank and over-5000 tests, which see a second emission and one sent
  message.
- [ ] **Step 3: Fix `SendMessageUseCase.invoke`.** Add `return@flow` after each of the two
  validation `emit`s. Change nothing else.
- [ ] **Step 4: Run the same command.** Expected: PASS.
- [ ] **Step 5: Write the other three tests.**
  - `GetMessagesUseCaseTest`:
    - `blank conversation id emits invalid conversation error` (exact text
      `"Invalid conversation ID"`)
    - `stream emits Loading then Success with the messages`
    - `stream exception becomes an Error with its message`
  - `SyncMessagesUseCaseTest`:
    - `each sync result maps to Success or Error`: `recentResults` is
      `flowOf(success, failure("x"))`; expect `Loading`, `Success(Unit)`, `Error("x")`.
  - `MarkMessageAsReadUseCaseTest`:
    - `success maps to Success`
    - `failure maps to Error with the exception text`
- [ ] **Step 6: Run** `GRADLE :app:testDebugUnitTest --tests "*domain.usecase.message*"`.
  Expected: PASS.
- [ ] **Step 7: Run `/verify`, then commit** in two commits:
  - `fix(chat): stop SendMessageUseCase sending text it rejected` with body `Fixes #17`
  - `test: cover the message use cases that hold logic`

---

## Phase 2: Data-layer integration

### Task 3: ObjectBox test store and MessageRepositoryImpl

**Files:**
- Create: `app/src/test/java/com/example/messenger/testutil/ObjectBoxExtension.kt`
- Create: `app/src/test/java/com/example/messenger/data/repository/MessageRepositoryImplIntegrationTest.kt`
- Modify (only if Step 1 finds no native library): `gradle/libs.versions.toml`,
  `app/build.gradle.kts`

**Interfaces:**
- Produces: `class ObjectBoxExtension : BeforeEachCallback, AfterEachCallback`, with:
  - `lateinit var store: BoxStore`
  - `inline fun <reified T> box(): Box<T>`
  - `beforeEach`: builds `MyObjectBox.builder().directory(dir).build()` in a fresh
    `Files.createTempDirectory("obx")`
  - `afterEach`: `store.close()`, then `BoxStore.deleteAllFiles(dir)`
- Produces: a private helper inside each integration test,
  `fun firestore(): FirestoreService = mockk(relaxed = true)`, with `coEvery` overrides per test.
  Auth is `mockk<FirebaseAuthService> { every { getCurrentUserId() } returns "me" }`.
- Seed conversations with the existing `Conversation.toObx()` mapper. Build messages as domain
  `Message` values.

- [ ] **Step 1: Check the native library.** Run
  `GRADLE :app:dependencies --configuration debugUnitTestRuntimeClasspath` and search the output
  for `objectbox-windows` (local) and `objectbox-linux` (CI).
  - If either is missing, add `io.objectbox:objectbox-windows` and `io.objectbox:objectbox-linux`
    at `version.ref = "objectbox"` as `testImplementation`.
- [ ] **Step 2: Write the extension and the test class.** Register both `ObjectBoxExtension` and
  `AndroidLogExtension`. The tests:
  - **Sending:**
    - `sent message is stored and appears in the conversation stream`: Turbine on
      `getMessagesStream("c1")`; the last item contains the message with status `SENDING`.
    - `failed send marks the message FAILED and queues a SEND retry`: `sendMessage` returns a
      failure. Expect:
      - the stream shows `MessageStatus.FAILED`
      - `box<ObxSyncQueueItem>()` holds one item with `action == SyncAction.SEND` and the
        message id as `entityId`
    - `sending updates the conversation preview`: `lastMessage`, `latestMessageText` and
      `lastMessageSenderId` on the `c1` row match the message.
    - `three photos without a caption preview as 3 photos`: a `Message.Media` with three
      `MediaItem.IMAGE` items gives the preview `"3 photos"`.
  - **Deleting:**
    - `deleting someone else's message for everyone is refused and keeps it`: expect a
      `SecurityException` failure, and the message is still in the stream.
    - `deleting for me hides the message even when Firestore fails`: the message leaves the
      stream, and one `SyncAction.DELETE_FOR_ME` item is queued.
  - **Incoming and re-fetched messages:**
    - `incoming duplicate is stored once`: `persistIncomingMessage` twice gives
      `box<ObxMessage>().count() == 1`.
    - `re-fetched history keeps local read, deleted and played flags`:
      1. Seed a local row with `isRead`, `deleted` and `clipPlayed` true.
      2. `fetchRecentMessagesOnce` returns the same id with all three false.
      3. Call `backfillRecentMessages`. The stored row keeps all three true.
  - **Loading older pages:**
    - `loading older messages passes the clear-for-me cutoff`: with the conversation's
      `clearedAtForMe = 500`, expect
      `coVerify { fetchOlderMessages("c1", oldestId, 100, 500) }`.
    - `loading older with no local messages returns zero without calling Firestore`.
- [ ] **Step 3: Run** `GRADLE :app:testDebugUnitTest --tests "*MessageRepositoryImplIntegrationTest"`.
  Expected: PASS. If a test fails, it has found a real bug: stop and report it rather than
  bending the test.
- [ ] **Step 4: Commit** `test: run MessageRepositoryImpl against a real ObjectBox store`.

### Task 4: ConversationRepositoryImpl

**Files:**
- Create: `app/src/test/java/com/example/messenger/data/repository/ConversationRepositoryImplIntegrationTest.kt`

**Interfaces:**
- Consumes: `ObjectBoxExtension` (Task 3), `AndroidLogExtension` (Task 1).

- [ ] **Step 1: Write the tests.**
  - **Read state:**
    - `marking read zeroes unread and marks only partner messages read`: my own unread message
      stays `isRead == false`, and `resetUnreadCount("c1", "me")` is verified.
    - `marking unread flips only the latest partner message and sets unread to at least 1`.
  - **Syncing:**
    - `sync prunes conversations the server no longer has, with their messages`
    - `sync keeps everything when the server fetch fails`: `fetchAllConversationsOnce` returns
      a failure, and both local rows stay.
  - **Creating:**
    - `creating reuses an existing remote conversation`: `findExistingConversation` returns `c9`;
      the result id is `c9`, and `createConversation` on Firestore is never called.
    - `creating with a deleted account fails`: `getUserProfile` returns `deleted = true`; the
      result is a failure, and nothing is stored.
  - **Clearing and observing:**
    - `clearing for me sends the latest timestamp as cutoff and prunes locally`: with
      `lastMessageTimestamp = 10` and `latestMessageTimestamp = 20`, the cutoff is 20.
    - `getAllConversations emits local rows newest first`: with a null user id, no remote
      listener starts.
- [ ] **Step 2: Run** `GRADLE :app:testDebugUnitTest --tests "*ConversationRepositoryImplIntegrationTest"`.
  Expected: PASS.
- [ ] **Step 3: Commit** `test: run ConversationRepositoryImpl against a real ObjectBox store`.

### Task 5: SyncRepositoryImpl and the offline-send flow

**Files:**
- Create: `app/src/test/java/com/example/messenger/data/repository/SyncRepositoryImplIntegrationTest.kt`

**Interfaces:**
- Consumes: `ObjectBoxExtension`, `FakeConversationRepository`, `FakeUserRepository`, and a real
  `MessageRepositoryImpl` that shares the same store.

- [ ] **Step 1: Write the tests.**
  - **Retrying sends:**
    - `queued send is resent and the message flips to SENT`: the queue empties, and the row
      status is `"SENT"`.
    - `failed resend increments retryCount and keeps the item`
    - `fifth failed resend drops the item and leaves the message FAILED`: seed `retryCount = 4`.
  - **Other queue items:**
    - `queued item for a missing message is dropped`
    - `queued DELETE_FOR_ME calls deleteMessageForMe`
    - `items are processed oldest first`: `coVerifyOrder` over two sends with timestamps 2 and 1.
  - **Full sync:**
    - `syncAllData still processes the queue when conversation sync throws`: status ends
      `SyncStatus.SYNCED`.
    - `a message that failed offline is delivered by the next sync`:
      1. `MessageRepositoryImpl.sendMessage` runs while Firestore fails.
      2. Firestore recovers, and `SyncRepositoryImpl.processSyncQueue()` runs.
      3. `getMessagesStream` shows `SENT`, and the queue is empty.
- [ ] **Step 2: Run** `GRADLE :app:testDebugUnitTest --tests "*SyncRepositoryImplIntegrationTest"`.
  Expected: PASS.
- [ ] **Step 3: Commit** `test: cover the sync queue and offline delivery end to end`.

### Task 6: UserRepositoryImpl and the small ObjectBox repositories

**Files:**
- Create: `app/src/test/java/com/example/messenger/data/repository/UserRepositoryImplIntegrationTest.kt`
- Create: `app/src/test/java/com/example/messenger/data/repository/RecentContactsRepositoryIntegrationTest.kt`
- Create: `app/src/test/java/com/example/messenger/data/repository/CallHistoryRepositoryIntegrationTest.kt`
- Create: `app/src/test/java/com/example/messenger/data/repository/LocalRepositoryImplIntegrationTest.kt`

- [ ] **Step 1: Write the tests.**
  - `UserRepositoryImplIntegrationTest`:
    - **Contact aliases:**
      - `renaming a contact trims the name, caches it, and the alias stream emits it`
      - `a blank contact name is rejected and nothing is cached`
      - `refreshing aliases caches every alias the server returns`
    - **Fetching users:**
      - `a user missing on the server is cached as deleted`: `fetchRemoteUser` returns
        `RemoteUserSnapshot(user = null, authoritative = true)` for a cached user. The result
        and the stored row both have `deleted == true`.
      - `an unreachable server returns the cached user`: `fetchRemoteUser` returns a failure.
  - `RecentContactsRepositoryIntegrationTest`:
    - `opening the same contact twice keeps one entry with the newest name`
    - `a blank name does not erase the stored one`
    - `recently opened lists newest first and stops at the limit`
    - `blank user id is ignored`
  - `CallHistoryRepositoryIntegrationTest`:
    - `call history lists call messages newest first and skips deleted ones`
    - `entry takes the partner's name and avatar from the conversation`
    - `outgoing is true only for calls I placed`
    - `a call whose conversation is gone is dropped`
  - `LocalRepositoryImplIntegrationTest`:
    - `resetDB empties every box`
- [ ] **Step 2: Run** `GRADLE :app:testDebugUnitTest --tests "*IntegrationTest"`. Expected: PASS.
- [ ] **Step 3: Commit** `test: cover user cache, recent contacts, call history and local reset`.

Out of scope for Phase 2, with reasons:
- `AuthRepositoryImpl`: delegates to Firebase services with no local state.
- `ClipRepositoryImpl`, `MediaRepositoryImpl`: their decision logic is already pulled out and
  tested (`StrandedClips`, `NextVoiceClip`, `ClipUploadTimeout`, `InFlight`). The rest is file
  transfer through `MediaTransferManager` and `MediaFileTransfer`, which needs its own
  fake-transfer harness. That is left out of this plan.

---

## Phase 3: ViewModels

Shared rules for Tasks 7 to 10:
- Use `runTest(mainDispatcher.dispatcher)` wherever a test needs virtual time. That way `delay`
  in `viewModelScope` and `advanceTimeBy` share one scheduler.
- Construct use cases for real over fakes. Never mock a use case.
- `ChatViewModel` starts an endless 60 s last-seen ticker whenever `partnerId` is set. Any test
  that advances virtual time must build it without `partnerId`, or the scheduler never goes idle.
- `CallsViewModel.calls` is `stateIn(WhileSubscribed)`. Collect it with Turbine, or it never
  starts.

### Task 7: New fakes and ChatViewModel

**Files:**
- Create in `app/src/test/java/com/example/messenger/testutil/fakes/`:
  - `FakeMediaRepository.kt`
  - `FakeTypingService.kt`
  - `FakePresenceService.kt`
- Create: `app/src/test/java/com/example/messenger/presentation/screens/chat/ChatViewModelTest.kt`

**Interfaces:**
- Produces three fakes:
  - `FakeMediaRepository : IMediaRepository`:
    - `override val transfers = MutableStateFlow(emptyMap<String, MediaTransfer>())`
    - `val sentMedia: MutableList<Triple<String, List<String>, String>>`
    - `val downloads: MutableList<MediaItem>`
    - `val retried: MutableList<String>`
  - `FakeTypingService : ITypingService`:
    - `val typing = MutableStateFlow(emptySet<String>())`
    - `var setCount: Int`, `var clearCount: Int`, `var clearNowCount: Int`
  - `FakePresenceService : IPresenceService`:
    - `val presence = MutableStateFlow(UserPresence())`
    - `val multiple = MutableStateFlow(emptyMap<String, UserPresence>())`
- The test's `createViewModel(args: Map<String, Any?>)` builds:
  - mocks: `mockk(relaxed = true)` for `VoiceRecorder`, `VideoNoteRecorder`, `VoiceDraftStore`,
    `ClipPlaybackController` and `RecentContactsRepository`. Stub the recorders' `state` with a
    `MutableStateFlow` of their idle value, and stub `voiceDrafts.observe(any())` with
    `flowOf(null)`.
  - `FirebaseAuthService` with `getCurrentUserId()` returning `"me"`.
  - `SavedStateHandle(mapOf("conversationId" to "c1", "partnerId" to "bob", "partnerName" to "Bob"))`.

- [ ] **Step 1: Write the tests.**
  - **Loading:**
    - `messages from the repository land in state and loading clears`
    - `first unread partner message becomes the unread anchor and stays fixed`: a later emission
      with a new unread message does not move `firstUnreadMessageId`.
  - **Sending:**
    - `sending text sends the reply target and then clears it`: `sent.single().replyToMessageId`
      is the reply id, and `replyingTo == null`.
    - `pending attachments are sent as one media message instead of text`: `sentMedia` has one
      entry with the trimmed caption, `sent` is empty, and `pendingAttachments` is empty.
    - `adding the same attachment twice keeps one`
  - **Read receipts:**
    - `seen messages are marked read once each`: dispatch `MessagesSeen` twice with the same
      message; `markedRead.size == 1`.
    - `read receipts are skipped while the user id is blank`
  - **Jumping and paging:**
    - `jumping to a loaded message highlights it for 2500 ms`: `highlightedMessageId` is set, then
      `null` after `advanceTimeBy(2501)`.
    - `jumping to a message that never loads emits Original message not found`: set
      `loadOlderResult = success(0)`.
    - `older pages stop once a page returns fewer than 100`: `hasMoreOlder` is false after
      `success(40)`.
  - **Typing:**
    - `typing sets typing and clears it after 2 s of silence`: `setCount == 1`, then
      `clearCount == 1` after `advanceTimeBy(2001)`.
  - **Errors and recording:**
    - `delete failure emits ShowError`
    - `permanent mic denial shows the prompt; temporary denial emits an error`
    - `starting a recording while a draft exists does not start the recorder`
- [ ] **Step 2: Run** `GRADLE :app:testDebugUnitTest --tests "*ChatViewModelTest"`. Expected: PASS.
- [ ] **Step 3: Commit** `test: cover ChatViewModel send, read, jump, paging and typing`.

### Task 8: Chat list ViewModels (Chats, Forward, Calls)

**Files:**
- Create: `app/src/test/java/com/example/messenger/presentation/screens/chats/ChatsViewModelTest.kt`
- Create: `app/src/test/java/com/example/messenger/presentation/screens/forwardmessage/ForwardMessageViewModelTest.kt`
- Create: `app/src/test/java/com/example/messenger/presentation/screens/calls/CallsViewModelTest.kt`

**Interfaces:**
- Consumes: `FakePresenceService` (Task 7). `FirebaseAuth` is a MockK mock with
  `currentUser?.uid` returning `"me"`. `CallHistoryRepository` is a MockK mock whose
  `getCallHistory` returns a `MutableStateFlow`.

- [ ] **Step 1: Write the tests.**
  - `ChatsViewModelTest`:
    - **Loading:**
      - `signed-out user sees an empty list`
      - `conversations arrive and loading clears`
    - **Avatars:**
      - `partner avatars are seeded from the conversation, then fetched once per user`
      - `a failed avatar fetch is retried on the next emission`
    - **Actions:**
      - `delete for me failure surfaces the error`
      - `refresh sets isRefreshing during the sync and clears it after`: `syncCount == 1`.
  - `ForwardMessageViewModelTest`:
    - `closing search clears the query`
    - `list error is shown and loading clears`
  - `CallsViewModelTest`:
    - `entries show the contact alias over the stored name`
    - `missing avatars are fetched once per partner`
    - `opening calls backfills the 20 most recent conversations`: 25 conversations give 20
      `backfilled` entries, newest first.
- [ ] **Step 2: Run** `GRADLE :app:testDebugUnitTest --tests "*ChatsViewModelTest" --tests "*ForwardMessageViewModelTest" --tests "*CallsViewModelTest"`.
  Expected: PASS.
- [ ] **Step 3: Commit** `test: cover the chats, forward and calls list ViewModels`.

### Task 9: Phone-auth ViewModels (Auth, DeleteAccount, ChangePhone)

**Files:**
- Create: `app/src/test/java/com/example/messenger/presentation/screens/auth/AuthViewModelTest.kt`
- Create: `app/src/test/java/com/example/messenger/presentation/screens/settings/deleteaccount/DeleteAccountViewModelTest.kt`
- Create: `app/src/test/java/com/example/messenger/presentation/screens/changephone/ChangePhoneViewModelTest.kt`

**Interfaces:**
- Consumes:
  - MockK mocks for `FirebaseAuthService`, `PresenceManager`, `SyncCoordinator`,
    `FirebaseMessagingManager`, `FirestoreService`, `Activity` and `PhoneAuthCredential`.
  - `FakeAuthRepository` for the auth use cases.
  - `VerificationResult.CodeSent("vid")` and `VerificationResult.AutoVerified(credential)` as
    stubbed results.

- [ ] **Step 1: Write the tests.**
  - `AuthViewModelTest`:
    - **Code request:**
      - `code sent stores the verification id and shows the OTP step`
      - `failed code request shows the verification-failed error`
      - `wrong OTP shows the invalid-code error`: the error is
        `UiText.StringResource(R.string.auth_error_invalid_code)`.
    - **Session:**
      - `successful login goes online, restarts sync and emits AuthSucceeded`: verify
        `goOnline` and `restart`.
      - `account deleted elsewhere logs out and shows the account-deleted error`: the session
        emits `SignedOut(ACCOUNT_DELETED)` while `isAuthenticated()` is true.
      - `logout stops sync and resets state`
  - `DeleteAccountViewModelTest`:
    - **Before the code is sent:**
      - `sending the code before typing the confirmation shows an error and calls nothing`
      - `no phone number on file shows the no-number error`
      - `code sent moves to the verify step`
    - **Re-authentication and deletion:**
      - `failed re-authentication does not delete the account`
      - `successful deletion emits Deleted`
      - `go back returns to confirm and clears the code`
  - `ChangePhoneViewModelTest`:
    - **Verifying the current number:**
      - `blank current number shows the no-number-on-file error`
      - `verified current number moves to ENTER_NEW`
    - **Entering the new number:**
      - `non-digits are stripped from the new number`
      - `an invalid new number is rejected before any code is sent`
      - `a re-auth older than 3 minutes sends the user back to confirm`: seed
        `"change_phone_reauth_at"` with `now - 180_001`. Expect `CONFIRM_CURRENT` and the
        reauth-expired error.
    - **Persistence and navigation:**
      - `step survives process death`: seed `"change_phone_step" = "VERIFY_NEW"`.
      - `go back from VERIFY_NEW returns to ENTER_NEW`
- [ ] **Step 2: Run** `GRADLE :app:testDebugUnitTest --tests "*AuthViewModelTest" --tests "*DeleteAccountViewModelTest" --tests "*ChangePhoneViewModelTest"`.
  Expected: PASS.
- [ ] **Step 3: Commit** `test: cover the phone-auth, delete-account and change-phone flows`.

### Task 10: Edit and profile ViewModels, fixing #18

**Files:**
- Modify: `app/src/main/java/com/example/messenger/presentation/screens/editchat/EditChatViewModel.kt`
- Create in `app/src/test/java/com/example/messenger/presentation/screens/`:
  - `editchat/EditChatViewModelTest.kt`
  - `profile/ProfileViewModelTest.kt`
  - `editprofile/EditProfileViewModelTest.kt`
  - `chatuserprofile/ChatUserProfileViewModelTest.kt`
  - `chatuserprofile/SafetyNumberViewModelTest.kt`

**Interfaces:**
- `SafetyNumberViewModel` runs on `Dispatchers.IO`, so assert its state with Turbine
  (wall-clock wait), not virtual time. `IdentityKeyStore` is built the way
  `IdentityKeyStoreTest` builds it, over a temp directory. `PeerKeyRegistry` is a MockK mock.

- [ ] **Step 1: Write `EditChatViewModelTest`.**
  - `all conversations start selected`
  - `deselecting every chat survives a list update`:
    1. Emit `[a, b]`, then toggle both off.
    2. Emit `[a, b]` again. `selectedIds` stays empty.
  - `deleteSelected deletes only the selected ids, then emits Done`
- [ ] **Step 2: Run** `GRADLE :app:testDebugUnitTest --tests "*EditChatViewModelTest"`.
  Expected: FAIL in `deselecting every chat survives a list update`.
- [ ] **Step 3: Fix `EditChatViewModel`.** Seed the selection only on the first emission. Use a
  private `var seeded = false` set inside the collector, replacing the `selectedIds.isEmpty()`
  check.
- [ ] **Step 4: Run the same command.** Expected: PASS.
- [ ] **Step 5: Write the profile tests.**
  - `ProfileViewModelTest`:
    - `no signed-in user shows no profile`
    - `remote profile replaces the auth user`
    - `a blank username is rejected without saving`
    - `avatar upload updates the avatar and reloads photos`
  - `EditProfileViewModelTest`:
    - `profile loads once and is not reloaded after process death`: seed
      `"edit_profile_loaded" = true`; `getUserById` is never called.
    - `phone falls back to the auth number when the profile has none`
    - `a blank username shows an error`
    - `save writes username and usernameLower, then emits Saved`
  - `ChatUserProfileViewModelTest`:
    - `contact alias overrides the username`
    - `a deleted user is never shown online`
  - `SafetyNumberViewModelTest`:
    - `missing peer key shows unavailable`
    - `known peer key shows digits and the verified flag`
- [ ] **Step 6: Run** `GRADLE :app:testDebugUnitTest --tests "*presentation.screens*"`.
  Expected: PASS.
- [ ] **Step 7: Run `/verify`, then commit** in two commits:
  - `fix(editchat): keep the user's deselection when the chat list updates` with body `Fixes #18`
  - `test: cover the profile, edit-profile and safety-number ViewModels`

Out of scope for Phase 3, with reasons:
- `ChangeAccountViewModel`, `ClipPlayerViewModel`: one-line pass-throughs.
- `MediaPickerViewModel`: bound to MediaStore.
- `AppearanceViewModel`: bound to Context and DataStore. Revisit under Robolectric if needed.

---

## Phase 4: Screen tests under Robolectric, plus one device test

### Task 11: Robolectric Compose harness, proven on DeleteAccountScreenContent

**Files:**
- Modify: `gradle/libs.versions.toml` (add
  `junit-vintage-engine = { group = "org.junit.vintage", name = "junit-vintage-engine" }`)
- Modify: `app/build.gradle.kts`:
  - `testRuntimeOnly(libs.junit.vintage.engine)`
  - `testImplementation(libs.junit)`
  - `testImplementation(platform(libs.androidx.compose.bom))`
  - `testImplementation(libs.androidx.ui.test.junit4)`
- Create: `app/src/test/resources/robolectric.properties` containing `sdk=35` and
  `application=android.app.Application`
- Create: `app/src/test/java/com/example/messenger/presentation/screens/settings/deleteaccount/DeleteAccountScreenContentTest.kt`

**Interfaces:**
- Produces the screen-test pattern: `@RunWith(RobolectricTestRunner::class)`, a
  `@get:Rule val compose = createComposeRule()`, and content wrapped in the app theme
  composable that the existing contrast tests use. Find text with
  `ApplicationProvider.getApplicationContext<Context>().getString(R.string...)`.
  `application=android.app.Application` keeps Hilt and Firebase out of the sandbox.

- [ ] **Step 1: Write the tests.**
  - `an error in state is shown`: state error is
    `UiText.StringResource(R.string.delete_account_error_confirm)`, and its text is displayed.
  - `the verify step calls onVerify`: state `step = VERIFY`; click the verify action;
    `onVerify` is called once.
  - `the confirm step calls onStartDeletion`
- [ ] **Step 2: Run** `GRADLE :app:testDebugUnitTest --tests "*DeleteAccountScreenContentTest"`.
  Expected: PASS. If it fails:
  - **Sandbox fails on SDK 35:** bump Robolectric to the newest 4.x and remove `sdk=35`.
  - **OutOfMemoryError:** raise `test.maxHeapSize` to `"1g"`.
  - **Neither fix works:** stop and report. The fallback is to move Phase 4 to `androidTest`,
    run it by adb install and `am instrument`, and accept that CI will not run it.
- [ ] **Step 3: Run** `GRADLE :app:testDebugUnitTest`. Expected: every JUnit 5 test still runs,
  so the total equals the previous count plus 3.
- [ ] **Step 4: Commit** `test: run Compose screen tests on the JVM with Robolectric`.

### Task 12: Auth and chat screens

**Files:**
- Create: `app/src/test/java/com/example/messenger/presentation/screens/auth/AuthScreenContentTest.kt`
- Create: `app/src/test/java/com/example/messenger/presentation/screens/chat/ChatScreenContentTest.kt`

**Interfaces:**
- `ChatScreenContentTest` has a private `render(state: ChatUiState, messageText: String = "",
  onSendClick: () -> Unit = {}, onClearReply: () -> Unit = {})` helper. It passes no-op lambdas
  for every other parameter and `mutableStateOf(ClipPlaybackState())` for `clipPlayback`.

- [ ] **Step 1: Write the tests.**
  - `AuthScreenContentTest`:
    - `before the code is sent, Send calls onSendOtp`
    - `after the code is sent, Verify calls onVerifyOtp`
    - `an error is shown`
    - `loading disables the send action`
  - `ChatScreenContentTest`:
    - `text messages are shown`
    - `with text typed, the send action calls onSendClick`
    - `the reply banner shows the replied text, and closing it calls onClearReply`
- [ ] **Step 2: Run** `GRADLE :app:testDebugUnitTest --tests "*ScreenContentTest"`.
  Expected: PASS.
- [ ] **Step 3: Commit** `test: cover the auth and chat screens`.

### Task 13: List and edit screens

**Files:**
- Create in `app/src/test/java/com/example/messenger/presentation/screens/`:
  - `chats/ChatsTabContentTest.kt`
  - `editchat/EditChatScreenContentTest.kt`
  - `editprofile/EditProfileScreenContentTest.kt`

- [ ] **Step 1: Write the tests.**
  - `ChatsTabContentTest`:
    - `each row shows the partner name and last message`
    - `tapping a row calls onChatClick with its conversation id`
  - `EditChatScreenContentTest`:
    - `tapping a row calls onToggle with its id`
    - `delete is disabled with nothing selected`
    - `delete calls onDelete`
  - `EditProfileScreenContentTest`:
    - `the blank-username error is shown`
    - `save calls onSaveClick`
- [ ] **Step 2: Run** `GRADLE :app:testDebugUnitTest --tests "*ContentTest"`. Expected: PASS.
- [ ] **Step 3: Commit** `test: cover the chats list, edit-chats and edit-profile screens`.

### Task 15: Keystore key wrapping on the tablet

**Files:**
- Create: `app/src/androidTest/java/com/example/messenger/data/crypto/KeystoreKeyWrapperTest.kt`

**Interfaces:**
- Consumes: `KeystoreKeyWrapper()`, the production class.
- Safety: the wrapper uses the app's real master-key alias `e2ee_master_v1`, and the test runs in
  the app's process. The test only calls `wrap` and `unwrap`. It never deletes, replaces or
  regenerates a Keystore entry, so the real E2EE keys stay untouched.

- [ ] **Step 1: Write the tests.** JUnit 4, `@RunWith(AndroidJUnit4::class)`.
  - `a wrapped key unwraps to the same bytes`: 32 random bytes.
  - `a new wrapper instance unwraps what another wrapped`: simulates an app restart.
  - `two wraps of the same key differ`: a fresh IV per wrap.
  - `a tampered wrapped key is rejected`: flip one byte after the 12-byte IV;
    expect `javax.crypto.AEADBadTagException`.
- [ ] **Step 2: Build** with `GRADLE :app:assembleDebug :app:assembleDebugAndroidTest`.
  Expected: BUILD SUCCESSFUL.
- [ ] **Step 3: Run on the tablet.**
  1. `adb -s HVA6F12F install -r app/build/outputs/apk/debug/app-debug.apk`
  2. `adb -s HVA6F12F install -r app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk`
  3. `adb -s HVA6F12F shell am instrument -w -r -e class com.example.messenger.data.crypto.KeystoreKeyWrapperTest com.example.messenger.test/androidx.test.runner.AndroidJUnitRunner`

  Expected: `OK (4 tests)`. If the tablet is not connected, record it in the ledger, and the
  owner runs this step.
- [ ] **Step 4: Commit** `test: check Keystore key wrapping on a real device`.

---

## Phase 5: Coverage gate

### Task 14: Fail CI when coverage drops

**Files:**
- Modify: `app/build.gradle.kts` (`kover { reports { verify { rule { minBound(N) } } } }`)
- Modify: `.github/workflows/blank.yml` (run `koverVerifyDebug` after the unit tests)

- [ ] **Step 1: Measure.** Run `GRADLE :app:koverHtmlReportDebug` and read the line coverage.
  Set `N` to that percentage rounded down, minus 2.
- [ ] **Step 2: Run** `GRADLE :app:koverVerifyDebug`. Expected: BUILD SUCCESSFUL.
- [ ] **Step 3: Check the gate bites.** Temporarily set `N` to 99 and run again. Expected: FAIL
  naming the rule. Then restore `N`.
- [ ] **Step 4: Commit** `ci: fail the build when line coverage drops below the gate`. Put the
  baseline from Task 1 and the final number in the PR description, which closes #21.

---

## Release note

Everything here merges into `develop`. While the default branch is `main`, finished phases reach
the contribution graph only through the usual release (fast-forward or merge commit, never
squash).
