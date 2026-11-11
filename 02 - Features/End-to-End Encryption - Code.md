# End-to-End Encryption - Code

Companion to [[End-to-End Encryption]]. The crypto source, block by block. Sections 1, 2, 5, 6 and the
safety-number block show the real code; sections 3, 4, 7 and 8 changed a lot with the keyring and
recovery work (PR #79) and are walked through in prose with minimal excerpts, so this vault does not
copy them. `path:line` refs are relative to `app/src/main/java/com/example/messenger/`.

Files covered (`data/crypto/`): `ConversationCrypto.kt`, `model/MessagePayload.kt`,
`model/MediaSecret.kt`, `model/ClipSecret.kt`, `CryptoEncoding.kt`, `E2eeMessageCodec.kt`,
`IdentityKeyStore.kt`, `IdentityKeyring.kt`, `KeyWrapper.kt`, `MediaCipher.kt`, `PeerKeyRegistry.kt`,
`E2eeBootstrap.kt`, `E2eeState.kt`, `SafetyNumber.kt`. Long expressions are sometimes joined onto one line.

---

## 1. ConversationCrypto - the primitives

`ConversationCrypto.kt` in full (X25519 ECDH, HKDF, AES-256-GCM):

```kotlin
class EncryptedBlob(val nonce: ByteArray, val ciphertext: ByteArray)

object ConversationCrypto {
    const val NONCE_SIZE = 12
    private const val TAG_BITS = 128
    private const val KEY_SIZE = 32
    private const val INFO_PREFIX = "msgr-e2ee-v1"

    fun generateKeyPair(): Pair<ByteArray, ByteArray> {
        val privateKey = X25519.generatePrivateKey()
        return privateKey to X25519.publicFromPrivate(privateKey)
    }

    fun deriveConversationKey(
        myPrivateKey: ByteArray, peerPublicKey: ByteArray,
        myUid: String, peerUid: String, myEpoch: Int, peerEpoch: Int,
    ): ByteArray {
        val shared = X25519.computeSharedSecret(myPrivateKey, peerPublicKey)
        val (loUid, hiUid) = if (myUid < peerUid) myUid to peerUid else peerUid to myUid
        val (loEpoch, hiEpoch) = if (myUid < peerUid) myEpoch to peerEpoch else peerEpoch to myEpoch
        val salt = MessageDigest.getInstance("SHA-256").digest("$loUid|$hiUid".toByteArray())
        return Hkdf.computeHkdf("HMACSHA256", shared, salt,
            "$INFO_PREFIX|$loEpoch|$hiEpoch".toByteArray(), KEY_SIZE)
    }

    fun encrypt(key: ByteArray, plaintext: ByteArray, aad: ByteArray): EncryptedBlob {
        val nonce = ByteArray(NONCE_SIZE).also { SecureRandom().nextBytes(it) }
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, SecretKeySpec(key, "AES"), GCMParameterSpec(TAG_BITS, nonce))
        cipher.updateAAD(aad)
        return EncryptedBlob(nonce, cipher.doFinal(plaintext))
    }

    fun decrypt(key: ByteArray, nonce: ByteArray, ciphertext: ByteArray, aad: ByteArray): ByteArray {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.DECRYPT_MODE, SecretKeySpec(key, "AES"), GCMParameterSpec(TAG_BITS, nonce))
        cipher.updateAAD(aad)
        return cipher.doFinal(ciphertext)
    }
}
```

Both sides compute the same key: `deriveConversationKey` runs ECDH then HKDF, with uids/epochs
sorted so salt and info are identical regardless of direction. AES-256-GCM with a random 12-byte
nonce, 128-bit tag, and AAD binding. Deep discussion in [[End-to-End Encryption]].

---

## 2. Payload types and helpers

The payload types, one per file in `data/crypto/model/`:

```kotlin
@Serializable
data class MessagePayload(
    val text: String = "",
    val replyToText: String? = null,
    val media: List<MediaSecret> = emptyList(),
    val clip: ClipSecret? = null,
)

@Serializable
data class MediaSecret(
    val itemId: String,
    val key: String,
    val blurHash: String = "",
)

@Serializable
data class ClipSecret(
    val key: String? = null,
    val waveform: List<Int> = emptyList(),
)
```

`CryptoEncoding.kt`:

```kotlin
fun messageAad(conversationId: String, messageId: String, senderId: String, type: String): ByteArray =
    "$conversationId|$messageId|$senderId|$type".toByteArray()

fun b64(bytes: ByteArray): String = java.util.Base64.getEncoder().encodeToString(bytes)

fun unb64(value: String): ByteArray = java.util.Base64.getDecoder().decode(value)
```

`MessagePayload` is what actually gets encrypted (the text, the reply snippet, per-item media
file keys as `MediaSecret`, and for a voice clip its file key and waveform as `ClipSecret`).
`messageAad` binds the ciphertext to the message identity; `b64`/`unb64` are the wire encoding for
keys/nonces/ciphertext.

---

## 3. E2eeMessageCodec - encode/decode a Message

`E2eeMessageCodec.kt` (200 lines). `isEncryptable` (`:32`) says whether a conversation has a resolvable
partner and peer key.

`encode` (`:38`):

1. Calls are returned as plaintext (`:40`). It notes whether the message already carries encrypted media
   (`hasEncryptedMedia`, a media item or clip with a `fileKey`, `:41`-`:45`).
2. Resolves `myUid`, the partner (`participantResolver.partnerOf`) and the peer key
   (`peerKeyRegistry.ensurePeerKey`); any miss goes to `plainOrFail` (`:46`-`:49`, `:87`), which throws
   when media is already encrypted rather than send stranded plaintext.
3. Takes the newest identity key with `identityKeyStore.require(myUid)` (`:50`) - sending always uses
   the newest epoch - and builds a `MessagePayload` per message type (`:51`-`:68`): text and reply snippet;
   caption plus one `MediaSecret(itemId, key, blurHash)` per encrypted item; or `clip.toSecret()`
   (`:119`).
4. Derives the conversation key (`ConversationCrypto.deriveConversationKey(identity.privateKey,
   peer.publicKey, myUid, partnerId, identity.epoch, peer.epoch)`, `:69`-`:70`), computes the AAD
   (`messageAad`, `:71`), encrypts the JSON payload, and returns the DTO with text, reply snippet,
   blurHashes and waveform cleared and `enc = 1`, `ciphertext`, `nonce`, `senderEpoch`,
   `recipientEpoch`, `recipientId` set (`:72`-`:85`).

`decode` (`:94`): `enc != 1` is plaintext. Otherwise `decryptPayload` (`:129`) picks the sender or
recipient side, then looks up this device's key **for the message's epoch**:

```kotlin
val identity = identityKeyStore.keyFor(myUid, myEpoch) ?: error("no local key for epoch $myEpoch")
```

and the partner's key for their epoch (`cachedPeerKey`, then `peerKeyForEpoch`), derives the key and
decrypts with the same AAD. On failure it tries `retryWithFreshPeerKey` (`:153`, at most once per
peer and epoch per 5 minutes, `KEY_REFRESH_THROTTLE_MS`), and then returns an `undecryptable` message
(`:179`) whose text comes from `placeholderFor` (`:168`): `UNDECRYPTABLE_TEXT`, `LOCKED_TEXT` ("Sent
before your encryption key was reset") or `BEHIND_TEXT` ("Sent to a newer encryption key that this
device doesn't have yet") (`:191`-`:195`). On success it puts the media `fileKey`s and blurHashes
(`:103`-`:112`) or the clip key and waveform (`:113`-`:114`) back onto the domain message.
`isPlaceholder` (`:197`) is used by `UndecryptableCachePurge` and by the FCM handler to avoid showing
placeholder text in a notification. Wired into `FirestoreService` (see [[Chat and Messaging - Code]])
and the FCM handler (see [[Notifications - Code]]).

---

## 4. IdentityKeyStore and IdentityKeyring

`IdentityKey(privateKey, publicKey, epoch)` (`data/crypto/model/IdentityKey.kt`) is one X25519 pair.
`IdentityKeyring` (`IdentityKeyring.kt:8`) holds one key per epoch, sorted by epoch: `newest` (`:17`) is
used for sending, `keyFor(epoch)` (`:19`) for decrypting, `with`/`without` return changed copies, and
`digest()` (`:29`) is a SHA-256 over the epochs and public keys (used by the recovery code store to know
when to re-wrap). `IdentityKeyring.merge` (`:46`) merges an incoming ring with the local one, per epoch:
the key the account published wins, a ring that contradicts a published local key yields `null`, and
epochs newer than anything published are dropped.

`IdentityKeyStore` (`IdentityKeyStore.kt:12`, built in `di/CryptoModule.kt:39`-`:42` over
`filesDir/e2ee`) persists the ring in `identity_v1_{uid}.bin`:

- `ring(uid)` / `current(uid)` / `keyFor(uid, epoch)` / `require(uid)` (`:21`-`:32`) read it (cached in
  memory); `require` throws `IdentityUnavailableException` (`:9`).
- `create(uid, epoch = 1)` (`:35`) and `addEpoch(uid, epoch)` (`:39`) generate a pair and persist;
  `install(uid, ring, published)` (`:46`) merges a restored ring; `discard` (`:53`) removes one epoch;
  `setAside` (`:64`) moves keys the account does not hold out of the ring (keeping a copy);
  `delete` (`:89`) and `quarantine` (`:97`) handle account deletion and unreadable files.
- `behindState` / `rememberBehind` / `forgetBehind` (`:70`-`:84`) persist the "behind" marker in
  `behind_v1_{uid}.bin`.
- `persist` (`:121`) wraps the encoded ring with `KeyWrapper`, writes a `.tmp` staging file and moves it
  into place atomically (`:121`-`:127`). A file from the old single-key format decodes as a one-key
  ring.

There is no `getOrCreate` or `rotate` any more: creating a key is `E2eeBootstrap`'s decision (section 8).

---

## 5. KeyWrapper - Android Keystore at-rest protection

`KeyWrapper.kt` in full:

```kotlin
interface KeyWrapper {
    fun wrap(plain: ByteArray): ByteArray
    fun unwrap(wrapped: ByteArray): ByteArray
}

class KeystoreKeyWrapper : KeyWrapper {
    private val generationLock = Any()

    private fun masterKey(): SecretKey {
        existingMasterKey()?.let { return it }
        return synchronized(generationLock) { existingMasterKey() ?: generateMasterKey() }
    }

    private fun existingMasterKey(): SecretKey? {
        val store = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
        return store.getKey(ALIAS, null) as? SecretKey
    }

    private fun generateMasterKey(): SecretKey {
        val generator = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
        generator.init(
            KeyGenParameterSpec.Builder(ALIAS, KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
                .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                .setKeySize(256).build())
        return generator.generateKey()
    }

    override fun wrap(plain: ByteArray): ByteArray {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, masterKey())
        return cipher.iv + cipher.doFinal(plain)
    }

    override fun unwrap(wrapped: ByteArray): ByteArray {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        val iv = wrapped.copyOfRange(0, 12)
        cipher.init(Cipher.DECRYPT_MODE, masterKey(), GCMParameterSpec(128, iv))
        return cipher.doFinal(wrapped.copyOfRange(12, wrapped.size))
    }

    companion object { private const val ALIAS = "e2ee_master_v1" }
}
```

A hardware-backed AES-256-GCM master key lives in the Android Keystore (alias `e2ee_master_v1`)
and never leaves it; `wrap`/`unwrap` prepend/consume the 12-byte IV. The double-checked
`synchronized` block avoids two threads generating the master key at once. This is what protects
the identity private key on disk.

---

## 6. MediaCipher - streaming file encryption

`MediaCipher.kt` in full:

```kotlin
object MediaCipher {
    private const val NONCE_SIZE = 12
    private const val TAG_BITS = 128

    fun generateFileKey(): ByteArray = ByteArray(32).also { SecureRandom().nextBytes(it) }

    fun encrypt(key: ByteArray, source: File, dest: File) {
        val nonce = ByteArray(NONCE_SIZE).also { SecureRandom().nextBytes(it) }
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, SecretKeySpec(key, "AES"), GCMParameterSpec(TAG_BITS, nonce))
        dest.outputStream().use { out ->
            out.write(nonce)
            CipherOutputStream(out, cipher).use { encrypted -> source.inputStream().use { it.copyTo(encrypted) } }
        }
    }

    fun decrypt(key: ByteArray, source: File, dest: File) {
        dest.parentFile?.mkdirs()
        val part = File.createTempFile(dest.name + ".", ".part", dest.parentFile)
        try {
            source.inputStream().use { input ->
                val nonce = ByteArray(NONCE_SIZE)
                check(input.read(nonce) == NONCE_SIZE) { "encrypted file too short" }
                val cipher = Cipher.getInstance("AES/GCM/NoPadding")
                cipher.init(Cipher.DECRYPT_MODE, SecretKeySpec(key, "AES"), GCMParameterSpec(TAG_BITS, nonce))
                part.outputStream().use { out ->
                    CipherInputStream(input, cipher).use { it.copyTo(out) }
                }
            }
            Files.move(part.toPath(), dest.toPath(), ATOMIC_MOVE, REPLACE_EXISTING)
        } catch (e: Exception) {
            part.delete()
            throw e
        }
    }
}
```

Per-file AES-256-GCM with the nonce prepended to the file, streamed via
`CipherInputStream`/`CipherOutputStream`. `decrypt` writes to a uniquely named `.part` file
(`createTempFile`, so two concurrent decrypts never share one) and moves it over the destination
with `ATOMIC_MOVE` on success; on failure it deletes only its own part, so an existing good copy
survives a failed decrypt. The per-file key is
generated in `MediaRepositoryImpl.uploadAndSend` / `ClipRepositoryImpl` and carried inside the
encrypted `MessagePayload` (see [[Chat and Messaging - Code]], [[Voice Messages - Code]]).

---

## 7. PeerKeyRegistry

`PeerKeyRegistry.kt` (177 lines). Keys live in Firestore `users/{uid}/keys/{epoch}` and are cached as
`ObxPeerKey` rows (unique on `"uid:epoch"`).

- `publishIfNeeded(myUid, identity, allowNewEpoch = false)` (`:31`) first runs the pure
  `precheckPublish` (`:167`), then a Firestore **transaction** (`:40`-`:59`) that creates
  `users/{uid}/keys/{epoch}` only if absent and returns `Published`, `AlreadyPublished` (same key) or
  `Conflict` (different key); `PERMISSION_DENIED` is also `Conflict` (`:60`-`:62`). It never overwrites
  and never rotates. `PublishResult` is at `:19`-`:23`.
- `publishedKeysOnServer(uid)` (`:82`) reads every published key from the server; `observeNewestKey(uid)`
  (`:65`) is a snapshot listener on the newest epoch.
- `ensurePeerKey` (`:101`) serves a fresh cached key (< 30 min, `REFRESH_MS`, `:164`) or fetches the
  latest epoch; `peerKeyForEpoch` (`:115`) fetches a specific historical epoch; `refreshPeerKey` (`:94`)
  bypasses the cache for the codec's one-shot decode retry; `cachedPeerKey` / `latestPeerKey`
  (`:123`, `:125`) are local-only.
- `store` (`:133`) upserts the cache row and clears the `verified` flag when the public key under the same
  epoch changes; `setVerified` (`:127`) is used by safety-number verification.

---

## 8. E2eeBootstrap and SafetyNumber

`E2eeState.kt` defines the state the app gates on: `E2eeState` (`:3`) is `Resolving`, `Ready`,
`NeedsLink(hasVault)`, `Behind(hasVault)` or `Offline`; `E2eeStatus` (`:11`) pairs it with a uid and
`stateFor(sessionUid)` ignores a status that belongs to another account; `RecoveryStatus` (`:15`) and
`RestoreResult` (`:21`) describe the recovery code and a restore attempt.

`E2eeBootstrap.kt` (446 lines, `@Singleton`, own `SupervisorJob` IO scope and a mutex):

- `refresh` (`:55`) resets the status when the signed-in uid changes and launches `resolve` (`:64`),
  which runs `resolveLocked` (`:181`) under the mutex; a failure becomes `Offline` and `scheduleRetry`
  (`:426`, `RETRY_DELAY_MS = 5_000`).
- `resolveLocked` decides: local ring -> `Ready` + background `sync`; else a Block Store ring that
  shares a key with the account is installed; else, if the account already has published keys or a
  recovery vault, `NeedsLink`; else create epoch 1 and publish it (a lost race discards the key).
- `sync` (`:248`) publishes the newest key if needed (a `Conflict` triggers `recheck`, `:303`), saves the
  ring to Block Store and refreshes the recovery status (`refreshRecoveryStatus`, `:340`, which re-wraps
  the vault via `rewrapVault`, `:375`).
- `watch` (`:269`) observes the account's newest key; `looksBehind` (`:298`) and `settle` (`:326`) set
  aside keys the account contradicts and move to `NeedsLink` or `Behind` after `BEHIND_GRACE_MS` (5 s).
- `restoreWithCode` (`:81`), `startFresh` (`:110`), `newRecoveryCode` (`:142`), `saveRecoveryCode`
  (`:144`), `savedRecoveryCode` (`:165`), `stop` (`:170`) and `forget` (`:174`) are the entry points used
  by the recovery view models and by `AndroidSessionCleaner`.

Called from `MessengerApplication.onCreate` (`MessengerApplication.kt:118`) and after every successful
sign-in (`data/repository/AuthRepositoryImpl.kt:69`, `:99`; see [[Authentication - Code]]). The gate that
shows `KeyRestoreScreen` until the state is `Ready` is in
`presentation/navigation/graph/AppNavigation.kt:43`-`:60`.

`SafetyNumber.kt` in full - the comparable fingerprint:

```kotlin
object SafetyNumber {
    private const val ITERATIONS = 5200
    private val MOD = BigInteger.TEN.pow(30)

    fun compute(uidA: String, publicKeyA: ByteArray, uidB: String, publicKeyB: ByteArray): String {
        val halfA = half(uidA, publicKeyA)
        val halfB = half(uidB, publicKeyB)
        val (first, second) = if (halfA < halfB) halfA to halfB else halfB to halfA
        return (first + second).chunked(5).joinToString(" ")
    }

    private fun half(uid: String, publicKey: ByteArray): String {
        val digest = MessageDigest.getInstance("SHA-512")
        var hash = uid.toByteArray() + publicKey
        repeat(ITERATIONS) { hash = digest.digest(hash + publicKey) }
        return BigInteger(1, hash).mod(MOD).toString().padStart(30, '0')
    }
}
```

Each side is SHA-512 iterated 5200 times, reduced mod 10^30; the two halves are sorted so both
users see the same number, then formatted in groups of five. Surfaced by `SafetyNumberViewModel` /
`SafetyNumberSection` on the chat partner's profile (`ChatUserProfileScreenContent.kt:175`, see
[[Chat Profile and Shared Media - Code]]).

---

## Related

[[End-to-End Encryption]] - [[Chat and Messaging - Code]] - [[Notifications - Code]] -
[[Chat Profile and Shared Media - Code]] - [[Settings - Code]] - [[Dependency Injection (Hilt)]] - [[Data Layer]]
