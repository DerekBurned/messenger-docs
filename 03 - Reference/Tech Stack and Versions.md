# Tech Stack and Versions

Every library, its version, and why it is here. Versions from `gradle/libs.versions.toml` and
`app/build.gradle.kts` as of release 0.2.2.4 (2026-11-11), re-checked against the source at that
release. Not everything has a `[versions]` entry in the catalog - Agora is declared with an inline
version in `app/build.gradle.kts` (`:204`), and core-splashscreen, Gson and the google-services and
Crashlytics Gradle plugins carry an inline version inside the catalog (noted below). Since 0.1.13.1
the core library versions did not move; what is new is Remote Config, Crashlytics, Block Store,
libphonenumber, Kover, the `e2e` build type and the CI workflows (all below).

## App identity and versioning

- Application id `com.example.messenger`; the launcher name is **Switch**
  (`app_name`, `res/values/strings.xml:2`).
- Release 0.2.2.4: `VERSION_NAME=0.2.2.4`, `VERSION_CODE=20204`, kept in `gradle.properties` and
  read by `app/build.gradle.kts:45` and `:46`. The code follows
  `major * 1_000_000 + minor * 10_000 + patch * 100 + hotfix`, so 0.2.2.4 is
  `0 + 20_000 + 200 + 4 = 20204`. Releases are annotated git tags on `main` (`0.1.13`, `0.1.13.1`,
  `0.2.0.0`, `0.2.2.0` to `0.2.2.3` exist; the 0.2.2.4 release merge on `main` is PR #87 and had
  no tag at the commit this note was checked against). How versions are bumped and released is in
  [[Build, CI and Releases]].
- License: the app and the Cloud Functions package are `AGPL-3.0-only`. The full text is the
  repo-root `LICENSE`; the bundled Inter typeface (SIL OFL) and the adapted BlurHash code (MIT) keep
  their own texts in `licenses/` (`Inter-OFL.txt`, `BlurHash-MIT.txt`). The licensing change landed
  in PR #27 (merged 2026-10-21): the repo README adds a License section with an additional
  permission under AGPL section 7 for linking the Agora RTC SDK, Google Play services and the
  Firebase SDKs, `functions/package.json` declares `AGPL-3.0-only`, and the Settings footer shows
  the copyright and AGPL notice (`presentation/screens/settings/SettingsScreenContent.kt:286`,
  link target at `:295`).

## Build and language

| Thing | Version | Where |
|---|---|---|
| Android Gradle Plugin | 9.1.1 | `libs.versions.toml:2` |
| Kotlin | 2.3.10 | `libs.versions.toml:3` |
| KSP | 2.3.9 | `libs.versions.toml:39` |
| Gradle wrapper | 9.3.1 | `gradle/wrapper/gradle-wrapper.properties` (`distributionUrl`) |
| compileSdk / targetSdk / minSdk | 37 / 36 / 29 | `app/build.gradle.kts:18`, `:44`, `:43` |
| Java source/target | 21 | `app/build.gradle.kts:83`, `:84` |
| Kover (coverage) | 0.9.11 | `libs.versions.toml:17` |

Plugins applied (`app/build.gradle.kts:3` to `:14`): android-application, kotlin-compose,
kotlin-serialization, KSP, google-services (`4.4.4`), firebase-crashlytics (`3.0.8`; both plugin
versions are inline in `libs.versions.toml:171`, `:172`), hilt-android, kotlin-kapt (via
`com.android.legacy-kapt`, needed for ObjectBox's annotation processor under AGP 9,
`libs.versions.toml:176`), objectbox, kover. Release build has R8 minify + resource shrink on
(`app/build.gradle.kts:67`, `:68`) and is signed with the debug key for now (`:69`).

Three build types exist: `debug`, `release`, and `e2e` (`app/build.gradle.kts:75`), which starts
from `debug`, installs as `com.example.messenger.e2e`, and is the `testBuildType` (`:81`) so that
instrumented tests never touch the real install. `debug` and `e2e` turn Crashlytics collection off
through a manifest placeholder, `release` turns it on (`:61`, `:65`). The release build also takes
a list of flags that start switched off - see [[Feature Flags and Remote Config]].

The Gradle configuration cache is **not** enabled (`gradle.properties` has no setting for it); an
earlier revision of this note recorded the reason as the ObjectBox 5.4.2 Gradle plugin not
supporting it, which could not be re-verified from the source. The ObjectBox
model file `app/objectbox-models/default.json` is pinned to LF line endings in `.gitattributes`
so Windows checkouts do not show it as modified, and `.kotlin/` (Kotlin daemon session files) is
git-ignored.

On a memory-constrained machine the build is run with an in-process Kotlin compiler and a
capped heap, for example
`./gradlew -Pkotlin.compiler.execution.strategy=in-process -Dorg.gradle.jvmargs="-Xmx3072m -XX:MaxMetaspaceSize=768m -Dfile.encoding=UTF-8" --max-workers=2 <tasks>`
(the flags `app/src/test/RUNNING_TESTS.md` uses for every command; see [[Build, CI and Releases]]).

## UI - Jetpack Compose

