---
tags:
  - _UA
---
# Наскрізне шифрування

Опціональне E2EE по розмовах для чатів 1:1, побудоване на узгодженні ключів X25519, деривації
ключів HKDF та AES-256-GCM. Маркер версії в коді - `msgr-e2ee-v1`
(`data/crypto/ConversationCrypto.kt:18`). Криптографічні примітиви - з Google Tink
(`tink-android` `1.19.0`, `gradle/libs.versions.toml:50`) плюс платформний `javax.crypto`.

## Складові (`data/crypto/`)

- `ConversationCrypto` (`ConversationCrypto.kt:13`) - шар примітивів (ECDH, HKDF, AES-GCM).
- `E2eeMessageCodec` (`E2eeMessageCodec.kt:18`) - encode/decode `Message` у/з `RemoteMessageDto`.
- `IdentityKeyStore` (`IdentityKeyStore.kt:12`) - **зв'язка ключів** ідентичності цього пристрою: одна
  пара X25519 на епоху (`IdentityKeyring`, `IdentityKeyring.kt:8`), найновіша використовується для
  надсилання, обгорнута у спокої.
- `KeyWrapper` / `KeystoreKeyWrapper` - обгортання приватного ключа сховищем Android Keystore у
  спокої.
- `PeerKeyRegistry` (`PeerKeyRegistry.kt:26`) - публікація/отримання/кешування публічних ключів
  пірінгів.
- `E2eeBootstrap` (`E2eeBootstrap.kt:28`) - життєвий цикл ключів для акаунта: вирішує, чи пристрій
  `Ready`, чи потребує коду відновлення, чи відстає (`E2eeState`, `E2eeState.kt:3`).
- Резервне копіювання й відновлення: `BlockStoreIdentityBackup` (`BlockStoreIdentityBackup.kt:16`,
  Google Block Store), `RecoveryVault` / `RecoveryCipher` (`RecoveryVault.kt:57`, `:27`),
  `RecoveryCode` (`RecoveryCode.kt:6`), `RecoveryCodeStore` (`RecoveryCodeStore.kt:8`) та
  `UndecryptableCachePurge` (`UndecryptableCachePurge.kt:13`).
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
   відкритий текст (`plainOrFail`, `:87`). Голосовий кліп із ключем файлу тут теж рахується як
   зашифроване медіа.
3. Будує `MessagePayload` (текст + сніпет відповіді + `MediaSecret`-ключі файлів по елементах,
   а для голосового кліпу `ClipSecret(key, waveform)` з `Clip.toSecret()`, `:119`), деривує ключ
   розмови й шифрує JSON-навантаження.
4. Повертає DTO з очищеними полями відкритого тексту й `enc = 1`, `ciphertext`, `nonce`,
   `senderEpoch`, `recipientEpoch`, `recipientId`. BlurHash'і й хвиля кліпу теж прибираються з
   відкритого боку й несуться всередині зашифрованого навантаження. Див.
   [[Voice Messages_UA|Голосові повідомлення]].

## Декодування (`E2eeMessageCodec.decode`, `E2eeMessageCodec.kt:94`)

Якщо `enc != 1`, декодувати як відкритий текст. Інакше `decryptPayload` (`:129`) визначає, які епохи
застосовуються, залежно від того, відправник цей пристрій чи отримувач, завантажує **ключ цього
пристрою для епохи, названої в повідомленні** зі зв'язки (`identityKeyStore.keyFor(myUid, myEpoch)`, тож
усе, надіслане на ранній ключ, що ще є у зв'язці, лишається читабельним, зокрема й власні надіслані
повідомлення цього пристрою; надсилання завжди використовує найновіший ключ), завантажує відповідний
ключ пірінга (`cachedPeerKey`, потім `peerKeyForEpoch`), виводить ключ і розшифровує. Якщо щось
зривається (немає ключа, не та епоха, невідповідність тега), він спершу один раз повторно отримує
опублікований ключ пірінга для цієї епохи (`retryWithFreshPeerKey`, `:153`, з обмеженням на
пірінг-епоху раз на 5 хвилин), і якщо й це не вдається, повертає повідомлення-заглушку `undecryptable`
(`:179`), а не кидає виняток. Текст заглушки (`placeholderFor`, `:168`) каже, що сталося:

