# Multi-Device Encryption (Research)

Status: research and proposal, nothing implemented. Date: 2026-11-01. Based on the `ai_encrypt`
branch at `1332cf5` (recovery code, recovery vault, Block Store backup). Builds on
[[End-to-End Encryption]].

## Summary

- Message encryption is static. The conversation key is `HKDF(X25519(myIdentity, peerIdentity), uids, epochs)`, with no ratchet and no per-device state. Any device that holds the user's identity private key can therefore decrypt every message, past and future, directly from Firestore. Sharing one account key across devices needs no change to the message format, the message rules, or the push decrypt path, and history comes for free.
- Multi-device half-works today. A second phone that enters the recovery code gets the same identity, and both phones work side by side. Four things are missing:
  1. A second sign-in without a recovery vault silently forks the identity and cuts the first phone off from incoming messages.
  2. Push reaches only the last-signed-in device, and logging out on one device kills push on the others.
  3. There is no way to link a device without the code, no device list, and no revocation.
  4. Rotating keeps only the newest private key, so revoking by rotation would cost history.
- **Recommendation: Approach C.** Keep one account key (today's identity, extended to a keyring of all epochs). Give each device its own X25519 *device key* that is used only to deliver the account keyring. A device joins either by a QR scan approved on an existing device, or with the recovery code. Add a device list. Revoke by rotating the epoch and re-wrapping the new keyring for the remaining devices. Signal uses the same split for linked devices (a shared identity key plus per-device keys), but we can skip Signal's per-device sessions because we have no ratchet.
- **Not recommended now:**
  - Per-device fan-out (B) rewrites the DTO, push, rules and codec, and breaks the "new device downloads history from Firestore" model. It buys little until forward secrecy is a goal.
  - MLS (D) is built for groups and by design gives new devices no history (RFC 9750 §6.7). Its maintained libraries are written in Rust.

---

## 1. What the code does today (facts that drive the design)

| Area | Fact | Where |
|---|---|---|
| Key derivation | Static-static X25519 → HKDF-SHA256. Salt is `SHA-256(loUid\|hiUid)`, info is `msgr-e2ee-v1\|loEpoch\|hiEpoch`. Encryption is AES-GCM with a random nonce. Every message between two users in one epoch pair uses the same key, so there is no forward secrecy or post-compromise security. | `ConversationCrypto.deriveConversationKey` |
| Encrypt | One ciphertext per message, to the partner's latest epoch. That epoch comes from `ensurePeerKey`, which caches it for 30 min (`REFRESH_MS`). The DTO carries `senderEpoch`, `recipientEpoch`, `recipientId`. | `E2eeMessageCodec.encode`, `PeerKeyRegistry.ensurePeerKey` |
| Media | File keys for media and clips (and waveform, blurhash) travel inside the encrypted `MessagePayload`. Whoever can read the message can read the media, so per-device media keys aren't a separate problem. | `MessagePayload`, `MediaSecret`, `ClipSecret` |
| Decrypt | `check(identity.epoch == myEpoch)`. The store keeps only the current epoch; `rotate()` overwrites the file, so messages to older epochs become `LOCKED_TEXT`. | `E2eeMessageCodec.decryptPayload`, `IdentityKeyStore.persist` |
| History | Firestore keeps every message: rules disallow hard delete. The ObjectBox cache can be re-downloaded, and `UndecryptableCachePurge` relies on that. **History sync for a new device = download and decrypt**, as long as it holds the right private keys. | `firestore.rules` messages block |
| Silent fork | `resolveLocked` falls through to `identityKeyStore.create(uid)` when there is no local key, no Block Store copy and no vault. `publishIfNeeded` then finds epoch 1 taken and rotates above the server's maximum. The new device ends up at epoch N+1. Partners encrypt to N+1, but the first device holds only N, so its incoming messages turn undecryptable. `placeholderFor` shows `UNDECRYPTABLE_TEXT`, not `LOCKED_TEXT`, so the user gets no hint, and the first device is never told. `startFresh` on any device causes the same outcome. | `E2eeBootstrap.resolveLocked`, `PeerKeyRegistry.publishIfNeeded` |
| Block Store | Restore-only. Data moves to a new phone during setup (device-to-device or cloud restore), not live to a phone that is already set up. Limits: 16 entries, 4 KB each [BlockStore]. It never helps a second device that is already running. | `BlockStoreIdentityBackup` |
| Push | A single `users/{uid}.fcmToken` field. Logout deletes it (`FirestoreService` line 125). `onMessageCreated` and `onCallSignalCreated` send to that one token, so only the last-signed-in device gets message or call pushes, and logging out on one device silences the others. The users doc is readable by any signed-in user (`allow read: if isAuthenticated()`), so every FCM token is too. | `FcmTokenSyncer`, `functions/src/index.ts` |
| Push payload | Carries the ciphertext when it is 2800 characters or less, so `MyFirebaseMessagingService` can decrypt it for the notification. The FCM data limit is 4096 bytes [FCM-types]. `sendEachForMulticast` sends the same payload to every token; per-token payloads need `sendEach` [FCM-send]. | `onMessageCreated` |
| Safety number | Computed from uid plus the *current epoch's* public key, so it changes on every rotation. `PeerKeyRegistry.store` resets `verified` when the key changes, and a new epoch row starts unverified. No UI warns when a contact's key changes. | `SafetyNumber`, `SafetyNumberViewModel` |
| Rules | `users/{uid}/keys/{epoch}`: only the owner can create, and `pub` is immutable. `recovery/identity`: owner only, `wraps` 1–4. | `firestore.rules` |

---

## 2. How Signal, WhatsApp, Matrix and iMessage do it

### Signal
- **Linking:** the new device creates a temporary provisioning address and a Curve25519 key pair, and shows both as a QR code. The primary phone scans it, after the user authenticates, and sends a provisioning message encrypted to that public key. The message carries the shared keys, account info and a one-time linking token [Signal-sync].
- **Keys:** linked devices share one identity key pair, but each device has its own prekeys and sessions, and senders encrypt one copy per device [Signal-sync]. In Sesame, the sender encrypts to every non-stale device of each recipient, *including the sender's own other devices* [Sesame §3.1, §3.3]. If the device list is out of date, the server rejects the send and returns old and new DeviceIDs plus the new devices' identity keys, and the sender retries [Sesame §3.3 steps 4–8]. Sesame supports both per-user and per-device identity keys [Sesame §3.1].
- **History:** optional, chosen when linking. The primary uploads one archive encrypted with a one-time AES-256 key, and that key travels in the provisioning message. The archive includes attachment pointers for the last 45 days of media [Signal-sync].
- **Server-added devices:** Sesame's only defence is users verifying keys for all devices [Sesame §6.1]. X3DH requires each signed prekey to be signed by the identity key [X3DH §3.2, §4.5]. So with a shared identity key, the server can't make up a device that passes signature checks.

### WhatsApp
- **Keys:** each device has its own identity key, and the server maps each account to all its device identities [WA-md, "Meeting the security challenges of multiple devices"].
- **Linking:** by QR code, with biometric confirmation where enabled. "Automatic Device Verification" means contacts re-compare security codes only when the account re-registers, not each time a device is linked [WA-md].
- **Fan-out:** a pairwise session from each sender device to each recipient device, plus copies to the sender's own other devices [WA-md, "Maintaining message privacy"].
- **History:** the phone sends an encrypted bundle of recent chats, with its key delivered in an end-to-end encrypted message. App state is stored encrypted on the server [WA-md, "Keeping message history … in sync"].
- **Devices:** remote logout for any device [WA-md].
- **Key transparency:** an Auditable Key Directory, an append-only log of account→key mappings with public audit proofs, protects against the server handing out different keys [WA-kt].
- **Unverified:** the whitepaper's Account and Device Signature details (the primary signs each companion and a signed device list). The whitepaper PDF [WA-wp] downloaded but couldn't be parsed here, so these come from memory.

### Matrix (Element)
- **Device keys:** each device has an Ed25519 signing key and a Curve25519 identity key, plus one-time and fallback keys. Olm runs per device pair [Matrix-e2ee, "Device keys", "m.olm.v1…"].
- **Room keys:** Megolm uses one outbound session per sender device per room, and the session key goes to each recipient device over Olm. The ratchet only moves forward, so a device can't read anything before the index it received [Megolm].
- **Cross-signing:** the master key signs the self-signing and user-signing keys. The self-signing key signs the user's own devices, and the user-signing key signs other users' master keys. A new device becomes trusted once the self-signing key signs it, and contacts don't re-verify [Matrix-e2ee, "Cross-signing"].
- **History for new devices:** room-key requests answered only by the user's *verified* own devices, server-side key backup encrypted to a backup public key, and Secret Storage (SSSS). SSSS holds the cross-signing private keys and the backup key, unlocked by a recovery key or a PBKDF2 passphrase. Secrets are shared only with verified own devices [Matrix-e2ee, "Sharing keys between devices", "Server-side key backups"; Matrix-secrets].
- **Defences:** because server admins can create devices, clients SHOULD NOT send keys to devices that aren't cross-signed. Clients must also rotate the Megolm session when a device leaves [Matrix-e2ee, "Recommended client behaviour", "m.megolm.v1.aes-sha2"].

### Apple iMessage
- **Per-device keys:** each device registers its own keys with IDS. The sender fetches every device's keys and wraps a per-message key for each device [Apple-imsg, Apple-send].
- **New-device alerts:** all of a user's devices show an alert when a new device is added [Apple-imsg].
- **Contact Key Verification:** an account-level ECDSA key, kept in iCloud Keychain, signs every device's keys. A CONIKS-style Key Transparency log is checked on the device, with log hashes gossiped inside messages. Contacts compare SAS codes over the *account* key, so the code stays the same when devices are added [Apple-ckv].
- **PQ3:** each device pair gets a session with Kyber + ECDH, rekeyed post-quantum about every 50 messages and at least every 7 days [Apple-pq3].

### What all four share
1. A **stable account-level identity** that contacts verify once: Signal's shared identity key, Matrix's master key, Apple's account key, WhatsApp's primary identity with ADV.
2. **New devices get in only through an existing trusted device** (QR or SAS) **or a recovery secret**: Signal's backup key, Matrix's SSSS recovery key, Apple's keychain escrow, WhatsApp's backup key. The server alone can never add one.
3. **Adding or removing devices doesn't make contacts re-verify.** Only an account reset does.

---

## 3. The four options for this app

### A. Copy the shared identity as-is (today, plus fixes)
Every device holds the same identity, delivered by the recovery code (already built), by Block Store (new-phone restore only), or by a QR handoff.
- **Pros:** no message-format change, and history is free.
- **Cons:**
  - No device list, so no revocation.
  - Without QR, the recovery code is the only way to link.
  - A leaked code makes a silent, permanent reader until a reset.
- **Size:** S on its own, but it's a dead end without C's device list.

### B. Per-device keys with fan-out (Sesame / WhatsApp / iMessage style)
- **Changes:**
  - **DTO:** encrypt the payload once under a random message key, then add `keys: {deviceId: wrappedKey}` for every device of both users. This replaces `senderEpoch`/`recipientEpoch`.
  - **Rules:** validate the size of the `keys` map, and optionally require a matching `deviceListVersion` via `get()`. That would be our version of Sesame's server-side device-list check.
  - **Functions:** switch to `sendEach` with a per-device payload, under the 4096-byte limit.
  - **Registry:** `PeerKeyRegistry` becomes a device registry that tracks every contact's device list, detects changes and retries stale sends.
- **History:** a new device can read nothing older unless every old message key is re-wrapped for it, or a Signal-style archive is uploaded. That breaks the Firestore-as-history model the app relies on, including `UndecryptableCachePurge`'s re-download.
- **Safety numbers:** must cover every device (WhatsApp combines them) or needs cross-signing.
- **Pros:**
  - Revocation is just "stop wrapping for that device".
  - Device private keys never leave the device.
  - A natural base for a ratchet later.
- **Cons:**
  - Each message grows by about 100 B per device.
  - The server-added-device attack becomes real unless device lists are signed [Sesame §6.1].
- **Size:** XL.

### C. Account key plus per-device wrapping (recommended)
- **What changes:** the message crypto stays exactly as it is. Each device gets an X25519 device key and an entry in the device list. The account keyring is sealed to each device key by another trusted device, or opened with the recovery code. Revocation rotates the epoch and re-wraps the keyring for the remaining devices.
- **Pros:**
  - Each step is small.
  - History works as it does today.
  - Contacts are unaffected when devices change.
  - The server can't add a reader without a wrap from a trusted device or the code. A device row with no wrap is harmless.
- **Cons:**
  - The account private key lives on several devices. Compromising one exposes everything until the key is rotated.
  - Revocation needs a rotation.
  - Partners see a new epoch late unless they're told.
  - Still no forward secrecy.
- **Size:** L in total, spread across phases.

### D. MLS (RFC 9420)
- **Model:** each leaf is a *client* (device), not a user, and the app has to keep users and clients in sync [RFC 9750 §3.7, §6.1].
- **Delivery Service:** must settle on exactly one Commit per epoch [RFC 9750 §5.2, §8.1.4]. A Firestore transaction on a group-epoch document could do that.
- **History:** a new device gets none by design, and restoring it outside MLS weakens FS and PCS [RFC 9750 §6.7].
- **Fit:** MLS covers groups of two clients [RFC 9750 §1], but its value is in large groups.
- **Libraries:** all Rust. OpenMLS (MIT), mls-rs (Apache-2.0/MIT, conformance-tested but not fully audited), and Wire core-crypto (GPL-3.0, with JVM and Android bindings). Every option adds an NDK toolchain and APK weight, and the APK is already heavy from Agora.
- **Verdict:** not realistic now. Revisit if E2EE group chats become a goal. The rules allow 50 participants, but the codec is 1:1 only through `partnerOf`.

---

## 4. Approach C mapped onto this codebase

### Firestore schema and rules
- `users/{uid}/devices/{deviceId}`: `{name, model, appVersion, createdAt, lastSeenAt, devicePub, fcmToken, status: active|revoked}`. **Read and write by the owner only**, so tokens stop being public.
- `users/{uid}/devices/{deviceId}/wraps/{epoch}`: `{fromDeviceId, epk, nonce, ct, version}`, the keyring sealed to that device's `devicePub`. Owner only.
- `users/{uid}.keyEpoch`: the current epoch, public. It lets partners and rules spot staleness with one read.
- `users/{uid}/keys/{epoch}`: unchanged for now. Phase 4 adds `sig`.
- Message `create`, optional: require `senderEpoch == get(users/senderId).keyEpoch` (a revoked device can no longer send readable messages) and `recipientEpoch == get(users/recipientId).keyEpoch` (a stale sender fails fast and refreshes). This costs 1–2 extra document reads per message write.
- `deleteAccount`: also recursively delete `devices`.

### Identity keyring (`IdentityKeyStore`, `IdentityKeyCodec`)
- Store `Map<epoch, IdentityKey>` instead of a single key. Codec v2 is `[2][count][entries…]`, and v1 files decode as a one-entry ring.
- Block Store allows 4 KB per entry and each epoch takes 69 B, so about 55 epochs fit. Prune or cap the ring, and say what happens past the cap.
- `decryptPayload`: look up `ring[myEpoch]` instead of `check(current)`. `placeholderFor` returns LOCKED only when that epoch is missing from the ring.

### Recovery vault
- Same document and same `kind: "code"` wrap, but the plaintext becomes the v2 ring and `version` goes to 2. v1 documents still open.
- `rewrapVault` already handles epoch drift. Also call it after a revocation.

### `E2eeBootstrap` states
- **Fork fix (most important):** in `resolveLocked`, call `create()` only when `publishedKeysOnServer(uid)` is empty, meaning a genuinely new account. Otherwise go to a link state.
- `NeedsRestore` becomes **`NeedsLink(hasVault)`**, with three choices: show a QR to scan from another device, enter the recovery code, or reset.
- New **`Behind`** state: the local newest epoch is below `keyEpoch` and no wrap has arrived. This is how a device that was cut off finds out. Normally a listener on `devices/{me}/wraps` upgrades the ring automatically after a rotation, with no user action.
- New **`Revoked`** state: wipe the ring and the recovery code (`forget`), then sign out.

### Linking (the new device shows the QR, as in Signal and WhatsApp)
1. The new device signs in with phone OTP, generates its device key, writes `devices/{id}` with no wrap, and shows a QR code containing `{v, uid, deviceId, devicePub}`.
2. On an existing device: Security → Linked devices → Link. The user confirms with BiometricPrompt or the screen lock, then scans with CameraX (already a dependency) plus a decoder, either ML Kit barcode or ZXing.
3. The scanner checks that the uid matches and that `devices/{deviceId}.devicePub` equals the value in the QR. A server swapping the key gets caught here, because the QR carries the key outside the server.
4. It seals the ring to `devicePub` with an ephemeral X25519 key, HKDF and AES-GCM, using `uid|deviceId|v` as AAD, and writes `wraps/{epoch}`.
5. The new device opens the wrap, installs the ring, purges the placeholder cache, and is Ready. History arrives by the existing Firestore download.
6. Every other device shows "New device linked: <model>", as Apple does.

### Revocation and reset
- **Revoke:** done from any active device.
  1. Set `status = revoked` and delete the device's wraps.
  2. Rotate: the new epoch is the server maximum plus 1. Publish `keys/{new}` and set `keyEpoch`.
  3. Seal the new ring to every remaining active device.
  4. Re-wrap the vault. That needs the recovery code on this device (`RecoveryCodeStore`); if it isn't there, ask the user to re-enter it.

  The revoked device keeps old epochs, so it can still read what it could already read. Every system has that limit.
- **Revoked device's session:** Firebase Auth can't kill one device's session, because `revokeRefreshTokens` works per user. The `senderEpoch` rule stops it sending readable messages. For a stolen device, offer "sign out everywhere", which revokes all refresh tokens and makes every device sign in again.
- **`startFresh` becomes "Reset encryption":** it rotates, revokes every other device (they drop to `NeedsLink`), deletes the vault, and shows contacts a "security code changed" notice. The confirmation dialog lists the devices that will be cut off.

### Partners learning a new epoch
`ensurePeerKey` caches for 30 minutes. Either add the `recipientEpoch` rule (a stale write fails, and the codec refreshes and retries once) or listen to the partner's `users` doc for `keyEpoch`. The rule is simpler and also covers offline senders.

### Push
- `FcmTokenSyncer` writes `devices/{id}.fcmToken`, and logout deletes only that device's row.
- `onMessageCreated` and `onCallSignalCreated` read all active devices of each recipient. The ciphertext in the push stays one shared payload, so `sendEachForMulticast` still works.
- Keep reading the legacy `fcmToken` field as a fallback for one release.
- Out of scope: calls ringing on several devices need "answered elsewhere" handling.

### Safety numbers (`SafetyNumber`, `SafetyNumberViewModel`)
- **Phase 3, simple:** keep the number based on the epoch key. When a contact's epoch changes, insert a "security code changed" notice in the chat. For verified contacts, block sending until the user acknowledges it, as Sesame §6.1 and Matrix's recommended behaviour suggest.
- **Phase 4, better:** add a stable Ed25519 *account signing key* to the ring (Tink `Ed25519Sign`) and compute the safety number from it. Each new epoch's X25519 public key gets signed (`keys/{epoch}.sig`), and contacts check the signature before switching. A server can then no longer inject an epoch unnoticed, and routine revocations stop changing the number. A reset makes a new signing key, so the number changes and contacts are warned. One caveat: a revoked device also knows the signing key. If that device was *stolen*, revoking it should roll the signing key too, and the number changes. See the open questions.

### Migration
- **Messages:** none needed. The DTO and message crypto don't change, and old messages stay readable by any device with the right epoch key.
- **Key file, vault, Block Store:** v1 is read and v2 written on the next save. The rules already accept any int `version`.
- **Old app versions** still have the fork path. Ship the fork fix first, put the linked-devices UI behind the existing feature-flag setup, and consider a minimum-version check before advertising multi-device.
- **Already-forked users** (a first phone silently cut off) recover through the `Behind` state: enter the code or link.
- **`fcmToken`:** write both the old field and the new device row for one release. Functions read the device rows first and fall back to the old field.

---

## 5. Security notes

- **A thief linking a stolen phone** needs a Firebase sign-in, which is phone OTP and so exposed to SIM swaps, plus **either** the recovery code **or** an approved scan on an unlocked trusted device. Without those, all they can do is Reset, and with these changes a reset is loud: the other devices are cut off and contacts are warned. Today a reset is silent, which is the first thing to fix.
- **A compromised linked device** holds the whole ring and can read everything until it's revoked. That's inherent to any shared-key design, including Signal's linked devices. The mitigations are a visible device list with `lastSeenAt`, new-device alerts on every other device, and revoke-with-rotate.
- **A server adding a device:**
  - A device row with no wrap gains nothing, because the server can't produce a wrap. That's a structural advantage of C over B.
  - Swapping `devicePub` during linking is caught by the key carried in the QR.
  - Publishing a fake epoch to contacts is caught today only by manual safety numbers. Phase 4 signatures catch it automatically.
  - A split view (different keys shown to different people) is only fully solved by key transparency (WhatsApp AKD, Apple CKV), which is out of proportion for this app.
  - Firebase project admins count as "the server" in all of this.
- **Revocation** can't erase the past. The `senderEpoch` rule and "sign out everywhere" cover the future.
- **The recovery code is now a key to every device.** Treat it like Signal's backup key or Matrix's SSSS recovery key.
- **No forward secrecy:** a leaked ring decrypts all history stored in Firestore, and multi-device multiplies the places the ring lives. Adding forward secrecy conflicts with "server holds readable history". Matrix and WhatsApp pay for that with key backups and history transfer. It's a separate product decision.

---

## 6. Phased plan

Sizes assume one developer: S is 2 days or less, M about a week, L about two weeks.

| Phase | Scope | Size |
|---|---|---|
| 0. Stop the damage | Fork fix (`create()` only when no keys are published). `Behind` detection plus a prompt to link or enter the code. Per-device push tokens in `devices/{id}`, read by the functions, with logout removing only its own row. Owner-only reads for tokens. | S–M |
| 1. Keyring | Multi-epoch `IdentityKeyStore`, codec v2, vault and Block Store v2, decrypt by epoch, migration tests. | M |
| 2. Link by QR | Device keys, device registration, show and scan QR, sealed wraps, the new option on `KeyRestoreScreen`, new-device alerts. | M–L |
| 3. Devices and revoke | Linked-devices screen, revoke with rotate and re-wrap (including the vault), wrap listener, epoch rules, "Reset encryption" rework, key-change notices in chats. | L |
| 4. Contact integrity | Account signing key, epoch signatures, safety number from the account key, block-until-acknowledged for verified contacts. | M |
| Later | Per-device fan-out or MLS, only if forward secrecy or E2EE groups become goals. A desktop or web client would reuse Phase 2. | XL |

Phase 0 is worth shipping even if the rest waits: it fixes behaviour that already exists in `ai_encrypt`.

---

## 7. Open questions for the owner

1. Is "every linked device reads all history" what you want? It's the natural result here. The alternative is Signal's opt-in history transfer.
2. Should any active device be able to approve a link (peer model, recommended), or only a designated primary phone (WhatsApp's original model)?
3. Should there be a maximum number of devices per account?
4. Safety numbers: keep them based on the epoch key (they change on every revoke), or move to a stable account signing key in Phase 4?
5. When a device is revoked as stolen, should every session be signed out and every device made to sign in again?
6. Are E2EE group chats planned? That changes the case for B or D.
7. Is forward secrecy a goal? It conflicts with Firestore keeping full readable history.
8. QR decoder: ML Kit barcode or ZXing?
9. Is 1–2 extra Firestore reads per message write acceptable for the epoch rules?

## 8. Problems found in passing (not fixed; issue candidates)

1. **Silent identity fork:** a second sign-in without a vault, or `startFresh`, cuts the other device off from incoming messages and shows the generic "cannot be decrypted" text. Fixed by Phase 0.
2. **FCM token readable by any signed-in user:** `users/{uid}.fcmToken` sits on a doc with `allow read: if isAuthenticated()`.
3. **Logout on one device deletes the shared `fcmToken`,** which silences push on the account's other devices.

---

## Sources

- [Sesame] Signal, "The Sesame Algorithm", rev. 2: https://signal.org/docs/specifications/sesame/ (§2.1, §3.1, §3.3, §4.1, §6.1)
- [X3DH] Signal, "The X3DH Key Agreement Protocol": https://signal.org/docs/specifications/x3dh/ (§3.2, §4.1, §4.5, §4.7)
- [PQXDH] Signal, "The PQXDH Key Agreement Protocol": https://signal.org/docs/specifications/pqxdh/ (§2.5, §3.3, §4.1)
- [Signal-sync] Signal blog, "A Synchronized Start for Linked Devices" (2025-01-27): https://signal.org/blog/a-synchronized-start-for-linked-devices/
- [Signal-backup] Signal blog, "Introducing Signal Secure Backups" (2025-09-08): https://signal.org/blog/introducing-secure-backups/
- [WA-md] Meta Engineering, "How WhatsApp enables multi-device capability" (2021-07-14): https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/
- [WA-kt] Meta Engineering, "Deploying key transparency at WhatsApp" (2023-04-13): https://engineering.fb.com/2023/04/13/security/whatsapp-key-transparency/
- [WA-bk] Meta Engineering, "How WhatsApp is enabling end-to-end encrypted backups" (2021-09-10): https://engineering.fb.com/2021/09/10/security/whatsapp-e2ee-backups/
- [WA-wp] WhatsApp Encryption Overview (whitepaper): https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf (downloaded, text not extracted in this pass)
- [Matrix-e2ee] Matrix spec source, End-to-End Encryption module: https://raw.githubusercontent.com/matrix-org/matrix-spec/main/content/client-server-api/modules/end_to_end_encryption.md (rendered at https://spec.matrix.org/latest/client-server-api/#end-to-end-encryption)
- [Matrix-secrets] Matrix spec source, Secrets module: https://raw.githubusercontent.com/matrix-org/matrix-spec/main/content/client-server-api/modules/secrets.md
- [Megolm] Matrix spec, Megolm: https://spec.matrix.org/v1.19/olm-megolm/megolm/ ; Olm: https://spec.matrix.org/v1.19/olm-megolm/olm/
- [RFC 9420] The Messaging Layer Security (MLS) Protocol: https://www.rfc-editor.org/rfc/rfc9420.html
- [RFC 9750] The MLS Architecture: https://www.rfc-editor.org/rfc/rfc9750.html (§1, §3.7, §5.2, §6.1, §6.7, §8.1.4, §8.2.4)
- [Apple-imsg] Apple Platform Security, "iMessage security overview": https://support.apple.com/guide/security/imessage-security-overview-secd9764312f/web
- [Apple-send] Apple Platform Security, "How iMessage sends and receives messages securely": https://support.apple.com/guide/security/how-imessage-sends-and-receives-messages-sec70e68c949/web
- [Apple-esc] Apple Platform Security, "Escrow security for iCloud Keychain": https://support.apple.com/guide/security/escrow-security-for-icloud-keychain-sec3e341e75d/web
- [Apple-ckv] Apple Security Research, "Advancing iMessage security: iMessage Contact Key Verification" (2023-10-27): https://security.apple.com/blog/imessage-contact-key-verification/
- [Apple-pq3] Apple Security Research, "iMessage with PQ3" (2024-02-21): https://security.apple.com/blog/imessage-pq3/
- [BlockStore] Android Developers, "Block Store": https://developer.android.com/identity/block-store
- [FCM-types] Firebase, "FCM message types": https://firebase.google.com/docs/cloud-messaging/customize-messages/set-message-type
- [FCM-send] Firebase, "Send a message using Firebase Admin SDK": https://firebase.google.com/docs/cloud-messaging/send-message
- Libraries: OpenMLS https://github.com/openmls/openmls · mls-rs https://github.com/awslabs/mls-rs · Wire core-crypto https://github.com/wireapp/core-crypto