Compose BOM `2026.05.01` (`libs.versions.toml:10`) manages Compose library versions. Material 3,
material-icons-extended, foundation, runtime-saveable, activity-compose, and the
lifecycle-runtime-compose / lifecycle-viewmodel-compose integrations. The Haze blur library
(`dev.chrisbanes.haze:haze`) was declared but never used and was removed in 0.1.13.1; the frosted
look comes from translucent token colors. `androidx.core:core-splashscreen` `1.0.1`
(`libs.versions.toml:132`, used at `app/build.gradle.kts:179`) provides the launch splash via
`installSplashScreen()` in `MainActivity`. See [[Presentation Layer (MVI)]] and
[[Appearance and Theming]].

## Navigation - Navigation 3

`androidx.navigation3` `1.1.3` (`libs.versions.toml:24`) with
`lifecycle-viewmodel-navigation3` and `kotlinx-serialization-json` `1.8.1` for typed routes. See
[[Navigation]].

## DI - Hilt

Hilt `2.59.2` (`libs.versions.toml:20`) via KSP, plus `hilt-navigation-compose` `1.3.0` and
`hilt-work` + `androidx.hilt:hilt-compiler` `1.3.0` for `@HiltWorker`. See
[[Dependency Injection (Hilt)]].

## Async - Coroutines

kotlinx-coroutines `1.11.0` (`libs.versions.toml:29`): core, android, and play-services (the
last provides `Task.await()` used across the Firebase services). See [[Async and Reactivity]].

## Local persistence

- ObjectBox `5.4.2` (`libs.versions.toml:16`) - the local database (replaced Room). Annotation
  processing via kapt. See [[Data Layer]].
- DataStore Preferences `1.1.7` (`libs.versions.toml:47`) - the appearance preferences (theme,
  wallpaper, intensity, bar position, text size, font). See [[Settings]] and
  [[Appearance and Theming]].
- Gson `2.14.0` (`libs.versions.toml:124`, version inline) - used by ObjectBox property converters for JSON
  columns.

## Backend - Firebase

Firebase BOM `34.10.0` (`libs.versions.toml:13`) manages: Auth (phone sign-in), Firestore
(messages/conversations/users), Realtime Database (presence, typing, call signaling), Storage
(media and voice clips), Cloud Messaging (push), Functions (client calls through
`FirebaseFunctions` in `europe-west1`), Remote Config (`firebase-config`, the feature kill-switches)
and Crashlytics (`firebase-crashlytics`, crash reports) - the last two were added after 0.1.13.1
and are explained in [[Feature Flags and Remote Config]]. See [[Data Layer]], [[Authentication]],
[[Presence and Typing]], [[Notifications]].

Cloud Functions (`functions/`, TypeScript, Node 20, `firebase-functions` ^6.0.1, `firebase-admin`
^12.6.0, `agora-token` ^2.0.5, per `functions/package.json`; all deployed to `europe-west1`,
`functions/src/index.ts:28`):

