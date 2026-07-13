# Tech Stack and Versions

Every library, its version, and why it is here. Versions from `gradle/libs.versions.toml` and
`app/build.gradle.kts` at time of writing. Not everything is in the version catalog - two
dependencies are declared directly in `build.gradle.kts` (noted below).

## Build and language

| Thing | Version | Where |
|---|---|---|
| Android Gradle Plugin | 9.1.1 | `libs.versions.toml:2` |
| Kotlin | 2.3.10 | `libs.versions.toml:3` |
| KSP | 2.3.9 | `libs.versions.toml:38` |
| compileSdk / targetSdk / minSdk | 37 / 36 / 29 | `app/build.gradle.kts:16`, `:34`, `:33` |
| Java source/target | 21 | `app/build.gradle.kts:60` |

Plugins applied (`app/build.gradle.kts:3`): android-application, kotlin-compose,
kotlin-serialization, KSP, google-services, hilt-android, kotlin-kapt (via
`com.android.legacy-kapt`, needed for ObjectBox's annotation processor under AGP 9), objectbox.
Release build has R8 minify + resource shrink on (`app/build.gradle.kts:48`).

## UI - Jetpack Compose

Compose BOM `2026.05.01` (`libs.versions.toml:10`) manages Compose library versions. Material 3,
material-icons-extended, foundation, runtime-saveable, activity-compose, and the
lifecycle-runtime-compose / lifecycle-viewmodel-compose integrations. Blur/glass effects use
`dev.chrisbanes.haze:haze:1.7.2`, declared directly in `app/build.gradle.kts:124` (not in the
catalog). See [[Presentation Layer (MVI)]].

## Navigation - Navigation 3

`androidx.navigation3` `1.1.3` (`libs.versions.toml:23`) with
`lifecycle-viewmodel-navigation3` and `kotlinx-serialization-json` `1.8.1` for typed routes. See
[[Navigation]].

## DI - Hilt

Hilt `2.59.2` (`libs.versions.toml:19`) via KSP, plus `hilt-navigation-compose` `1.3.0` and
`hilt-work` + `androidx.hilt:hilt-compiler` `1.3.0` for `@HiltWorker`. See
[[Dependency Injection (Hilt)]].

## Async - Coroutines

kotlinx-coroutines `1.11.0` (`libs.versions.toml:28`): core, android, and play-services (the
last provides `Task.await()` used across the Firebase services). See [[Async and Reactivity]].

## Local persistence

- ObjectBox `5.4.2` (`libs.versions.toml:16`) - the local database (replaced Room). Annotation
  processing via kapt. See [[Data Layer]].
- DataStore Preferences `1.1.7` (`libs.versions.toml:46`) - the theme preference. See
  [[Settings]].
- Gson `2.14.0` (`libs.versions.toml:115`) - used by ObjectBox property converters for JSON
  columns.

## Backend - Firebase

Firebase BOM `34.10.0` (`libs.versions.toml:13`) manages: Auth (phone sign-in), Firestore
(messages/conversations/users), Realtime Database (presence, typing, call signaling), Storage
(media), Cloud Messaging (push), Functions (server-side triggers under `functions/`). See
[[Data Layer]], [[Authentication]], [[Presence and Typing]], [[Notifications]].

## Calls - Agora RTC

`io.agora.rtc:full-sdk:4.6.3`, declared directly in `app/build.gradle.kts:174` (not in the
catalog). Full SDK because video needs the native video modules; App ID via
`BuildConfig.AGORA_APP_ID`. `jniLibs { useLegacyPackaging = true }` and an
`androidx.concurrent` force in the resolution strategy support it. See
[[Calls (Voice and Video)]].

## Media

- Coil 3 `3.5.0` (`libs.versions.toml:41`) - image loading; `coil-video` for video-frame
  thumbnails, `coil-network-okhttp` for the fetcher. Configured in
  `MessengerApplication.onCreate`.
- Media3 (ExoPlayer + UI) `1.10.0` (`libs.versions.toml:42`) - video playback.
- CameraX `1.4.2` (`libs.versions.toml:43`) - live camera preview cell and capture.

## Crypto - E2EE

Google Tink `tink-android` `1.19.0` (`libs.versions.toml:49`) for X25519 + HKDF, plus platform
`javax.crypto` AES-GCM and Android Keystore for private-key wrapping. See
[[End-to-End Encryption]].

## Background work

WorkManager `2.11.2` (`libs.versions.toml:31`) with `hilt-work` `1.3.0` - the periodic
`SyncWorker`. Lifecycle `2.10.0` (`libs.versions.toml:35`) incl. `lifecycle-process` for the
`ProcessLifecycleOwner`-based app-foreground/background presence hook. See [[Data Layer]] and
[[Presence and Typing]].

## Testing

JUnit 5 (Jupiter) `5.11.4` (`libs.versions.toml:52`) on `useJUnitPlatform()`
(`app/build.gradle.kts:85`), AssertK `0.28.1`, Turbine `1.2.0` (Flow testing), MockK `1.13.13`,
`kotlinx-coroutines-test`, Robolectric `4.14.1`, `androidx.test:core`. JUnit 4 + Espresso remain
for instrumented tests. Test classes live in `app/src/test/` and `app/src/androidTest/`.

## Notable build-config details

- `resolutionStrategy` (`app/build.gradle.kts:94`) forces `androidx.concurrent:*` to 1.2.0 and
  pins `kotlin-stdlib` to the compiler version, because the media libs transitively pull a newer
  stdlib whose metadata the 2.3.10 toolchain + Hilt processor cannot read.
- `buildConfig = true` and `AGORA_APP_ID` `buildConfigField` (`app/build.gradle.kts:43`).

## Related

[[Architecture Overview]] - [[Data Layer]] - [[Dependency Injection (Hilt)]] -
[[Async and Reactivity]] - [[Calls (Voice and Video)]] - [[End-to-End Encryption]] - [[Home]]
