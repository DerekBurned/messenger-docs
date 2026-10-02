---
tags:
  - _UA
---
# Наскрізне шифрування

Опціональне E2EE по розмовах для чатів 1:1, побудоване на узгодженні ключів X25519, деривації
ключів HKDF та AES-256-GCM. Маркер версії в коді - `msgr-e2ee-v1`
(`data/crypto/ConversationCrypto.kt:18`). Криптографічні примітиви - з Google Tink
(`tink-android` `1.19.0`, `gradle/libs.versions.toml:49`) плюс платформний `javax.crypto`.

## Складові (`data/crypto/`)

- `ConversationCrypto` (`ConversationCrypto.kt:13`) - шар примітивів (ECDH, HKDF, AES-GCM).
- `E2eeMessageCodec` (`E2eeMessageCodec.kt:18`) - encode/decode `Message` у/з `RemoteMessageDto`.
- `IdentityKeyStore` - довготривала пара ключів ідентичності X25519 цього пристрою, епоха й
  обгортання приватного ключа.
- `KeyWrapper` / `KeystoreKeyWrapper` - обгортання приватного ключа сховищем Android Keystore у
  спокої.
- `PeerKeyRegistry` (`PeerKeyRegistry.kt:14`) - публікація/отримання/кешування публічних ключів
  пірінгів.
- `E2eeBootstrap` (`E2eeBootstrap.kt:12`) - публікує ключ цього пристрою під час запуску.
- `ParticipantResolver` / `ObxParticipantResolver` - резолвить uid пірінга для розмови.
- `MediaCipher` - шифрування медіа на файл.
- `SafetyNumber` (`SafetyNumber.kt:6`) - порівнюваний відбиток двох ключів ідентичності.
- `MessagePayload` / `MediaSecret` / `ClipSecret` (`data/crypto/model/`) - форма серіалізованого
  відкритого тексту.

Обв'язка DI - у `di/CryptoModule.kt` (див. [[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]]).

## Узгодження й деривація ключів

`ConversationCrypto.deriveConversationKey` (`ConversationCrypto.kt:25`):

```kotlin
val shared = X25519.computeSharedSecret(myPrivateKey, peerPublicKey)
val (loUid, hiUid) = if (myUid < peerUid) myUid to peerUid else peerUid to myUid
val (loEpoch, hiEpoch) = if (myUid < peerUid) myEpoch to peerEpoch else peerEpoch to myEpoch
val salt = MessageDigest.getInstance("SHA-256").digest("$loUid|$hiUid".toByteArray())
return Hkdf.computeHkdf("HMACSHA256", shared, salt, "$INFO_PREFIX|$loEpoch|$hiEpoch".toByteArray(), KEY_SIZE)
```

- X25519 ECDH продукує спільний секрет із мого приватного ключа й публічного ключа пірінга; обидві
  сторони обчислюють той самий секрет.
- Uid'и й епохи сортуються, тож обидва учасники деривують ідентичні salt і info-рядок незалежно від
  того, хто "я" - ключ симетричний до пари, а не до напряму.
- HKDF-SHA256 розтягує сирий ECDH-секрет у 32-байтовий AES-ключ, засолений ідентичністю пари й
  прив'язаний до епох пар ключів (тож ротація ключа будь-якої сторони змінює деривований ключ). Це
  свіжа деривація на кожне надсилання/отримання з кешованого ключового матеріалу, а не збережена
  сесія.

## Шифрування повідомлення (AES-256-GCM з AAD)

`encrypt`/`decrypt` (`ConversationCrypto.kt:46`, `:54`) використовують `AES/GCM/NoPadding` з
випадковим 12-байтовим nonce і 128-бітним тегом, і прив'язують додаткові автентифіковані дані:

```kotlin
val aad = messageAad(plain.conversationId, plain.id, plain.senderId, plain.type)
val blob = ConversationCrypto.encrypt(key, json.encodeToString(payload).toByteArray(), aad)
```

AAD прив'язує шифротекст до id розмови, id повідомлення, відправника й типу
(`messageAad`, `CryptoEncoding.kt:3`; викликається в `E2eeMessageCodec.kt:71`), тож шифротекст не можна відтворити під іншою ідентичністю повідомлення,
не провалюючи перевірку GCM-тегу.

## Кодування повідомлення (`E2eeMessageCodec.encode`, `E2eeMessageCodec.kt:38`)

Мета: продукувати `RemoteMessageDto`, що несе шифротекст замість відкритого тексту, коли розмова
піддається шифруванню, і граційно відкочуватися до відкритого тексту, коли ні.

1. Повідомлення-дзвінки ніколи не шифруються (вони не несуть тексту користувача) - повертаються як
   plain.
2. Резолвить пірінга (`ParticipantResolver.partnerOf`) і забезпечує публічний ключ пірінга
   (`peerKeyRegistry.ensurePeerKey`); якщо будь-що відсутнє, надсилає відкритий текст - хіба що
   повідомлення вже має зашифроване медіа, у такому разі відмовляється надіслати "покинутий"
   відкритий текст (`plainOrFail`, `:87`).
3. Будує `MessagePayload` (текст + сніпет відповіді + `MediaSecret`-ключі файлів по елементах),
   деривує ключ розмови й шифрує JSON-навантаження.
4. Повертає DTO з очищеними полями відкритого тексту й `enc = 1`, `ciphertext`, `nonce`,
   `senderEpoch`, `recipientEpoch`, `recipientId`. BlurHash'і теж прибираються з відкритого боку й
   несуться всередині зашифрованого навантаження.

