# Tech Stack and Versions

Every library, its version, and why it is here. Versions from `gradle/libs.versions.toml` and
`app/build.gradle.kts` as of release 0.1.13.1 (2026-10). Not everything is in the version
catalog - Agora and core-splashscreen are declared with an inline version (noted below).

## App identity and versioning

- Application id `com.example.messenger`; the launcher name is **Switch**
  (`app_name`, `res/values/strings.xml:2`).
- Release 0.1.13.1: `versionName = "0.1.13.1"`, `versionCode = 11301`. From this release the
  code follows `major * 1_000_000 + minor * 10_000 + patch * 100 + hotfix`, so 0.1.13.1 is
  `0 + 10_000 + 1_300 + 1 = 11301` and stays above the previous 0.1.13 (`113`). Releases are
  annotated git tags on `main` (`0.1.13`, `0.1.13.1`).

## Build and language

| Thing | Version | Where |
|---|---|---|
| Android Gradle Plugin | 9.1.1 | `libs.versions.toml:2` |
| Kotlin | 2.3.10 | `libs.versions.toml:3` |
| KSP | 2.3.9 | `libs.versions.toml:38` |
| compileSdk / targetSdk / minSdk | 37 / 36 / 29 | `app/build.gradle.kts:16`, `:30`, `:29` |
| Java source/target | 21 | `app/build.gradle.kts:56` |

Plugins applied (`app/build.gradle.kts:3`): android-application, kotlin-compose,
kotlin-serialization, KSP, google-services, hilt-android, kotlin-kapt (via
`com.android.legacy-kapt`, needed for ObjectBox's annotation processor under AGP 9,
`libs.versions.toml:163`), objectbox. Release build has R8 minify + resource shrink on
(`app/build.gradle.kts:46`) and is signed with the debug key for now.

The Gradle configuration cache is **not** enabled: the ObjectBox 5.4.2 Gradle plugin does not
support it, and the 6.0.0 line that does was still in beta when this was checked. The ObjectBox
model file `app/objectbox-models/default.json` is pinned to LF line endings in `.gitattributes`
so Windows checkouts do not show it as modified, and `.kotlin/` (Kotlin daemon session files) is
git-ignored.

On a memory-constrained Windows machine the build is run with an in-process Kotlin compiler and a
capped heap, for example
`./gradlew -Pkotlin.compiler.execution.strategy=in-process -Dorg.gradle.jvmargs="-Xmx3072m -XX:MaxMetaspaceSize=768m -Dfile.encoding=UTF-8" --max-workers=2 <tasks>`.

## UI - Jetpack Compose

Compose BOM `2026.05.01` (`libs.versions.toml:10`) manages Compose library versions. Material 3,
material-icons-extended, foundation, runtime-saveable, activity-compose, and the
lifecycle-runtime-compose / lifecycle-viewmodel-compose integrations. The Haze blur library
(`dev.chrisbanes.haze:haze`) was declared but never used and was removed in 0.1.13.1; the frosted
look comes from translucent token colors. `androidx.core:core-splashscreen` `1.0.1`
(`libs.versions.toml:121`, used at `app/build.gradle.kts:147`) provides the launch splash via
`installSplashScreen()` in `MainActivity`. See [[Presentation Layer (MVI)]] and
[[Appearance and Theming]].

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
- DataStore Preferences `1.1.7` (`libs.versions.toml:46`) - the appearance preferences (theme,
  wallpaper, intensity, bar position, text size, font). See [[Settings]] and
  [[Appearance and Theming]].
- Gson `2.14.0` (`libs.versions.toml:115`) - used by ObjectBox property converters for JSON
  columns.

## Backend - Firebase

Firebase BOM `34.10.0` (`libs.versions.toml:13`) manages: Auth (phone sign-in), Firestore
(messages/conversations/users), Realtime Database (presence, typing, call signaling), Storage
(media and voice clips), Cloud Messaging (push), Functions (client calls through
`FirebaseFunctions` in `europe-west1`). See [[Data Layer]], [[Authentication]],
[[Presence and Typing]], [[Notifications]].

