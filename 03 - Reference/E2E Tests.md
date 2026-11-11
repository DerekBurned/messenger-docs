# E2E Tests

End-to-end tests that drive the real app on the tablet against the Firebase emulators on this
computer. They were added in PR #79 on `ai_encrypt` (2026-11-07) after a real E2EE bug: after a
reinstall, your own messages showed "Message cannot be decrypted on this device" while your
partner could read everything. This note covers why they're built the way they are. How to run
them is in `app/src/test/RUNNING_TESTS.md`.

## Why they work this way

- **Device only.** The owner's rule is no Android emulator on this PC, which is short on memory.
  Local runs use the Lenovo tablet (serial `HVA6F12F`).
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
| `second-device` | A reset on another device prompts this one, and its code restores everything. Written, not yet run on the tablet |
| `stale-key` | Fails until issue #78 is fixed, so `all` leaves it out |

Every reinstall check reports `partner read x/y, app read x/y, stored encrypted x/y`.

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
- Block Store survives uninstall and holds at most 16 entries, so every scenario clears it first.
- Don't touch the tablet during a run. A stray tap once opened Telegram mid-test.

## Follow-ups

#80 (run in CI), #81 (upgrade between releases), #82 (FLAG_SECURE on the recovery code screen).

Related: [[Multi-Device Encryption (Research)]], [[Tech Stack and Versions]].