## Декодування (`E2eeMessageCodec.decode`, `E2eeMessageCodec.kt:94`)

Якщо `enc != 1`, декодує як відкритий текст. Інакше `decryptPayload` (`:129`) визначає, які епохи
застосовуються, залежно від того, чи цей пристрій - відправник, чи отримувач, завантажує відповідний
ключ пірінга (`cachedPeerKey`, потім `peerKeyForEpoch`), деривує ключ і розшифровує. Якщо щось
провалюється (відсутній ключ, неправильна епоха, розбіжність тегу), він повертає плейсхолдер
`undecryptable` (`:169`; "Message cannot be decrypted on this device", `UNDECRYPTABLE_TEXT`,
`:181`), а не кидає виняток - UI показує
плейсхолдер замість краху.

## Реєстр ключів пірінгів та епохи

`PeerKeyRegistry` (`PeerKeyRegistry.kt:14`) зберігає публічні ключі кожного користувача під Firestore
`users/{uid}/keys/{epoch}` і кешує їх локально як рядки `ObxPeerKey` (унікальні за `"uid:epoch"`,
`data/local/obx/ObxPeerKey.kt:10`).
- `publishIfNeeded` (`PeerKeyRegistry.kt:20`) вивантажує публічний ключ/епоху цього пристрою, якщо віддалена копія
  відрізняється.
- `ensurePeerKey` (`:64`) повертає кешований ключ, коли свіжий (< 30 хв, `REFRESH_MS`), інакше
  отримує останню епоху з Firestore і зберігає її.
- `peerKeyForEpoch` (`:78`) отримує конкретну історичну епоху, тож старі повідомлення лишаються
  розшифровними після ротації.
- `setVerified` (`:90`) позначає ключ пірінга верифікованим (використовується верифікацією
  safety-number).

Епохи - це механізм ротації: новий ключ ідентичності означає нову епоху; ключі зберігаються по
епохах, тож перекеювання не ламає історію. Публікацію під час запуску керує `E2eeBootstrap`
(`E2eeBootstrap.kt:19`), який на власному `@Singleton` IO-scope викликає `publishIfNeeded` з
`MessengerApplication.onCreate` (`MessengerApplication.kt:108`).

## Ключі ідентичності у спокої

`IdentityKeyStore` (конструюється в `filesDir/e2ee`, `di/CryptoModule.kt:33`) тримає довготривалу
пару ключів X25519 пристрою та епоху; приватний ключ обгорнутий `KeystoreKeyWrapper` через Android
Keystore, тож він не зберігається у відкритому вигляді. `getOrCreate(uid)` ліниво створює пару при
першому використанні.

## Шифрування медіа

Коли розмова піддається шифруванню, `MediaRepositoryImpl.uploadAndSend`
(`data/repository/MediaRepositoryImpl.kt:148`) генерує ключ на файл через
`MediaCipher.generateFileKey()`, шифрує файл перед завантаженням і кладе ключ у `MediaSecret`-
навантаження повідомлення (тож він сам E2E-захищений). Завантаження розшифровує тим ключем
(`downloadMedia`, `:72`). Див. [[Chat and Messaging_UA|Чат і обмін повідомленнями]].

## Safety numbers

`SafetyNumber.compute` (`SafetyNumber.kt:11`) деривує порівнюваний відбиток із ключів ідентичності
двох користувачів: кожна сторона хешується SHA-512 із 5200 ітераціями, редукується за модулем 10^30,
сортується, тож обидва користувачі бачать те саме число, і форматується в групи по п'ять, розділені
пробілами. Двоє користувачів, що порівнюють safety numbers поза каналом, можуть підтвердити
відсутність "людини посередині". Виводиться через `SafetyNumberViewModel` / `SafetyNumberSection` на
екрані безпеки (див. [[Settings_UA|Налаштування]]).

## Чому такий дизайн

- X25519 + HKDF + AES-GCM - це стандартна, добре зрозуміла конструкція E2EE; Tink надає перевірені
  реалізації примітивів.
- Сортування uid/епох перед деривацією ключів робить деривацію незалежною від напряму, тож обидва
  клієнти узгоджуються без раунду рукостискання.
- Зберігання ключів по епохах плюс граційний плейсхолдер "undecryptable" тримають історію читабельною
  крізь ротації й ніколи не крашать чат на відсутньому ключі.
- Відмова понижуватися до відкритого тексту, коли медіа вже зашифроване (`plainOrFail`), запобігає
  випадковому витоку контенту, який мав бути захищеним.

## Застереження для звірки з мейнтейнером

- Це схема v1 (`msgr-e2ee-v1`) для розмов 1:1; групове E2EE тут не реалізоване.
- Довіра при першому використанні (trust-on-first-use): ключі отримуються з Firestore і кешуються;
  верифікація ручна через safety numbers. Точну політику ротації/зростання епох поза тим, що показує
  код, не зафіксовано в джерелі - звірте, перш ніж документувати це як гарантію.

## Пов'язане

[[End-to-End Encryption - Explained Simply_UA|Наскрізне шифрування - Простими словами]] (версія для початківців з аналогіями, покроковим прикладом і схемами Mermaid) -
[[Chat and Messaging_UA|Чат і обмін повідомленнями]] - [[Data Layer_UA|Шар даних]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] - [[Settings_UA|Налаштування]] -
[[Notifications_UA|Сповіщення]] - [[Async and Reactivity_UA|Асинхронність і реактивність]]
