# E2E Tests

End-to-end tests that drive the real app on the tablet against the Firebase emulators on this
computer. They were added in PR #79 on `ai_encrypt` (merged 2026-11-08) after a real E2EE bug: after a
reinstall, your own messages showed "Message cannot be decrypted on this device" while your
partner could read everything. This note covers why they're built the way they are. How to run
them is in `app/src/test/RUNNING_TESTS.md` and in [[Build, CI and Releases]]. Paths below are
repo-root relative.

## Why they work this way

- **Device only.** The development PC is short on memory (`app/src/test/RUNNING_TESTS.md` says it
  cannot hold Android Studio and a Gradle build at once), so local runs use a physical tablet. The
  serial defaults to `HVA6F12F` and can be overridden with `E2E_SERIAL` (`e2e/run.sh:7`).
- **A separate app.** The `e2e` build type installs as `com.example.messenger.e2e`, beside the
  real app. `testBuildType = "e2e"`, so no instrumented test can uninstall the real app or wipe its
  E2EE keys.
- **No production access.** `src/e2e/google-services.json` is a fake `demo-messenger` project.
  A ContentProvider in `src/e2e` points Auth, Firestore and Realtime Database at `127.0.0.1`. Its
  network security config allows cleartext only to `127.0.0.1`. None of this ships, because it
  exists only in the `e2e` source set.
- **Undeployed rules are tested too.** The emulators load the branch's own `firestore.rules`.
- **UiAutomator, not the Compose test rule.** The Compose rule's `waitForIdle` can hang on
  infinite animations, and the chat skeleton and typing dots both have them. The app sets
  `testTagsAsResourceId`, and a few fields carry test tags.
- **A real partner without a second phone.** `HeadlessClient` runs the app's own
  `IdentityKeyStore`, `PeerKeyRegistry`, `E2eeMessageCodec` and `FirestoreService` on a second
  `FirebaseApp` inside the test process. Both sides of every chat use the real crypto code.
- **Real reinstalls.** A test runs inside the app's own process, so it can't uninstall the app.
  `e2e/run.sh` runs `phase1`, then `adb uninstall` and `adb install`, then `phase2`. The partner's key
  file survives the uninstall in an `e2eFixtures/partner` document in the emulator.

## Scenarios

| Scenario | Checks |
|---|---|
| `signin` | Registering by phone reaches the chats list with a published key |
| `messages` | Messages decrypt both ways, and Firestore holds only ciphertext |
| `reinstall-blockstore` | Block Store restores the key after a reinstall, with no prompt |
| `reinstall-code` | A recovery code restores every message after a reinstall |
| `reinstall-fresh` | Start fresh locks old messages, and new ones read both ways |
| `stale-key` | Fails until issue #78 is fixed, so `all` leaves it out |

Every reinstall check reports `partner read x/y, app read x/y, stored encrypted x/y`
(`app/src/androidTest/.../e2e/ReinstallScenario.kt:83`).

An earlier draft of this note also listed a `second-device` scenario. It is not in `e2e/run.sh`
(`ALL=(...)` at `e2e/run.sh:15`, plus `stale-key`) and there is no such test under
`app/src/androidTest/` at release 0.2.2.4, so it is left out here.

## What they found

- Block Store does restore the key after a reinstall on the tablet.
- Issue #78: right after a Start fresh, the partner's 30-minute key cache sends new messages to a
  key nobody holds.
- The merge with develop would have hidden Settings > Security in release builds. That row is
  the only lasting way to reach the recovery code screen. Fixed by a separate
  `settings_security_placeholders` flag.
- To prove the tests catch the original bug, the app was temporarily broken three ways:
  bringing back develop's key fork, turning off Block Store restore, and making its own
  messages unreadable. Each time the matching scenario failed.

## Traps

- The tablet locks during long builds, and `run.sh` then stops with "device is locked". Turning
  on Stay awake in Developer options avoids it.
- Block Store survives uninstall, so every scenario clears it first (`clearBlockStore`,
  `app/src/androidTest/.../e2e/E2eRules.kt:11`). The 16-entry limit quoted in the first version of
  this note comes from the Block Store documentation cited in
  [[Multi-Device Encryption (Research)]]; it is not checkable from the source.
- Don't touch the tablet during a run. A stray tap once opened Telegram mid-test.

## Follow-ups

#80 (run in CI), #81 (upgrade between releases), #82 (FLAG_SECURE on the recovery code screen).
The issue tracker is not part of the repository, so these numbers could not be re-checked; the
CI workflows at 0.2.2.4 do not run the end-to-end scenarios (`.github/workflows/blank.yml`,
`release.yml`).

## Related

[[Multi-Device Encryption (Research)]] - [[Tech Stack and Versions]] - [[Build, CI and Releases]] -
[[End-to-End Encryption]]
