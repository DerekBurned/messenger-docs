# End-to-End Encryption - Code

Companion to [[End-to-End Encryption]]. The full crypto source, block by block. Every file here
is small enough to show complete; snippets are verbatim from the repo.

Files covered (`data/crypto/`): `ConversationCrypto.kt`, `MessageEnvelope.kt`,
`E2eeMessageCodec.kt`, `IdentityKeyStore.kt`, `KeyWrapper.kt`, `MediaCipher.kt`,
`PeerKeyRegistry.kt`, `E2eeBootstrap.kt`, `SafetyNumber.kt`.

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

## 2. MessageEnvelope - payload types and helpers

`MessageEnvelope.kt` in full:

```kotlin
@Serializable
data class MessagePayload(
    val text: String = "",
    val replyToText: String? = null,
    val media: List<MediaSecret> = emptyList(),
)

@Serializable
data class MediaSecret(
    val itemId: String,
    val key: String,
    val blurHash: String = "",
)

fun messageAad(conversationId: String, messageId: String, senderId: String, type: String): ByteArray =
    "$conversationId|$messageId|$senderId|$type".toByteArray()

fun b64(bytes: ByteArray): String = java.util.Base64.getEncoder().encodeToString(bytes)
fun unb64(value: String): ByteArray = java.util.Base64.getDecoder().decode(value)
```

`MessagePayload` is what actually gets encrypted (the text, the reply snippet, and per-item media
file keys as `MediaSecret`). `messageAad` binds the ciphertext to the message identity;
`b64`/`unb64` are the wire encoding for keys/nonces/ciphertext.

---

## 3. E2eeMessageCodec - encode/decode a Message

`E2eeMessageCodec.kt` in full:

```kotlin
@Singleton
class E2eeMessageCodec @Inject constructor(
    private val identityKeyStore: IdentityKeyStore,
    private val peerKeyRegistry: PeerKeyRegistry,
    private val authService: FirebaseAuthService,
    private val participantResolver: ParticipantResolver,
) {
    private val json = Json { ignoreUnknownKeys = true }

    suspend fun isEncryptable(conversationId: String): Boolean {
        val myUid = authService.getCurrentUserId() ?: return false
        val partnerId = participantResolver.partnerOf(conversationId, myUid) ?: return false
        return peerKeyRegistry.ensurePeerKey(partnerId) != null
    }

    suspend fun encode(message: Message): RemoteMessageDto {
        val plain = message.toRemoteDto()
        if (message is Message.Call) return plain
        val hasEncryptedMedia = message is Message.Media && message.items.any { !it.fileKey.isNullOrBlank() }
        val myUid = authService.getCurrentUserId() ?: return plainOrFail(hasEncryptedMedia, plain)
        val partnerId = participantResolver.partnerOf(message.conversationId, myUid)
            ?: return plainOrFail(hasEncryptedMedia, plain)
        val peer = peerKeyRegistry.ensurePeerKey(partnerId) ?: return plainOrFail(hasEncryptedMedia, plain)
        val identity = identityKeyStore.getOrCreate(myUid)
        val payload = when (message) {
            is Message.Text -> MessagePayload(text = message.text, replyToText = message.replyToText)
            is Message.Media -> MessagePayload(
                text = message.caption, replyToText = message.replyToText,
                media = message.items.mapNotNull { item ->
                    item.fileKey?.takeIf { it.isNotBlank() }?.let { key ->
                        MediaSecret(itemId = item.id, key = key, blurHash = item.blurHash)
                    }
                },
            )
            is Message.Call -> return plain
        }
        val key = ConversationCrypto.deriveConversationKey(
            identity.privateKey, peer.publicKey, myUid, partnerId, identity.epoch, peer.epoch)
        val aad = messageAad(plain.conversationId, plain.id, plain.senderId, plain.type)
        val blob = ConversationCrypto.encrypt(key, json.encodeToString(payload).toByteArray(), aad)
        return plain.copy(
            text = "", replyToText = null,
            mediaItems = plain.mediaItems.map { it.copy(blurHash = "") },
            enc = 1, ciphertext = b64(blob.ciphertext), nonce = b64(blob.nonce),
            senderEpoch = identity.epoch, recipientEpoch = peer.epoch, recipientId = partnerId)
    }

    private fun plainOrFail(hasEncryptedMedia: Boolean, plain: RemoteMessageDto): RemoteMessageDto {
        check(!hasEncryptedMedia) { "media already encrypted but no peer key resolvable; refusing to send stranded plaintext" }
        return plain
    }

    suspend fun decode(dto: RemoteMessageDto, hasPendingWrites: Boolean = false): Message {
        if (dto.enc != 1) return dto.toDomain(hasPendingWrites)
        val payload = runCatching { decryptPayload(dto) }.getOrNull()
            ?: return undecryptable(dto, hasPendingWrites)
        val message = dto.copy(text = payload.text, replyToText = payload.replyToText).toDomain(hasPendingWrites)
        if (message !is Message.Media || payload.media.isEmpty()) return message
        val secrets = payload.media.associateBy { it.itemId }
        return message.copy(items = message.items.map { item ->
            secrets[item.id]?.let { secret -> item.copy(fileKey = secret.key, blurHash = secret.blurHash) } ?: item
        })
    }

    private suspend fun decryptPayload(dto: RemoteMessageDto): MessagePayload {
        val myUid = authService.getCurrentUserId() ?: error("not signed in")
        val iAmSender = dto.senderId == myUid
        val partnerId = if (iAmSender) dto.recipientId else dto.senderId
        val myEpoch = if (iAmSender) dto.senderEpoch else dto.recipientEpoch
        val partnerEpoch = if (iAmSender) dto.recipientEpoch else dto.senderEpoch
        val identity = identityKeyStore.getOrCreate(myUid)
        check(identity.epoch == myEpoch) { "local epoch ${identity.epoch}, message needs $myEpoch" }
        val peer = peerKeyRegistry.cachedPeerKey(partnerId, partnerEpoch)
            ?: peerKeyRegistry.peerKeyForEpoch(partnerId, partnerEpoch)
            ?: error("no key for $partnerId epoch $partnerEpoch")
        val key = ConversationCrypto.deriveConversationKey(
            identity.privateKey, peer.publicKey, myUid, partnerId, identity.epoch, peer.epoch)
        val aad = messageAad(dto.conversationId, dto.id, dto.senderId, dto.type)
        val plaintext = ConversationCrypto.decrypt(key, unb64(dto.nonce), unb64(dto.ciphertext), aad)
        return json.decodeFromString(String(plaintext))
    }

    private fun undecryptable(dto: RemoteMessageDto, hasPendingWrites: Boolean): Message =
        dto.copy(text = UNDECRYPTABLE_TEXT, type = Message.TYPE_TEXT, mediaItems = emptyList(),
            replyToMessageId = null, replyToText = null, replyToSenderId = null).toDomain(hasPendingWrites)

    companion object { const val UNDECRYPTABLE_TEXT = "Encrypted message" }
}
```

`encode` builds the `MessagePayload`, derives the key, AES-GCM-encrypts, and returns a DTO with
plaintext cleared and `enc=1` + ciphertext/nonce/epochs set; it refuses to downgrade to plaintext
when media is already encrypted (`plainOrFail`). `decode` works out which epochs apply based on
sender/recipient, loads the matching peer key, decrypts, and re-hydrates media file keys; any
failure returns the `undecryptable` placeholder rather than throwing. Wired into `FirestoreService`
(see [[Chat and Messaging - Code]]) and the FCM handler (see [[Notifications - Code]]).

---

## 4. IdentityKeyStore

`IdentityKeyStore.kt` in full - the device's long-term X25519 identity, persisted wrapped:

