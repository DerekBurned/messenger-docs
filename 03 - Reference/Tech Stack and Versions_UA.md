---
tags:
  - _UA
---
# Технологічний стек і версії

Кожна бібліотека, її версія й навіщо вона тут. Версії з `gradle/libs.versions.toml` та
`app/build.gradle.kts` станом на реліз 0.2.2.4 (2026-11-11), повторно перевірено за вихідним кодом
цього релізу. Не все має запис у `[versions]` каталогу - Agora оголошена з версією напряму в
`app/build.gradle.kts` (`:204`), а core-splashscreen, Gson і Gradle-плагіни google-services та
Crashlytics мають версію прямо в каталозі (позначено нижче). Від 0.1.13.1 версії основних
бібліотек не змінилися; нове - Remote Config, Crashlytics, Block Store, libphonenumber, Kover, тип
збірки `e2e` і CI-воркфлоу (усе нижче).

## Ідентичність застосунку й версії

- Application id `com.example.messenger`; назва в лаунчері - **Switch**
  (`app_name`, `res/values/strings.xml:2`).
- Реліз 0.2.2.4: `VERSION_NAME=0.2.2.4`, `VERSION_CODE=20204`, зберігаються в `gradle.properties` і
  читаються в `app/build.gradle.kts:45` та `:46`. Код рахується як
  `major * 1_000_000 + minor * 10_000 + patch * 100 + hotfix`, тож 0.2.2.4 - це
  `0 + 20_000 + 200 + 4 = 20204`. Релізи - анотовані git-теги на `main` (існують `0.1.13`,
  `0.1.13.1`, `0.2.0.0`, `0.2.2.0` - `0.2.2.3`; злиття релізу 0.2.2.4 на `main` - це PR #87, і на
  перевіреному коміті тегу для нього не було). Як піднімають версії та випускають релізи - у
  [[Build, CI and Releases_UA|Збірка, CI і релізи]].
- Ліцензія: застосунок і пакет Cloud Functions - `AGPL-3.0-only`. Повний текст - `LICENSE` у корені
  репозиторію; вбудований шрифт Inter (SIL OFL) та адаптований код BlurHash (MIT) зберігають свої
  тексти в `licenses/` (`Inter-OFL.txt`, `BlurHash-MIT.txt`). Зміна ліцензії потрапила в PR #27
  (злито 2026-10-21): README репозиторію отримав розділ License із додатковим дозволом за розділом 7
  AGPL на лінкування з Agora RTC SDK, Google Play services і Firebase SDK, `functions/package.json`
  оголошує `AGPL-3.0-only`, а футер Налаштувань показує повідомлення про авторське право й AGPL
  (`presentation/screens/settings/SettingsScreenContent.kt:286`, посилання - `:295`).

## Збірка та мова

| Що | Версія | Де |
|---|---|---|
| Android Gradle Plugin | 9.1.1 | `libs.versions.toml:2` |
| Kotlin | 2.3.10 | `libs.versions.toml:3` |
| KSP | 2.3.9 | `libs.versions.toml:39` |
| Gradle wrapper | 9.3.1 | `gradle/wrapper/gradle-wrapper.properties` (`distributionUrl`) |
| compileSdk / targetSdk / minSdk | 37 / 36 / 29 | `app/build.gradle.kts:18`, `:44`, `:43` |
| Java source/target | 21 | `app/build.gradle.kts:83`, `:84` |
| Kover (покриття) | 0.9.11 | `libs.versions.toml:17` |

Застосовані плагіни (`app/build.gradle.kts:3` - `:14`): android-application, kotlin-compose,
kotlin-serialization, KSP, google-services (`4.4.4`), firebase-crashlytics (`3.0.8`; обидві версії
плагінів прямо в `libs.versions.toml:171`, `:172`), hilt-android, kotlin-kapt (через
`com.android.legacy-kapt`, потрібен для процесора анотацій ObjectBox під AGP 9,
`libs.versions.toml:176`), objectbox, kover. Release-збірка має R8-minify + shrink ресурсів
увімкненими (`app/build.gradle.kts:67`, `:68`) і поки що підписується debug-ключем (`:69`).

Існують три типи збірки: `debug`, `release` та `e2e` (`app/build.gradle.kts:75`), що починається з
`debug`, встановлюється як `com.example.messenger.e2e` і є `testBuildType` (`:81`), щоб
інструментовані тести ніколи не чіпали справжню інсталяцію. `debug` та `e2e` вимикають збір
Crashlytics через manifest-плейсхолдер, `release` вмикає (`:61`, `:65`). Release-збірка також
бере список функцій, що стартують вимкненими - див. [[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]].