- `UNDECRYPTABLE_TEXT` - "Message cannot be decrypted on this device" (`:191`), загальний випадок;
- `LOCKED_TEXT` - "Sent before your encryption key was reset" (`:192`), коли у зв'язці немає ключа
  для цієї епохи;
- `BEHIND_TEXT` - "Sent to a newer encryption key that this device doesn't have yet" (`:193`),
  коли повідомлення називає епоху новішу за будь-яку, яку має цей пристрій.

Усі три перелічені в `PLACEHOLDERS` (`:195`), які використовує `UndecryptableCachePurge.purge`
(`UndecryptableCachePurge.kt:18`): коли ключ надходить (відновлення, start fresh, встановлення з Block
Store), він видаляє кешовані рядки кожної розмови, що містила заглушку, окрім ненадісланих, тож їх
отримують і розшифровують знову.

## Реєстр ключів пірінгів та епохи

`PeerKeyRegistry` (`PeerKeyRegistry.kt:26`) зберігає публічні ключі кожного користувача у Firestore
`users/{uid}/keys/{epoch}` і кешує їх локально як рядки `ObxPeerKey` (унікальні за `"uid:epoch"`,
`data/local/obx/ObxPeerKey.kt:12`).
- `publishIfNeeded` (`:31`) тепер - транзакція **create-if-absent**, що ніколи не ротує: повертає
  `Published`, коли документа епохи не було, `AlreadyPublished`, коли там той самий ключ, і `Conflict`,
  коли там інший (відмова за правилами також рахується конфліктом, `:60`-`:62`). `precheckPublish`
  (`:167`) відмовляється додавати нову епоху до акаунта, що вже має ключі, якщо виклик не передав
  `allowNewEpoch` (використовується лише "start fresh"). Раніші версії ротували локальну ідентичність
  на епоху вище за найвищу серверну при конфлікті; цієї поведінки більше немає, бо другий вхід тоді
  перезаписував ключ першого пристрою й мовчки відрізав його.
- `ensurePeerKey` (`:101`) повертає кешований ключ, коли він свіжий (< 30 хв, `REFRESH_MS`, `:164`),
  інакше отримує останню епоху з Firestore й зберігає її.
- `peerKeyForEpoch` (`:115`) отримує конкретну історичну епоху, щоб старі повідомлення лишались
  розшифровуваними після ротації; `refreshPeerKey` (`:94`) примусово повторно отримує для повтору
  декодування.
- `publishedKeysOnServer` (`:82`) читає опубліковані ключі акаунта прямо із сервера, а
  `observeNewestKey` (`:65`) слухає найновіший - обидва живлять машину станів bootstrap нижче.
- `setVerified` (`:127`) позначає ключ пірінга верифікованим (використовується верифікацією
  safety number); змінений ключ під тією самою епохою скидає прапорець верифікації (`store`,
  `:133`-`:137`).

Епохи - це механізм ротації: новий ключ ідентичності означає нову епоху; ключі зберігаються на
епоху, тож перешифрування не ламає історію.

## Життєвий цикл ключів: `E2eeBootstrap` і `E2eeState`

`E2eeBootstrap` (`E2eeBootstrap.kt:28`) - це `@Singleton` з власним IO-scope і м'ютексом. Він віддає
`status: StateFlow<E2eeStatus>`, чий стан - це `Resolving`, `Ready`, `NeedsLink`, `Behind` (обидва з
`hasVault`) або `Offline` (`E2eeState.kt:3`-`:9`). `MessengerApplication` викликає
`e2eeBootstrap.refresh()` під час запуску (`MessengerApplication.kt:118`), а репозиторій автентифікації -
після входу (`data/repository/AuthRepositoryImpl.kt:69`, `:99`). `resolveLocked` (`E2eeBootstrap.kt:181`)
вирішує:

1. Локальна зв'язка є -> `Ready` і фонова `sync` (опублікувати за потреби, резервна копія Block Store,
   статус відновлення). Пристрій, запам'ятований як `Behind`, спершу перевіряє сервер.
2. Локальної зв'язки немає: спробувати резервну копію Block Store, приймається лише якщо ділить ключ
   з тим, що опублікував акаунт; встановити її й очистити заглушки.
