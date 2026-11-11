# End-to-End Encryption - Basic Example

The inherited pattern behind [[End-to-End Encryption]], in its plain textbook form - not the real
`ConversationCrypto`/`E2eeMessageCodec`, but the standard cryptographic construction they build on.
Then read [[End-to-End Encryption - Code]] for the real source.

(Teaching comments only; the real source has none.)

---

## What is inherited

The scheme is the textbook **X25519 (ECDH) -> HKDF -> AES-256-GCM** pipeline, using Google Tink's
vetted primitives: both sides derive the same shared secret from their key pair, HKDF stretches it
into an AES key, and AES-GCM encrypts with an authentication tag. Nothing here is app-specific -
it is the same construction Signal-style apps inherit.

## The basic example (canonical Tink X25519 + HKDF + AES-GCM)

```kotlin
// 1) Key agreement: both parties compute the SAME 32-byte secret without ever sending it.
val shared: ByteArray = X25519.computeSharedSecret(myPrivateKey, peerPublicKey)

// 2) Key derivation: stretch the raw secret into a real AES key, salted + bound to context.
val aesKey: ByteArray = Hkdf.computeHkdf(
    "HMACSHA256", shared, /* salt */ salt, /* info */ "chat-v1".toByteArray(), /* size */ 32,
)

// 3) Authenticated encryption: random 12-byte nonce, 128-bit tag, extra data bound in.
fun encrypt(key: ByteArray, plaintext: ByteArray, aad: ByteArray): ByteArray {
    val nonce = SecureRandom().generateSeed(12)
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.ENCRYPT_MODE, SecretKeySpec(key, "AES"), GCMParameterSpec(128, nonce))
    cipher.updateAAD(aad)                       // ties the ciphertext to this message's identity
    return nonce + cipher.doFinal(plaintext)    // ship nonce alongside the ciphertext
}
```

## How the real code specialises it

- The derivation is `ConversationCrypto.deriveConversationKey` (`data/crypto/ConversationCrypto.kt:25`).
  It makes the key **direction-independent** by sorting the two uids/epochs before building the
  salt and info, so both clients agree with no handshake round trip.
- The AAD is `messageAad(conversationId, id, senderId, type)` (`data/crypto/CryptoEncoding.kt:3`),
  so a ciphertext cannot be replayed under a different message identity.
- Real E2EE adds the machinery around the primitives: a per-epoch `PeerKeyRegistry`
  (`PeerKeyRegistry.kt:26`) so rotation does not break history, a keyring of identity keys (one per
  epoch, `IdentityKeyStore.kt:12`) wrapped by the Android Keystore at rest, a Block Store and recovery-code
  backup with a gate that keeps the app on `KeyRestoreScreen` until the key is `Ready` (`E2eeBootstrap.kt:28`), a one-shot key refetch and then a graceful "cannot be decrypted on this device"
  placeholder instead of a crash, per-file media and voice-clip keys, and safety numbers - all in
  [[End-to-End Encryption]].

## Related

[[End-to-End Encryption]] - [[End-to-End Encryption - Code]] -
[[End-to-End Encryption - Explained Simply]] - [[Reading the Code - Simple Patterns]] -
[[Chat and Messaging]] - [[Home]]