Кеш конфігурації Gradle **не** увімкнено (у `gradle.properties` немає відповідного налаштування);
ранішня редакція цієї нотатки пояснювала це тим, що Gradle-плагін ObjectBox 5.4.2 його не
підтримує, - це неможливо повторно перевірити за вихідним кодом. Файл моделі ObjectBox
`app/objectbox-models/default.json` закріплено на кінцях рядків LF у `.gitattributes`, щоб на Windows
він не показувався зміненим, а `.kotlin/` (файли сесій Kotlin-демона) ігнорується git.

На машині з малою пам'яттю збірка запускається з in-process компілятором Kotlin і обмеженою купою,
наприклад
`./gradlew -Pkotlin.compiler.execution.strategy=in-process -Dorg.gradle.jvmargs="-Xmx3072m -XX:MaxMetaspaceSize=768m -Dfile.encoding=UTF-8" --max-workers=2 <tasks>`
(ці прапорці `app/src/test/RUNNING_TESTS.md` використовує в кожній команді; див. [[Build, CI and Releases_UA|Збірка, CI і релізи]]).

## UI - Jetpack Compose

Compose BOM `2026.05.01` (`libs.versions.toml:10`) керує версіями бібліотек Compose. Material 3,
material-icons-extended, foundation, runtime-saveable, activity-compose та інтеграції
lifecycle-runtime-compose / lifecycle-viewmodel-compose. Бібліотеку розмиття Haze
(`dev.chrisbanes.haze:haze`) було оголошено, але ніде не використано; її прибрано в 0.1.13.1 -
"матовий" вигляд дають напівпрозорі кольори токенів. `androidx.core:core-splashscreen` `1.0.1`
(`libs.versions.toml:132`, використано в `app/build.gradle.kts:179`) дає заставку запуску через
`installSplashScreen()` у `MainActivity`. Див. [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]]
та [[Appearance and Theming_UA|Вигляд і теми]].

## Навігація - Navigation 3

`androidx.navigation3` `1.1.3` (`libs.versions.toml:24`) з `lifecycle-viewmodel-navigation3` і
`kotlinx-serialization-json` `1.8.1` для типізованих маршрутів. Див. [[Navigation_UA|Навігація]].

## DI - Hilt

Hilt `2.59.2` (`libs.versions.toml:20`) через KSP, плюс `hilt-navigation-compose` `1.3.0` і
`hilt-work` + `androidx.hilt:hilt-compiler` `1.3.0` для `@HiltWorker`. Див.
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]].

## Асинхронність - Coroutines

kotlinx-coroutines `1.11.0` (`libs.versions.toml:29`): core, android і play-services (останній надає
`Task.await()`, що використовується в сервісах Firebase). Див.
[[Async and Reactivity_UA|Асинхронність і реактивність]].

## Локальне сховище

- ObjectBox `5.4.2` (`libs.versions.toml:16`) - локальна база (замінила Room). Обробка анотацій
  через kapt. Див. [[Data Layer_UA|Шар даних]].
- DataStore Preferences `1.1.7` (`libs.versions.toml:47`) - налаштування вигляду (тема, шпалери,
  інтенсивність, позиція бару, розмір тексту, шрифт). Див. [[Settings_UA|Налаштування]] та
  [[Appearance and Theming_UA|Вигляд і теми]].
- Gson `2.14.0` (`libs.versions.toml:124`, версія прямо в записі) - використовується property-конвертерами ObjectBox для
  JSON-стовпців.

## Бекенд - Firebase

Firebase BOM `34.10.0` (`libs.versions.toml:13`) керує: Auth (вхід за телефоном), Firestore
(повідомлення/розмови/користувачі), Realtime Database (присутність, набір, сигналізація дзвінків),
Storage (медіа й голосові кліпи), Cloud Messaging (push), Functions (клієнтські виклики через
`FirebaseFunctions` у `europe-west1`), Remote Config (`firebase-config`, перемикачі функцій) і
Crashlytics (`firebase-crashlytics`, звіти про збої) - останні два додано після 0.1.13.1, вони
пояснені в [[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]]. Див. [[Data Layer_UA|Шар даних]],
[[Authentication_UA|Автентифікація]], [[Presence and Typing_UA|Присутність і набір тексту]],
[[Notifications_UA|Сповіщення]].