3. Досі нічого, а акаунт уже має опубліковані ключі або сховище відновлення -> `NeedsLink`: пристрій
   чекає на код відновлення (або пов'язаний пристрій), а не створює ключ.
4. Лише акаунт без опублікованого ключа й без сховища отримує **новий** ідентифікатор, опублікований
   як епоха 1; якщо публікація програє гонку, ключ відкидається, а пристрій переходить у `NeedsLink`.
5. Будь-який збій стає `Offline` із повтором через 5 с (`:426`).

Пристрій `Ready` також стежить за найновішим опублікованим ключем акаунта (`watch`, `:269`). Якщо це
ключ, якого пристрій не має, він чекає пільгові 5 с (`BEHIND_GRACE_MS`), повторно читає сервер і
вирішує під м'ютексом (`settle`, `:326`): ключі, яким акаунт суперечить або які він ніколи не
публікував, відкладаються вбік (з копією старого файлу ключа), далі стан стає `NeedsLink` (ключів не
лишилось), `Behind` (лише старіші ключі) або лишається `Ready`. `Behind` зберігається
(`behind_v1_<uid>.bin`), тож перезапуск не пускає пристрій назад, поки він не отримає найновіший ключ.

`AppNavigation` показує `MainDisplay` лише для `E2eeState.Ready`; кожен інший стан відображає
`KeyRestoreScreen`, шлюз на кшталт екрана входу
(`presentation/navigation/graph/AppNavigation.kt:43`-`:60`,
`presentation/screens/recovery/KeyRestoreScreen.kt:45`).

## Відновлення: Block Store, код відновлення, start fresh

- Резервна копія Block Store: після успішної синхронізації вся зв'язка (найновіші 50 епох, `MAX_KEYS`,
  `BlockStoreIdentityBackup.kt:49`) зберігається в Google Block Store, тож перевстановлення на тому
  самому акаунті Google може відновити її без дій користувача.
- Код відновлення: `RecoveryCode.generate` (`RecoveryCode.kt:26`) створює випадковий секрет, що
  показується як base32 із контрольними символами. `RecoveryCipher.seal` (`RecoveryVault.kt:34`)
  обгортає всю зв'язку AES-GCM під ключем HKDF, виведеним із коду, uid і випадкової солі, а
  `RecoveryVault` зберігає блоб у `users/{uid}/recovery/identity` з `version` 2
  (`RecoveryVault.kt:115`-`:118`, `:142`); сховища версії 1 (один ключ) усе ще відкриваються як
  зв'язка з одного ключа. Збережений код локально тримається із відбитком зв'язки, яку він обгорнув
  (`RecoveryCodeStore.kt:32`, файл `recovery_v1_<uid>.bin`), а сховище перезгортається, коли
  зв'язка змінюється, - як транзакція, що пише лише поки код цього пристрою ще відкриває сховище
  (`rewrapAsync`, `RecoveryVault.kt:74`; `rewrapVault`, `E2eeBootstrap.kt:375`).
- Відновлення: `restoreWithCode` (`E2eeBootstrap.kt:81`) розбирає код, відкриває сховище, вимагає, щоб
  відновлений найновіший ключ збігався з опублікованим акаунтом (інакше `Outdated`), встановлює
  зв'язку, очищає заглушки й стає `Ready`. Результати - `RestoreResult` (`E2eeState.kt:21`).
- Start fresh: `startFresh` (`:110`) видаляє сховище й збережений код, додає епоху на одиницю вище за
  найновішу на сервері (`IdentityKeyStore.addEpoch`, `IdentityKeyStore.kt:39`), публікує її з
  `allowNewEpoch = true` і відкидає знову, якщо публікація не вдалась. Старіші повідомлення лишаються
  заблокованими, а інші пристрої перестають отримувати нові, поки не відновлять; екран спершу просить
  підтвердження (`KeyRestoreViewModel.requestStartFresh`, `KeyRestoreViewModel.kt:59`).

UI: `KeyRestoreScreen` (шлюз), `RecoveryCodeScreen` / `RecoveryCodeViewModel` (створення й перегляд
коду, доступні з налаштувань і з банера через `RecoveryCodeRoute`,
`presentation/navigation/graph/MainDisplay.kt:316`, `:374`) і діалог-підказка (`RecoveryPromptDialog`,
`MainDisplay.kt:685`).