```kotlin
class IdentityKey(val privateKey: ByteArray, val publicKey: ByteArray, val epoch: Int)

class IdentityKeyStore(private val baseDir: File, private val keyWrapper: KeyWrapper) {
    private val cache = ConcurrentHashMap<String, IdentityKey>()

    fun getOrCreate(uid: String): IdentityKey = cache.computeIfAbsent(uid) {
        if (keyFile(uid).exists()) load(uid) else generateAndPersist(uid, epoch = 1)
    }

    fun rotate(uid: String): IdentityKey {
        val next = generateAndPersist(uid, epoch = getOrCreate(uid).epoch + 1)
        cache[uid] = next
        return next
    }

    private fun load(uid: String): IdentityKey {
        val bytes = keyWrapper.unwrap(keyFile(uid).readBytes())
        return decode(bytes) ?: error("identity key file for $uid is present but unreadable")
    }

    private fun generateAndPersist(uid: String, epoch: Int): IdentityKey {
        val (privateKey, publicKey) = ConversationCrypto.generateKeyPair()
        val buffer = ByteBuffer.allocate(69)
        buffer.put(1); buffer.putInt(epoch); buffer.put(privateKey); buffer.put(publicKey)
        baseDir.mkdirs()
        keyFile(uid).writeBytes(keyWrapper.wrap(buffer.array()))
        return IdentityKey(privateKey, publicKey, epoch)
    }

    private fun decode(bytes: ByteArray): IdentityKey? {
        if (bytes.size != 69 || bytes[0].toInt() != 1) return null
        val buffer = ByteBuffer.wrap(bytes, 1, 68)
        val epoch = buffer.int
        val privateKey = ByteArray(32).also { buffer.get(it) }
        val publicKey = ByteArray(32).also { buffer.get(it) }
        return IdentityKey(privateKey, publicKey, epoch)
    }

    private fun keyFile(uid: String) = File(baseDir, "identity_v1_$uid.bin")
}
```

The key is serialized as `[version=1][epoch:int][priv:32][pub:32]` = 69 bytes, wrapped by
`KeyWrapper` before being written to `filesDir/e2ee/identity_v1_{uid}.bin`. `rotate` bumps the
epoch - the rotation mechanism. Constructed in `di/CryptoModule.kt`.

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
        val part = File(dest.parentFile, dest.name + ".part")
        try {
            source.inputStream().use { input ->
                val nonce = ByteArray(NONCE_SIZE)
                check(input.read(nonce) == NONCE_SIZE) { "encrypted file too short" }
                val cipher = Cipher.getInstance("AES/GCM/NoPadding")
                cipher.init(Cipher.DECRYPT_MODE, SecretKeySpec(key, "AES"), GCMParameterSpec(TAG_BITS, nonce))
                part.outputStream().use { out -> CipherInputStream(input, cipher).use { it.copyTo(out) } }
            }
            dest.delete()
            check(part.renameTo(dest)) { "rename failed" }
        } catch (e: Exception) { part.delete(); dest.delete(); throw e }
    }
}
```

Per-file AES-256-GCM with the nonce prepended to the file, streamed via
`CipherInputStream`/`CipherOutputStream` so large media never loads fully into memory. `decrypt`
writes to a `.part` file and atomically renames on success, so a failed/aborted decrypt never
leaves a half-written file. The per-file key is generated in `MediaRepositoryImpl.uploadAndSend`
and carried inside the encrypted `MessagePayload` (see [[Chat and Messaging - Code]]).

---

## 7. PeerKeyRegistry

`PeerKeyRegistry.kt` in full - publish/fetch/cache peer public keys, per epoch:

```kotlin
class PeerKey(val uid: String, val publicKey: ByteArray, val epoch: Int, val verified: Boolean)

