---
tags:
  - _UA
---
# Технологічний стек і версії

Кожна бібліотека, її версія й навіщо вона тут. Версії з `gradle/libs.versions.toml` та
`app/build.gradle.kts` станом на реліз 0.1.13.1 (2026-10). Не все є в каталозі версій - Agora і
core-splashscreen оголошені з версією напряму (позначено нижче).

## Ідентичність застосунку й версії

- Application id `com.example.messenger`; назва в лаунчері - **Switch**
  (`app_name`, `res/values/strings.xml:2`).
- Реліз 0.1.13.1: `versionName = "0.1.13.1"`, `versionCode = 11301`. Від цього релізу код
  рахується як `major * 1_000_000 + minor * 10_000 + patch * 100 + hotfix`, тож 0.1.13.1 - це
  `0 + 10_000 + 1_300 + 1 = 11301`, і це більше за попередній 0.1.13 (`113`). Релізи - анотовані
  git-теги на `main` (`0.1.13`, `0.1.13.1`).

## Збірка та мова

| Що | Версія | Де |
|---|---|---|
| Android Gradle Plugin | 9.1.1 | `libs.versions.toml:2` |
| Kotlin | 2.3.10 | `libs.versions.toml:3` |
| KSP | 2.3.9 | `libs.versions.toml:38` |
| compileSdk / targetSdk / minSdk | 37 / 36 / 29 | `app/build.gradle.kts:16`, `:30`, `:29` |
| Java source/target | 21 | `app/build.gradle.kts:56` |

Застосовані плагіни (`app/build.gradle.kts:3`): android-application, kotlin-compose,
kotlin-serialization, KSP, google-services, hilt-android, kotlin-kapt (через
`com.android.legacy-kapt`, потрібен для процесора анотацій ObjectBox під AGP 9,
`libs.versions.toml:163`), objectbox. Release-збірка має R8-minify + shrink ресурсів увімкненими
(`app/build.gradle.kts:46`) і поки що підписується debug-ключем.

Кеш конфігурації Gradle **не** увімкнено: Gradle-плагін ObjectBox 5.4.2 його не підтримує, а лінія
6.0.0, що підтримує, на момент перевірки ще була в beta. Файл моделі ObjectBox
`app/objectbox-models/default.json` закріплено на кінцях рядків LF у `.gitattributes`, щоб на Windows
він не показувався зміненим, а `.kotlin/` (файли сесій Kotlin-демона) ігнорується git.

На Windows-машині з малою пам'яттю збірка запускається з in-process компілятором Kotlin і обмеженою
купою, наприклад
`./gradlew -Pkotlin.compiler.execution.strategy=in-process -Dorg.gradle.jvmargs="-Xmx3072m -XX:MaxMetaspaceSize=768m -Dfile.encoding=UTF-8" --max-workers=2 <tasks>`.

## UI - Jetpack Compose

Compose BOM `2026.05.01` (`libs.versions.toml:10`) керує версіями бібліотек Compose. Material 3,
material-icons-extended, foundation, runtime-saveable, activity-compose та інтеграції
lifecycle-runtime-compose / lifecycle-viewmodel-compose. Бібліотеку розмиття Haze
(`dev.chrisbanes.haze:haze`) було оголошено, але ніде не використано; її прибрано в 0.1.13.1 -
"матовий" вигляд дають напівпрозорі кольори токенів. `androidx.core:core-splashscreen` `1.0.1`
(`libs.versions.toml:121`, використано в `app/build.gradle.kts:147`) дає заставку запуску через
`installSplashScreen()` у `MainActivity`. Див. [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]]
та [[Appearance and Theming_UA|Вигляд і теми]].

## Навігація - Navigation 3

`androidx.navigation3` `1.1.3` (`libs.versions.toml:23`) з `lifecycle-viewmodel-navigation3` і
`kotlinx-serialization-json` `1.8.1` для типізованих маршрутів. Див. [[Navigation_UA|Навігація]].