## Ключі ідентичності у спокої

`IdentityKeyStore` (створюється в `filesDir/e2ee`, `di/CryptoModule.kt:42`) тримає зв'язку пристрою в
`identity_v1_<uid>.bin` (`IdentityKeyStore.kt:129`); байти обгортає `KeystoreKeyWrapper` через Android
Keystore, тож приватні ключі не зберігаються відкрито, а файл пишеться у проміжний `.tmp` і атомарно
переноситься на місце (`persist`, `:121`-`:127`), тож перерваний запис не може втратити всю зв'язку. Файл
ключа зі старого формату з одним ключем читається як зв'язка з одного ключа. Нечитабельний файл
відкладається в `identity_v1_<uid>.rejected.bin` (`:97`, `:131`), а не блокує відновлення. Встановлення
зв'язки зливає її з уже наявними ключами (`IdentityKeyring.merge`, `IdentityKeyring.kt:46`): для кожної
епохи перемагає ключ, який опублікував акаунт, зв'язка, що суперечить опублікованому локальному ключу,
відхиляється, а ключі новіші за будь-що опубліковане відкидаються.

Видалення акаунта прибирає зв'язку, збережений код відновлення й запис Block Store через
`ISessionCleaner.clearOnDelete` -> `E2eeBootstrap.forget` (`data/service/AndroidSessionCleaner.kt:63`,
`E2eeBootstrap.kt:174`); звичайний вихід зупиняє спостерігач і зберігає ключ (див.
[[Settings_UA|Налаштування]]).

## Шифрування медіа

Коли розмова піддається шифруванню, `MediaRepositoryImpl.uploadAndSend`
(`data/repository/MediaRepositoryImpl.kt:148`) генерує ключ на файл через
`MediaCipher.generateFileKey()`, шифрує файл перед завантаженням і кладе ключ у `MediaSecret`-
навантаження повідомлення (тож він сам E2E-захищений). Завантаження розшифровує тим ключем
(`downloadMedia`, `:72`). Голосові кліпи використовують ту саму схему з ключем на файл через
`ClipRepositoryImpl`, з ключем у `ClipSecret`. `MediaCipher.decrypt` пише в унікальний файл `.part`
і атомарно переміщує його на місце, тож невдале розшифрування ніколи не лишає зламаний файл під
кінцевим ім'ям. Див. [[Chat and Messaging_UA|Чат і обмін повідомленнями]] та
[[Voice Messages_UA|Голосові повідомлення]].

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
- Публікація ключів create-if-absent і шлюз застосунку на `Ready` означають, що другий пристрій більше не
  може мовчки замінити ключ першого: він має довести володіння ключем акаунта (код або Block Store) або
  явно почати заново.

## Застереження для звірки з мейнтейнером

- Це схема v1 (`msgr-e2ee-v1`) для розмов 1:1; групового E2EE у коді не знайдено.
- Trust-on-first-use: ключі пірінгів отримуються з Firestore й кешуються; верифікація ручна через
  safety numbers. Ротація більше не мовчазна: нові епохи створюються лише "start fresh" (або першим
  ключем акаунта), а інші пристрої переводяться в `Behind`/`NeedsLink`, а не перезаписуються. Як
  продукт планує розвивати зв'язування кількох пристроїв, описано в
  [[Multi-Device Encryption (Research)]], а не в цій нотатці.
- E2EE не за feature flag: жоден запис `FeatureFlag` не читається в `data/crypto/` чи
  `presentation/screens/recovery/` (див. [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]]).

## Пов'язане

[[End-to-End Encryption - Explained Simply_UA|Наскрізне шифрування - Простими словами]] (версія для початківців з аналогіями, покроковим прикладом і схемами Mermaid) -
[[Chat and Messaging_UA|Чат і обмін повідомленнями]] - [[Data Layer_UA|Шар даних]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] - [[Settings_UA|Налаштування]] -
[[Notifications_UA|Сповіщення]] - [[Async and Reactivity_UA|Асинхронність і реактивність]] - [[Multi-Device Encryption (Research)]] - [[Feature Flags and Remote Config_UA|Feature flags і Remote Config]]