Cloud Functions (`functions/`, TypeScript, Node 20, `firebase-functions` ^6.0.1, `firebase-admin`
^12.6.0, `agora-token` ^2.0.5 за `functions/package.json`; усі розгорнуті в `europe-west1`,
`functions/src/index.ts:28`):

- `onMessageCreated` - Firestore-тригер, що надсилає пуш повідомлення з прев'ю ("Voice message" /
  "Video message" для кліпів, порожнє для зашифрованого тексту);
- `onCallSignalCreated` - тригер Realtime Database, що надсилає пуш вхідного дзвінка;
- `getRtcToken` - callable, що видає RTC-токен Agora, і лише учаснику: він читає
  `callChannels/<channel>` у Realtime Database і відмовляє (`permission-denied`), якщо викликач не є
  `callerId` або `calleeId` цього дзвінка (`functions/src/index.ts:259`, перевірка учасника в
  `functions/src/callAccess.ts:9`; додано в PR #64, див. [[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]]);
- `deleteAccount` - callable, що ставить надгробок на профіль, прибирає ключі, дані відновлення,
  пристрої, псевдоніми, аватари, вузли присутності й дзвінків і видаляє користувача auth (див.
  [[Settings_UA|Налаштування]] та `functions/src/index.ts:305`).

Правила безпеки лежать у репозиторії (`firestore.rules`, `storage.rules`, `database.rules.json`) і
деплояться вручну через `firebase deploy --only firestore:rules,functions` (правила Storage - після
порівняння `storage.rules` з консоллю).

## Дзвінки - Agora RTC

`io.agora.rtc:full-sdk:4.6.3`, оголошений напряму в `app/build.gradle.kts:204` (не в каталозі).
Повний SDK, бо відео потребує нативних відеомодулів (він же - більша частина розміру APK). App ID
через `BuildConfig.AGORA_APP_ID` (`app/build.gradle.kts:20` - `:27`, `:53`), що береться з
`local.properties`, потім із Gradle project property (`-PAGORA_APP_ID` або
`ORG_GRADLE_PROJECT_AGORA_APP_ID`), потім порожній рядок. App ID у репозиторії не зберігається: до
2026-10-03 останнім фолбеком був справжній App ID, захардкоджений у build-файлі, тож у старій історії
git він досі є. Збірка без нього проходить і застосунок запускається, але дзвінки не з'єднуються.
`jniLibs { useLegacyPackaging = true }` (`:102`) і force `androidx.concurrent` у стратегії
розв'язання (`:123`) підтримують його. Див. [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].

## Медіа

- Coil 3 `3.5.0` (`libs.versions.toml:42`) - завантаження зображень; `coil-video` для мініатюр
  кадрів відео, `coil-network-okhttp` для fetcher. Налаштовується в `MessengerApplication.onCreate`.
- Media3 (ExoPlayer + UI) `1.10.0` (`libs.versions.toml:43`) - відтворення відео й голосових
  повідомлень (`ClipPlaybackController`), а з 12.10.2026 також круглих відеонотаток, які
  виводяться в `TextureView` через `attachVideo` / `detachVideo`. Див.
  [[Video Messages_UA|Відеоповідомлення]].
- Платформний `MediaRecorder` (AAC у MPEG-4, 64 кбіт/с моно) - запис голосу; без додаткових
  бібліотек. Див. [[Voice Messages_UA|Голосові повідомлення]].
- CameraX `1.4.2` (`libs.versions.toml:44`) - живе прев'ю камери й захоплення
  (`LifecycleCameraController`), а також круглі відеонотатки (`VideoNoteRecorder`:
  `ProcessCameraProvider` + `Preview` + `VideoCapture<Recorder>`, прив'язані до
  `ProcessLifecycleOwner`, SD з 1,2 Мбіт/с). Для відеонотаток використано API нижчого рівня, бо
  контролер не дає задати бітрейт кодера, а цього вимагає ліміт Storage у 40 MB.

## Crypto - E2EE

Google Tink `tink-android` `1.19.0` (`libs.versions.toml:50`) для X25519 + HKDF, плюс платформний
`javax.crypto` AES-GCM і Android Keystore для обгортання приватного ключа. Google Play services
Block Store (`play-services-auth-blockstore` `16.4.0`, `libs.versions.toml:51`) робить резервну
копію зв'язки ключів ідентичності, щоб її можна було відновити після перевстановлення
(`data/crypto/BlockStoreIdentityBackup.kt`). Див.
[[End-to-End Encryption_UA|Наскрізне шифрування]].

`libphonenumber` `9.0.38` (`libs.versions.toml:52`) перевіряє номери телефонів у формі входу
(`presentation/components/auth/PhoneNumberRules.kt`).

## Фонова робота

WorkManager `2.11.2` (`libs.versions.toml:32`) з `hilt-work` `1.3.0` - періодичний `SyncWorker`.
Lifecycle `2.10.0` (`libs.versions.toml:36`) з `lifecycle-process` для хука присутності
foreground/background на основі `ProcessLifecycleOwner`. Див. [[Data Layer_UA|Шар даних]] та
[[Presence and Typing_UA|Присутність і набір тексту]].

## Тестування

JUnit 5 (Jupiter) `5.11.4` (`libs.versions.toml:55`) на `useJUnitPlatform()`
(`app/build.gradle.kts:110`), AssertK `0.28.1`, Turbine `1.2.0` (тестування Flow), MockK `1.13.13`,
`kotlinx-coroutines-test`, `androidx.test:core`, з рукописними фейками в
`app/src/test/.../testutil/fakes/`. Тести на JUnit 4 (екранні тести Robolectric) працюють через
рушій JUnit Vintage (`junit-vintage-engine`, `app/build.gradle.kts:233`). Robolectric `4.14.1`
(`libs.versions.toml:59`) тепер використовується: на ньому працюють екранні тести `*ContentTest` у
`app/src/test/` (наприклад `presentation/screens/auth/AuthScreenContentTest.kt`).

На 0.2.2.4 у `app/src/test/` є 130 файлів Kotlin (114 з назвою `*Test.kt`), а в
`app/src/androidTest/` - 27. `app/src/test/RUNNING_TESTS.md` досі наводить 490 JVM-тестів (427
юніт, 42 інтеграційних із базою, 21 UI Robolectric) - таблиця з хвилі покриття 2026-10-21, тепер
застаріла: підрахунок анотацій `@Test` через grep дає 854 у `app/src/test/` і 88 в
`app/src/androidTest/` (кількість анотацій, а не запуск тестів).
Покриття рядків міряє Kover і перевіряє порогом 28% (`app/build.gradle.kts:248` -
`:273`).

Інструментовані тести (`app/src/androidTest/`, JUnit 4 + Compose UI test + UiAutomator `2.3.0` та
`androidx.test:rules` `1.6.1`) містять набір контрасту WCAG, описаний у
[[Appearance and Theming_UA|Вигляд і теми]], тести жестів, тест Keystore та наскрізні сценарії з
[[E2E Tests]]. Вони встановлюються як окремий застосунок `com.example.messenger.e2e`, а не поверх
справжньої інсталяції; точні команди - у "+B+".

## Помітні деталі build-конфігурації

- `resolutionStrategy` (`app/build.gradle.kts:121`) форсить `androidx.concurrent:*` до 1.2.0 і пінить
  `kotlin-stdlib` до версії компілятора, бо медіа-бібліотеки транзитивно тягнуть новіший stdlib,
  метадані якого набір інструментів 2.3.10 + процесор Hilt не можуть прочитати.
- `buildConfig = true` (`app/build.gradle.kts:96`) і три `buildConfigField`: `AGORA_APP_ID` (`:53`),
  `FEATURES_OFF_IN_RELEASE` (`:54`) та окреме для кожного типу збірки `FEATURES_OFF_BY_DEFAULT`
  (`:62` порожнє для debug, `:66` список для release). Останнє задає початковий стан feature flag-ів,
  поки Remote Config не відповів - див. [[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]].
- Завдання юніт-тестів оголошує `remoteconfig.template.json` своїм входом (`:113`), тож зміна шаблону
  перезапускає тести, що звіряють його з enum прапорців.

## Пов'язане

[[Architecture Overview_UA|Огляд архітектури]] - [[Data Layer_UA|Шар даних]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]] -
[[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]] -
[[End-to-End Encryption_UA|Наскрізне шифрування]] - [[Voice Messages_UA|Голосові повідомлення]] -
[[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]] - [[Build, CI and Releases_UA|Збірка, CI і релізи]] -
[[E2E Tests]] - [[Home_UA|Головна]]