## DI - Hilt

Hilt `2.59.2` (`libs.versions.toml:19`) через KSP, плюс `hilt-navigation-compose` `1.3.0` і
`hilt-work` + `androidx.hilt:hilt-compiler` `1.3.0` для `@HiltWorker`. Див.
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]].

## Асинхронність - Coroutines

kotlinx-coroutines `1.11.0` (`libs.versions.toml:28`): core, android і play-services (останній надає
`Task.await()`, що використовується в сервісах Firebase). Див.
[[Async and Reactivity_UA|Асинхронність і реактивність]].

## Локальне сховище

- ObjectBox `5.4.2` (`libs.versions.toml:16`) - локальна база (замінила Room). Обробка анотацій
  через kapt. Див. [[Data Layer_UA|Шар даних]].
- DataStore Preferences `1.1.7` (`libs.versions.toml:46`) - налаштування вигляду (тема, шпалери,
  інтенсивність, позиція бару, розмір тексту, шрифт). Див. [[Settings_UA|Налаштування]] та
  [[Appearance and Theming_UA|Вигляд і теми]].
- Gson `2.14.0` (`libs.versions.toml:115`) - використовується property-конвертерами ObjectBox для
  JSON-стовпців.

## Бекенд - Firebase

Firebase BOM `34.10.0` (`libs.versions.toml:13`) керує: Auth (вхід за телефоном), Firestore
(повідомлення/розмови/користувачі), Realtime Database (присутність, набір, сигналізація дзвінків),
Storage (медіа й голосові кліпи), Cloud Messaging (push), Functions (клієнтські виклики через
`FirebaseFunctions` у `europe-west1`). Див. [[Data Layer_UA|Шар даних]],
[[Authentication_UA|Автентифікація]], [[Presence and Typing_UA|Присутність і набір тексту]],
[[Notifications_UA|Сповіщення]].

Cloud Functions (`functions/`, TypeScript, Node 20, `firebase-functions` 6, `firebase-admin` 12,
`agora-token` 2):

- `onMessageCreated` - Firestore-тригер, що надсилає пуш повідомлення з прев'ю ("Voice message" /
  "Video message" для кліпів, порожнє для зашифрованого тексту);
- `onCallSignalCreated` - тригер Realtime Database, що надсилає пуш вхідного дзвінка;
- `getRtcToken` - callable, що видає RTC-токен Agora;
- `deleteAccount` - callable, що ставить надгробок на профіль, прибирає ключі, псевдоніми, аватари,
  вузли присутності й дзвінків і видаляє користувача auth (див. [[Settings_UA|Налаштування]]).

Правила безпеки лежать у репозиторії (`firestore.rules`, `storage.rules`, `database.rules.json`) і
деплояться вручну через `firebase deploy --only firestore:rules,functions` (правила Storage - після
порівняння `storage.rules` з консоллю).

## Дзвінки - Agora RTC

`io.agora.rtc:full-sdk:4.6.3`, оголошений напряму в `app/build.gradle.kts:170` (не в каталозі).
Повний SDK, бо відео потребує нативних відеомодулів (він же - більша частина розміру APK). App ID
через `BuildConfig.AGORA_APP_ID` (`app/build.gradle.kts:18`, `:39`), що береться з
`local.properties`, потім із Gradle project property (`-PAGORA_APP_ID` або
`ORG_GRADLE_PROJECT_AGORA_APP_ID`), потім порожній рядок. App ID у репозиторії не зберігається: до
2026-10-03 останнім фолбеком був справжній App ID, захардкоджений у build-файлі, тож у старій історії
git він досі є. Збірка без нього проходить і застосунок запускається, але дзвінки не з'єднуються.
`jniLibs { useLegacyPackaging = true }` і force `androidx.concurrent` у стратегії розв'язання
підтримують його. Див. [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].

## Медіа