@Singleton
class PeerKeyRegistry @Inject constructor(
    private val firestore: FirebaseFirestore,
    private val peerKeyBox: Box<ObxPeerKey>,
    private val identityKeyStore: IdentityKeyStore,
) {
    suspend fun publishIfNeeded(myUid: String) {
        val identity = identityKeyStore.getOrCreate(myUid)
        val doc = keysCollection(myUid).document(identity.epoch.toString())
        val snapshot = runCatching { doc.get().await() }.getOrNull()
        val localPub = b64(identity.publicKey)
        if (snapshot?.getString("pub") == localPub) return
        doc.set(mapOf("pub" to localPub, "epoch" to identity.epoch, "algo" to "X25519",
            "updatedAt" to System.currentTimeMillis())).await()
    }

    suspend fun ensurePeerKey(uid: String): PeerKey? {
        val cached = latestLocal(uid)
        if (cached != null && System.currentTimeMillis() - cached.seenAt < REFRESH_MS) return cached.toPeerKey()
        val snapshot = runCatching {
            keysCollection(uid).orderBy("epoch", Query.Direction.DESCENDING).limit(1).get().await()
        }.getOrNull()
        val doc = snapshot?.documents?.firstOrNull() ?: return cached?.toPeerKey()
        val pub = doc.getString("pub") ?: return cached?.toPeerKey()
        val epoch = doc.getLong("epoch")?.toInt() ?: return cached?.toPeerKey()
        return store(uid, epoch, pub).toPeerKey()
    }

    suspend fun peerKeyForEpoch(uid: String, epoch: Int): PeerKey? {
        localRow(uid, epoch)?.let { return it.toPeerKey() }
        val doc = runCatching { keysCollection(uid).document(epoch.toString()).get().await() }.getOrNull() ?: return null
        val pub = doc.getString("pub") ?: return null
        return store(uid, epoch, pub).toPeerKey()
    }

    fun cachedPeerKey(uid: String, epoch: Int): PeerKey? = localRow(uid, epoch)?.toPeerKey()
    fun latestPeerKey(uid: String): PeerKey? = latestLocal(uid)?.toPeerKey()

    fun setVerified(uid: String, verified: Boolean) {
        val row = latestLocal(uid) ?: return
        row.verified = verified
        peerKeyBox.put(row)
    }

    private fun store(uid: String, epoch: Int, pub: String): ObxPeerKey {
        val existing = localRow(uid, epoch)
        val row = ObxPeerKey(boxId = existing?.boxId ?: 0, uidEpoch = "$uid:$epoch", uid = uid,
            epoch = epoch, publicKey = pub, verified = existing?.verified ?: false,
            seenAt = System.currentTimeMillis())
        peerKeyBox.put(row)
        return row
    }

    private fun localRow(uid: String, epoch: Int): ObxPeerKey? =
        peerKeyBox.query(ObxPeerKey_.uidEpoch.equal("$uid:$epoch")).build().use { it.findFirst() }

    private fun latestLocal(uid: String): ObxPeerKey? =
        peerKeyBox.query(ObxPeerKey_.uid.equal(uid)).orderDesc(ObxPeerKey_.epoch).build().use { it.findFirst() }

    private fun keysCollection(uid: String) = firestore.collection("users").document(uid).collection("keys")
    private fun ObxPeerKey.toPeerKey() = PeerKey(uid, unb64(publicKey), epoch, verified)

    companion object { private const val REFRESH_MS = 30 * 60 * 1000L }
}
```

Keys are published under Firestore `users/{uid}/keys/{epoch}` and cached as `ObxPeerKey` rows
(unique on `"uid:epoch"`). `ensurePeerKey` serves a fresh cached key (< 30 min) or fetches the
latest epoch; `peerKeyForEpoch` fetches a specific historical epoch so old messages stay
decryptable across rotations.

---

## 8. E2eeBootstrap and SafetyNumber

`E2eeBootstrap.kt` in full - publishes this device's key on startup:

```kotlin
@Singleton
class E2eeBootstrap @Inject constructor(
    private val authService: FirebaseAuthService,
    private val peerKeyRegistry: PeerKeyRegistry,
) {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    fun publishKeys() {
        val uid = authService.getCurrentUserId() ?: return
        scope.launch { runCatching { peerKeyRegistry.publishIfNeeded(uid) } }
    }
}
```

Called from `MessengerApplication.onCreate` and every successful sign-in (see
[[Authentication - Code]]); it owns an app-scoped IO scope.

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
`SafetyNumberSection` on the security screen ([[Settings - Code]]).

---

## Related

[[End-to-End Encryption]] - [[Chat and Messaging - Code]] - [[Notifications - Code]] -
[[Settings - Code]] - [[Dependency Injection (Hilt)]] - [[Data Layer]]
