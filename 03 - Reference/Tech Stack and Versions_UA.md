---
tags:
  - _UA
---
# Технологічний стек і версії

Кожна бібліотека, її версія й навіщо вона тут. Версії з `gradle/libs.versions.toml` та
`app/build.gradle.kts` на момент написання. Не все є в каталозі версій - дві залежності оголошені
напряму в `build.gradle.kts` (позначено нижче).

## Збірка та мова

| Що | Версія | Де |
|---|---|---|
| Android Gradle Plugin | 9.1.1 | `libs.versions.toml:2` |
| Kotlin | 2.3.10 | `libs.versions.toml:3` |
| KSP | 2.3.9 | `libs.versions.toml:38` |
| compileSdk / targetSdk / minSdk | 37 / 36 / 29 | `app/build.gradle.kts:16`, `:34`, `:33` |
| Java source/target | 21 | `app/build.gradle.kts:60` |

Застосовані плагіни (`app/build.gradle.kts:3`): android-application, kotlin-compose,
kotlin-serialization, KSP, google-services, hilt-android, kotlin-kapt (через
`com.android.legacy-kapt`, потрібен для процесора анотацій ObjectBox під AGP 9), objectbox.
Release-збірка має R8-minify + shrink ресурсів увімкненими (`app/build.gradle.kts:48`).

## UI - Jetpack Compose

Compose BOM `2026.05.01` (`libs.versions.toml:10`) керує версіями бібліотек Compose. Material 3,
material-icons-extended, foundation, runtime-saveable, activity-compose та інтеграції
lifecycle-runtime-compose / lifecycle-viewmodel-compose. Ефекти розмиття/скла використовують
`dev.chrisbanes.haze:haze:1.7.2`, оголошений напряму в `app/build.gradle.kts:124` (не в каталозі).
Див. [[Presentation Layer (MVI)_UA|Шар представлення (MVI)]].

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
- DataStore Preferences `1.1.7` (`libs.versions.toml:46`) - налаштування теми. Див.
  [[Settings_UA|Налаштування]].
- Gson `2.14.0` (`libs.versions.toml:115`) - використовується property-конвертерами ObjectBox для
  JSON-стовпців.

## Бекенд - Firebase

Firebase BOM `34.10.0` (`libs.versions.toml:13`) керує: Auth (вхід за телефоном), Firestore
(повідомлення/розмови/користувачі), Realtime Database (присутність, набір, сигналізація дзвінків),
Storage (медіа), Cloud Messaging (push), Functions (серверні тригери під `functions/`). Див.
[[Data Layer_UA|Шар даних]], [[Authentication_UA|Автентифікація]],
[[Presence and Typing_UA|Присутність і набір тексту]], [[Notifications_UA|Сповіщення]].

## Дзвінки - Agora RTC

`io.agora.rtc:full-sdk:4.6.3`, оголошений напряму в `app/build.gradle.kts:174` (не в каталозі).
Повний SDK, бо відео потребує нативних відеомодулів; App ID через `BuildConfig.AGORA_APP_ID`.
`jniLibs { useLegacyPackaging = true }` і force `androidx.concurrent` у стратегії розв'язання
підтримують його. Див. [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]].

## Медіа

- Coil 3 `3.5.0` (`libs.versions.toml:41`) - завантаження зображень; `coil-video` для мініатюр
  кадрів відео, `coil-network-okhttp` для fetcher. Налаштовується в `MessengerApplication.onCreate`.
- Media3 (ExoPlayer + UI) `1.10.0` (`libs.versions.toml:42`) - відтворення відео.
- CameraX `1.4.2` (`libs.versions.toml:43`) - живе прев'ю камери й захоплення.

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
(`app/build.gradle.kts:85`), AssertK `0.28.1`, Turbine `1.2.0` (тестування Flow), MockK `1.13.13`,
`kotlinx-coroutines-test`, Robolectric `4.14.1`, `androidx.test:core`. JUnit 4 + Espresso лишаються
для інструментованих тестів. Тестові класи живуть у `app/src/test/` та `app/src/androidTest/`.

## Помітні деталі build-конфігурації

- `resolutionStrategy` (`app/build.gradle.kts:94`) форсить `androidx.concurrent:*` до 1.2.0 і пінить
  `kotlin-stdlib` до версії компілятора, бо медіа-бібліотеки транзитивно тягнуть новіший stdlib,
  метадані якого набір інструментів 2.3.10 + процесор Hilt не можуть прочитати.
- `buildConfig = true` і `buildConfigField` `AGORA_APP_ID` (`app/build.gradle.kts:43`).

## Пов'язане

[[Architecture Overview_UA|Огляд архітектури]] - [[Data Layer_UA|Шар даних]] -
[[Dependency Injection (Hilt)_UA|Впровадження залежностей (Hilt)]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]] -
[[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]] -
[[End-to-End Encryption_UA|Наскрізне шифрування]] - [[Home_UA|Головна]]