Cloud Functions (`functions/`, TypeScript, Node 20, `firebase-functions` 6, `firebase-admin` 12,
`agora-token` 2):

- `onMessageCreated` - Firestore trigger that sends the message push with a preview ("Voice
  message" / "Video message" for clips, empty for encrypted text);
- `onCallSignalCreated` - Realtime Database trigger that sends the incoming-call push;
- `getRtcToken` - callable that issues an Agora RTC token;
- `deleteAccount` - callable that tombstones the profile, removes keys, aliases, avatars,
  presence and call nodes, and deletes the auth user (see [[Settings]]).

Security rules live in the repo (`firestore.rules`, `storage.rules`, `database.rules.json`) and
are deployed by hand with `firebase deploy --only firestore:rules,functions` (Storage rules after
comparing `storage.rules` with the console).

## Calls - Agora RTC

`io.agora.rtc:full-sdk:4.6.3`, declared directly in `app/build.gradle.kts:170` (not in the
catalog). Full SDK because video needs the native video modules (it is also most of the APK
size). App ID via `BuildConfig.AGORA_APP_ID` (`app/build.gradle.kts:18`, `:39`), resolved from
`local.properties`, then a Gradle project property (`-PAGORA_APP_ID` or
`ORG_GRADLE_PROJECT_AGORA_APP_ID`), then an empty string. No App ID is committed: until
2026-10-03 the last fallback was the real App ID hardcoded in the build file, so it is still in
older git history. A build without one succeeds and the app launches, but calls do not connect. `jniLibs { useLegacyPackaging = true }` and an
`androidx.concurrent` force in the resolution strategy support it. See
[[Calls (Voice and Video)]].

## Media

- Coil 3 `3.5.0` (`libs.versions.toml:41`) - image loading; `coil-video` for video-frame
  thumbnails, `coil-network-okhttp` for the fetcher. Configured in
  `MessengerApplication.onCreate`.
- Media3 (ExoPlayer + UI) `1.10.0` (`libs.versions.toml:42`) - video playback and voice-message
  playback (`ClipPlaybackController`).
- Platform `MediaRecorder` (AAC in MPEG-4, 64 kbps mono) - voice recording; no extra library. See
  [[Voice Messages]].
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
(`app/build.gradle.kts:83`), AssertK `0.28.1`, Turbine `1.2.0` (Flow testing), MockK `1.13.13`,
`kotlinx-coroutines-test`, `androidx.test:core`, with hand-written fakes in
`app/src/test/.../testutil/fakes/`. At 0.1.13.1 that is 45 test classes, 236 tests. Robolectric
`4.14.1` is still declared but no test uses it.

Instrumented tests (`app/src/androidTest/`, JUnit 4 + Compose UI test) are the WCAG contrast
suite described in [[Appearance and Theming]]. Run them on a personal device with
`adb install -r` plus `adb shell am instrument -w -r
com.example.messenger.test/androidx.test.runner.AndroidJUnitRunner`, not with
`connectedDebugAndroidTest`, which uninstalls the app and wipes its data and E2EE keys.

## Notable build-config details

- `resolutionStrategy` (`app/build.gradle.kts:92`) forces `androidx.concurrent:*` to 1.2.0 and
  pins `kotlin-stdlib` to the compiler version, because the media libs transitively pull a newer
  stdlib whose metadata the 2.3.10 toolchain + Hilt processor cannot read.
- `buildConfig = true` and `AGORA_APP_ID` `buildConfigField` (`app/build.gradle.kts:39`).

## Related

[[Architecture Overview]] - [[Data Layer]] - [[Dependency Injection (Hilt)]] -
[[Async and Reactivity]] - [[Calls (Voice and Video)]] - [[End-to-End Encryption]] -
[[Voice Messages]] - [[Home]]