- `onMessageCreated` - Firestore trigger that sends the message push with a preview ("Voice
  message" / "Video message" for clips, empty for encrypted text);
- `onCallSignalCreated` - Realtime Database trigger that sends the incoming-call push;
- `getRtcToken` - callable that issues an Agora RTC token, and only to a participant: it reads
  `callChannels/<channel>` in Realtime Database and refuses (`permission-denied`) unless the caller
  is that call's `callerId` or `calleeId` (`functions/src/index.ts:259`, participant check in
  `functions/src/callAccess.ts:9`; added by PR #64, see [[Feature Flags and Remote Config]]);
- `deleteAccount` - callable that tombstones the profile, removes keys, recovery data, devices,
  aliases, avatars, presence and call nodes, and deletes the auth user (see [[Settings]]
  and `functions/src/index.ts:305`).

Security rules live in the repo (`firestore.rules`, `storage.rules`, `database.rules.json`) and
are deployed by hand with `firebase deploy --only firestore:rules,functions` (Storage rules after
comparing `storage.rules` with the console).

## Calls - Agora RTC

`io.agora.rtc:full-sdk:4.6.3`, declared directly in `app/build.gradle.kts:204` (not in the
catalog). Full SDK because video needs the native video modules (it is also most of the APK
size). App ID via `BuildConfig.AGORA_APP_ID` (`app/build.gradle.kts:20` to `:27`, `:53`), resolved from
`local.properties`, then a Gradle project property (`-PAGORA_APP_ID` or
`ORG_GRADLE_PROJECT_AGORA_APP_ID`), then an empty string. No App ID is committed: until
2026-10-03 the last fallback was the real App ID hardcoded in the build file, so it is still in
older git history. A build without one succeeds and the app launches, but calls do not connect.
`jniLibs { useLegacyPackaging = true }` (`:102`) and an `androidx.concurrent` force in the
resolution strategy (`:123`) support it. See
[[Calls (Voice and Video)]].

## Media

- Coil 3 `3.5.0` (`libs.versions.toml:42`) - image loading; `coil-video` for video-frame
  thumbnails, `coil-network-okhttp` for the fetcher. Configured in
  `MessengerApplication.onCreate`.
- Media3 (ExoPlayer + UI) `1.10.0` (`libs.versions.toml:43`) - video playback and voice-message
  playback (`ClipPlaybackController`), and since 2026-10-12 also round video notes, rendered into
  a `TextureView` attached through `attachVideo` / `detachVideo`. See [[Video Messages]].
- Platform `MediaRecorder` (AAC in MPEG-4, 64 kbps mono) - voice recording; no extra library. See
  [[Voice Messages]].
- CameraX `1.4.2` (`libs.versions.toml:44`) - live camera preview cell and capture
  (`LifecycleCameraController`), and round video notes (`VideoNoteRecorder`:
  `ProcessCameraProvider` + `Preview` + `VideoCapture<Recorder>` bound to `ProcessLifecycleOwner`,
  SD at 1.2 Mbps). The video-note path uses the lower-level API because the controller cannot set
  the encoder bitrate, which the 40 MB Storage limit requires.

## Crypto - E2EE

Google Tink `tink-android` `1.19.0` (`libs.versions.toml:50`) for X25519 + HKDF, plus platform
`javax.crypto` AES-GCM and Android Keystore for private-key wrapping. Google Play services
Block Store (`play-services-auth-blockstore` `16.4.0`, `libs.versions.toml:51`) backs up the
identity keyring so it can be restored after a reinstall (`data/crypto/BlockStoreIdentityBackup.kt`).
See [[End-to-End Encryption]].

`libphonenumber` `9.0.38` (`libs.versions.toml:52`) validates phone numbers on the sign-in form
(`presentation/components/auth/PhoneNumberRules.kt`).

## Background work

WorkManager `2.11.2` (`libs.versions.toml:32`) with `hilt-work` `1.3.0` - the periodic
`SyncWorker`. Lifecycle `2.10.0` (`libs.versions.toml:36`) incl. `lifecycle-process` for the
`ProcessLifecycleOwner`-based app-foreground/background presence hook. See [[Data Layer]] and
[[Presence and Typing]].

## Testing

JUnit 5 (Jupiter) `5.11.4` (`libs.versions.toml:55`) on `useJUnitPlatform()`
(`app/build.gradle.kts:110`), AssertK `0.28.1`, Turbine `1.2.0` (Flow testing), MockK `1.13.13`,
`kotlinx-coroutines-test`, `androidx.test:core`, with hand-written fakes in
`app/src/test/.../testutil/fakes/`. JUnit 4 tests (the Robolectric screen tests) run through the
JUnit Vintage engine (`junit-vintage-engine`, `app/build.gradle.kts:233`). Robolectric `4.14.1`
(`libs.versions.toml:59`) is now used: the `*ContentTest` screen tests under `app/src/test/`
(for example `presentation/screens/auth/AuthScreenContentTest.kt`) run on it.

At 0.2.2.4 there are 130 Kotlin files under `app/src/test/` (114 named `*Test.kt`) and 27 under
`app/src/androidTest/`. `app/src/test/RUNNING_TESTS.md` still lists 490 JVM tests (427 unit, 42
database integration, 21 Robolectric UI), a table that dates from the coverage push of
2026-10-21 and is now stale: counting `@Test` annotations with grep gives 854 under
`app/src/test/` and 88 under `app/src/androidTest/` (annotation counts, not a test run).
Line coverage is measured by Kover and gated at 28% (`app/build.gradle.kts:248` to
`:273`).

Instrumented tests (`app/src/androidTest/`, JUnit 4 + Compose UI test + UiAutomator `2.3.0` and
`androidx.test:rules` `1.6.1`) hold the WCAG contrast suite described in
[[Appearance and Theming]], gesture tests, a Keystore test and the end-to-end scenarios in
[[E2E Tests]]. They install as the separate `com.example.messenger.e2e` app, not on top of the
real install; the exact commands are in [[Build, CI and Releases]].

## Notable build-config details

- `resolutionStrategy` (`app/build.gradle.kts:121`) forces `androidx.concurrent:*` to 1.2.0 and
  pins `kotlin-stdlib` to the compiler version, because the media libs transitively pull a newer
  stdlib whose metadata the 2.3.10 toolchain + Hilt processor cannot read.
- `buildConfig = true` (`app/build.gradle.kts:96`) and three `buildConfigField`s: `AGORA_APP_ID`
  (`:53`), `FEATURES_OFF_IN_RELEASE` (`:54`) and a per-build-type `FEATURES_OFF_BY_DEFAULT`
  (`:62` empty for debug, `:66` the release list). The last one seeds the feature flags before
  Remote Config answers - see [[Feature Flags and Remote Config]].
- The unit-test task declares `remoteconfig.template.json` as an input (`:113`), so editing the
  template re-runs the tests that check it against the flag enum.

## Related

[[Architecture Overview]] - [[Data Layer]] - [[Dependency Injection (Hilt)]] -
[[Async and Reactivity]] - [[Calls (Voice and Video)]] - [[End-to-End Encryption]] -
[[Voice Messages]] - [[Feature Flags and Remote Config]] - [[Build, CI and Releases]] -
[[E2E Tests]] - [[Home]]