- Coil 3 `3.5.0` (`libs.versions.toml:41`) - завантаження зображень; `coil-video` для мініатюр
  кадрів відео, `coil-network-okhttp` для fetcher. Налаштовується в `MessengerApplication.onCreate`.
- Media3 (ExoPlayer + UI) `1.10.0` (`libs.versions.toml:42`) - відтворення відео й голосових
  повідомлень (`ClipPlaybackController`), а з 12.10.2026 також круглих відеонотаток, які
  виводяться в `TextureView` через `attachVideo` / `detachVideo`. Див.
  [[Video Messages_UA|Відеоповідомлення]].
- Платформний `MediaRecorder` (AAC у MPEG-4, 64 кбіт/с моно) - запис голосу; без додаткових
  бібліотек. Див. [[Voice Messages_UA|Голосові повідомлення]].
- CameraX `1.4.2` (`libs.versions.toml:43`) - живе прев'ю камери й захоплення
  (`LifecycleCameraController`), а також круглі відеонотатки (`VideoNoteRecorder`:
  `ProcessCameraProvider` + `Preview` + `VideoCapture<Recorder>`, прив'язані до
  `ProcessLifecycleOwner`, SD з 1,2 Мбіт/с). Для відеонотаток використано API нижчого рівня, бо
  контролер не дає задати бітрейт кодера, а цього вимагає ліміт Storage у 40 MB.

## Crypto - E2EE

Google Tink `tink-android` `1.19.0` (`libs.versions.toml:49`) для X25519 + HKDF, плюс платформний
`javax.crypto` AES-GCM і Android Keystore для обгортання приватного ключа. Див.
[[End-to-End Encryption_UA|Наскрізне шифрування]].

## Фонова робота

WorkManager `2.11.2` (`libs.versions.toml:31`) з `hilt-work` `1.3.0` - періодичний `SyncWorker`.
Lifecycle `2.10.0` (`libs.versions.toml:35`) з `lifecycle-process` для хука присутності
foreground/background на основі `ProcessLifecycleOwner`. Див. [[Data Layer_UA|Шар даних]] та
[[Presence and Typing_UA|Присутність і набір тексту]].

## Тестування

JUnit 5 (Jupiter) `5.11.4` (`libs.versions.toml:52`) на `useJUnitPlatform()`
(`app/build.gradle.kts:83`), AssertK `0.28.1`, Turbine `1.2.0` (тестування Flow), MockK `1.13.13`,
`kotlinx-coroutines-test`, `androidx.test:core`, з рукописними фейками в
`app/src/test/.../testutil/fakes/`. На 0.1.13.1 це 45 тестових класів, 236 тестів. Robolectric
`4.14.1` досі оголошено, але жоден тест його не використовує.

Інструментовані тести (`app/src/androidTest/`, JUnit 4 + Compose UI test) - це набір тестів
контрасту WCAG, описаний у [[Appearance and Theming_UA|Вигляд і теми]]. На особистому пристрої їх
запускають через `adb install -r` плюс `adb shell am instrument -w -r
com.example.messenger.test/androidx.test.runner.AndroidJUnitRunner`, а не через
`connectedDebugAndroidTest`, що видаляє застосунок разом із даними й ключами E2EE.

## Помітні деталі build-конфігурації

- `resolutionStrategy` (`app/build.gradle.kts:92`) форсить `androidx.concurrent:*` до 1.2.0 і пінить
  `kotlin-stdlib` до версії компілятора, бо медіа-бібліотеки транзитивно тягнуть новіший stdlib,
  метадані якого набір інструментів 2.3.10 + процесор Hilt не можуть прочитати.
- `buildConfig = true` і `buildConfigField` `AGORA_APP_ID` (`app/build.gradle.kts:39`).

## Пов'язане

[[Architecture Overview_UA|Огляд архітектури]] - [[Data Layer_UA|Шар даних]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]] -
[[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]] -
[[End-to-End Encryption_UA|Наскрізне шифрування]] - [[Voice Messages_UA|Голосові повідомлення]] -
[[Home_UA|Головна]]
