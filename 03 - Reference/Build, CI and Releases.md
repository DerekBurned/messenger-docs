# Build, CI and Releases

How the app is built, how its version number is set, what the two GitHub Actions workflows do, how
a release is cut, and how to run every kind of test the repository has. Everything here was read
from the repository at release 0.2.2.4 (2026-11-11); commands are copied from the workflows,
`app/src/test/RUNNING_TESTS.md`, `e2e/run.sh` and the `package.json` files.

Paths in this note are relative to the repository root (`.github/workflows/release.yml`,
`gradle.properties`, `app/build.gradle.kts`). No secret values appear here: secrets are named only,
and their contents live in the GitHub repository settings, `local.properties` or
`app/google-services.json`, none of which is committed.

## Build

- One Gradle module, `:app` (`settings.gradle.kts`), Gradle `9.3.1`
  (`gradle/wrapper/gradle-wrapper.properties`), Android Gradle Plugin `9.1.1`, Kotlin `2.3.10`,
  JDK 21 for source and target (`app/build.gradle.kts:83`, `:84`). Library versions are listed in
  [[Tech Stack and Versions]].
- Three build types (`app/build.gradle.kts:59` to `:80`): `debug`, `release` (R8 minify and resource
  shrinking on, `:67`, `:68`; signed with the debug signing config for now, `:69`) and `e2e`, which
  starts from `debug`, installs as `com.example.messenger.e2e` and is the `testBuildType` (`:81`).
  Crashlytics collection is off for `debug` and `e2e` and on for `release` through a manifest
  placeholder (`:61`, `:65`), and the off-by-default feature list differs per build type
  (`:62`, `:66`); see [[Feature Flags and Remote Config]].
- `gradle.properties` carries `android.onlyEnableUnitTestForTheTestedBuildType=false`. Because
  `testBuildType` is `e2e`, Android Gradle Plugin would otherwise create the unit-test task only for
  `e2e`; without the flag `testDebugUnitTest` does not exist and CI fails with "Task
  'testDebugUnitTest' not found". A develop merge once dropped the line and broke CI (fix
  `d82f95c`, 2026-11-10).
- Local-only inputs, all gitignored: `local.properties` (also the place for `AGORA_APP_ID`),
  `app/google-services.json` (`.gitignore:155`, `:156`), keystores. `app/src/e2e/google-services.json`
  is the one committed exception (`.gitignore:157`): a fake `demo-messenger` project used only by the
  `e2e` build, see [[E2E Tests]]. `AGORA_APP_ID` resolves from `local.properties`, then a Gradle
  project property, then an empty string (`app/build.gradle.kts:20` to `:27`); a build without it
  compiles and launches but calls do not connect.
- `app/objectbox-models/default.json` is pinned to LF endings by `.gitattributes`, and the unit-test
  task declares `remoteconfig.template.json` as an input (`app/build.gradle.kts:113`).

## Versioning

The version lives in one place, `gradle.properties`: `VERSION_NAME=0.2.2.4` and
`VERSION_CODE=20204` (lines 34 and 35). `app/build.gradle.kts` reads them with
`providers.gradleProperty("VERSION_CODE")` and `("VERSION_NAME")` (`:45`, `:46`); nothing is
hard-coded in the build file any more (the previous `versionCode = 11301` / `versionName =
"0.1.13.1"` lines were replaced by commit `fb2b0f5` on 2026-10-18).

The code is derived from the name: `major * 1_000_000 + minor * 10_000 + patch * 100 + hotfix`, so
`0.2.2.4` is `0 + 20_000 + 200 + 4 = 20204`, and each part after major must be below 100. The release
workflow enforces that when it sets a version (`.github/workflows/release.yml:72` to `:101`): it
accepts `major.minor.patch` or `major.minor.patch.hotfix`, computes the code (`:90`), and fails if
the new code is not above the one in `gradle.properties` (`:92`) or if the tag already exists
(`:96`).

## CI workflows

Two workflows live in `.github/workflows/`. Both use JDK 21 (Temurin), `gradle/actions/setup-gradle@v4`
and `actions/checkout@v4` on `ubuntu-latest`.

### `blank.yml` - "Android CI"

The file is called `blank.yml` but its `name:` is `Android CI` (`blank.yml:1`). It runs on pull
requests into `main` or `develop` and on manual dispatch (`:3` to `:6`); a newer run for the same ref
cancels the older one (`:8` to `:10`). It was narrowed from "every push" to pull requests only in
PR #36 (commit `6b32fce`, 2026-10-21).

Steps, in order:

1. Write `app/google-services.json`: from the `GOOGLE_SERVICES_JSON` secret (base64) when it is
   available, otherwise a placeholder file with a dummy project id, because pull requests from forks
   get no secrets and unit tests never contact Firebase (`:32` to `:71`).
2. `./gradlew testDebugUnitTest` (`:74`), then upload `app/build/reports/tests/testDebugUnitTest/`
   as the `test-report` artifact, even if tests failed (`:76` to `:81`).
3. `./gradlew koverXmlReportDebug koverHtmlReportDebug` (`:84`), then upload `app/build/reports/kover/`
   as `coverage-report` (`:86` to `:90`).
4. `./gradlew koverVerifyDebug` (`:93`): the coverage gate. The rule is `minBound(28)` in
   `app/build.gradle.kts:268` to `:272`, so CI fails when line coverage drops below 28%.

This workflow runs only the JVM tests. It does not run the Cloud Functions tests, the Firestore rules
tests, the device tests or the end-to-end scenarios (the last is a known follow-up, see [[E2E Tests]]).

### `release.yml` - "Release"

Triggers (`:3` to `:12`): pushing a tag that matches `[0-9]+.[0-9]+.[0-9]+*`, or a manual dispatch
with an optional `version` input ("Leave empty for a test build without a release"). Runs never
cancel each other (`cancel-in-progress: false`, `:14` to `:16`) and the job may write repository
contents (`:18`, `:19`).

Steps, in order (`:26` to `:180`):

1. Check out, set up JDK 21 and Gradle, make `gradlew` executable.
2. Configure Gradle memory: append `org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=1g` and
   `kotlin.compiler.execution.strategy=in-process` to the runner's Gradle user home (`:41` to `:48`),
   so the release tests and the build share one Gradle daemon within a memory budget (the memory
   settings arrived with the tag-driven workflow in `fb2b0f5`; PR #16, `7616d52`, is the "share one
   Gradle daemon" change).
3. `GOOGLE_SERVICES_JSON`: required here, the run fails without the secret (`:50` to `:58`).
4. `DEBUG_KEYSTORE_BASE64`: required; written to the runner's `debug.keystore` so CI builds carry the
   same signature as local ones (`:60` to `:70`).
5. If a `version` input was given: refuse unless the run is on the default branch (`:78`), validate
   and convert the version, check the tag is free, and rewrite `VERSION_CODE` and `VERSION_NAME` in
   `gradle.properties` with `sed` (`:100`).
6. Read the version back from `gradle.properties` and decide the release tag: the pushed tag, or the
   requested version (`:103` to `:122`).
7. For a tag push, fail unless the tag equals `VERSION_NAME` (`:124` to `:130`).
8. `./gradlew testDebugUnitTest` (`:132` to `:133`), then `./gradlew assembleRelease bundleRelease`
   with `AGORA_APP_ID` passed from the secret of that name as `ORG_GRADLE_PROJECT_AGORA_APP_ID`
   (a warning if it is unset: calls will not work in that build) (`:135` to `:142`).
9. Copy the outputs to `dist/Switch-<version>.apk` and `dist/Switch-<version>.aab` (`:147`, `:148`).
10. Only when a version was requested: commit the version bump as `chore(release): bump version to
    <tag>`, create an annotated tag, and push the branch and the tag in one atomic push
    (`:151` to `:161`). The tag therefore exists only if the tests and the build passed.
11. Whenever there is a release tag: `gh release create <tag> dist/*` with generated notes and
    `--verify-tag` (`:163` to `:173`).
12. With no tag (a manual run with an empty version): upload `dist/` as an artifact named
    `Switch-<version>` instead (`:175` to `:180`), a test build that publishes nothing.

## Release flow

Two routes produce a release; the second is the one added in PR #36 (`bbc6a71`, 2026-10-21).

1. Tag a prepared commit. Bump `gradle.properties` on a branch, merge to `main`, push a tag equal to
   `VERSION_NAME`. The workflow checks the match (step 7), builds, and publishes.
2. Dispatch from `main` with a version. Run "Release" on `main` and type `0.2.3.0`: the workflow
   edits `gradle.properties`, builds, commits the bump, tags and publishes. The version-bump commit
   `9dcf57f` (`chore(release): bump version to 0.2.1.0`, 2026-10-21) is the one made this way, by
   `github-actions[bot]`.

What the history shows for the 0.2.x line (`git log`, tags): `0.2.0.0` came from `release/0.2.0.0`
(PR #29); `0.2.2.0` from `chore/release-0.2.2.0` (PRs #55 and #56); `0.2.2.1` and `0.2.2.2` were
bumped on their own `chore/release-*` branches and folded into `chore/release-0.2.2.3` (PR #77);
the 0.2.2.4 release is `chore/release-0.2.2.4` merged as PR #87 (`d6374fb`), whose only change is
the two `gradle.properties` lines (`1af6676`). Those bump commits are authored by a person, not the
bot, so they were made by hand, and tags exist for `0.1.12`, `0.1.13`, `0.1.13.1`, `0.2.0.0` and
`0.2.2.0` to `0.2.2.3`. `git ls-remote --tags origin` lists exactly those plus
`backup/main-clean-slate`: there is no `0.2.1.0` tag and, as of 2026-11-11, **no `0.2.2.4` tag**.
So 0.2.2.4 exists only as the version bump merged to `main`, and because the tag is what triggers
the release workflow, the tag-driven path never ran for it and no GitHub release was published.
A `0.2.2.4` tag pushed later would still match `VERSION_NAME` and release normally. The branch naming
convention is `chore/release-<version>` for a version bump and `release/<version>` for the first
0.2.0.0 cut.

Release builds are signed with the debug key for now (`app/build.gradle.kts:69`) and `main` is the
default branch (the CI trigger was renamed from `master` on 2026-07-31).

## Running the tests

The repository has six kinds of test. `app/src/test/RUNNING_TESTS.md` is the long form for the first
three; the exact commands follow. On a memory-constrained machine close Android Studio first and add
the low-memory flags the document uses everywhere; below, `GRADLE` stands for

```
./gradlew -Pkotlin.compiler.execution.strategy=in-process -Dorg.gradle.jvmargs="-Xmx3072m -XX:MaxMetaspaceSize=768m -Dfile.encoding=UTF-8" --max-workers=2
```

| Kind | Where | Command |
|---|---|---|
| JVM unit, database integration and Robolectric screen tests | `app/src/test/` (130 Kotlin files) | `GRADLE :app:testDebugUnitTest` (CI uses plain `./gradlew testDebugUnitTest`) |
| One class | same | `GRADLE :app:testDebugUnitTest --tests "*ChatViewModelTest"`; narrow with `--tests "*ChatViewModelTest.typing*"` |
| Coverage report and gate | Kover | `GRADLE :app:testDebugUnitTest :app:koverHtmlReportDebug :app:koverVerifyDebug`; report at `app/build/reports/kover/htmlDebug/index.html` |
| Device tests (WCAG contrast, gestures, Keystore) | `app/src/androidTest/` (27 Kotlin files) | build `GRADLE :app:assembleE2e :app:assembleE2eAndroidTest`, then `adb install -r` both APKs and `adb shell am instrument -w -r ...` (see below) |
| End-to-end scenarios against the Firebase emulators | `app/src/androidTest/.../e2e/`, `e2e/run.sh` | `bash e2e/run.sh all`, or `bash e2e/run.sh signin` and the other scenario names |
| Cloud Functions | `functions/src/push.test.ts`, `functions/test/callAccess.test.js` | `cd functions && npm ci && npm test` |
| Firestore security rules | `firestore-tests/*.test.mjs` | `cd firestore-tests && npm ci && npm run emulator` (needs the Firebase CLI; Node 20 or newer) |

Details worth knowing:

- Filter patterns used in `RUNNING_TESTS.md`: `"*IntegrationTest"` (real ObjectBox store in a temp
  directory, `app/src/test/.../testutil/ObjectBoxExtension.kt`), `"*ContentTest"` (Robolectric
  screens, JUnit 4 through the Vintage engine, `sdk=35` in `app/src/test/resources/robolectric.properties`),
  `"*ViewModelTest"`, `"*UseCaseTest"`. Fakes live in `app/src/test/.../testutil/fakes/`.
- Device tests install as the separate app `com.example.messenger.e2e`, so they cannot touch a real
  install or its keys. Run one class with
  `adb shell am instrument -w -r -e class <fully.qualified.Class> com.example.messenger.e2e.test/androidx.test.runner.AndroidJUnitRunner`,
  or the whole device suite minus the end-to-end package with `-e notPackage com.example.messenger.e2e`.
  The test APK is `app/build/outputs/apk/androidTest/e2e/app-e2e-androidTest.apk`, the app APK
  `app/build/outputs/apk/e2e/app-e2e.apk`. Never run `connectedDebugAndroidTest`: it uninstalls the
  app and wipes its data and encryption keys. `RUNNING_TESTS.md` notes 21 known contrast failures in
  the device suite at the time it was written.
- `e2e/run.sh` builds both APKs, starts the Auth, Firestore and Realtime Database emulators
  (`firebase emulators:exec --only auth,firestore,database --project demo-messenger`), forwards their
  ports to the device, and runs the scenarios `signin`, `messages`, `reinstall-blockstore`,
  `reinstall-code` and `reinstall-fresh`; `stale-key` is left out of `all` because it fails on a known
  bug. It needs the Firebase CLI and an unlocked device. `bash e2e/run.sh emulators` and
  `bash e2e/run.sh reset` exist for running from Android Studio.
- Cloud Functions: `functions/package.json` defines `npm test` as `npm run build && node --test
  lib/push.test.js test/callAccess.test.js`, so it compiles TypeScript first; Node 20.
- Firestore rules: `npm run emulator` wraps `firebase --config ../firebase.json emulators:exec
  --only firestore --project demo-messenger "npm test"`; the tests (`devices.test.mjs`,
  `keys.test.mjs`) load `firestore.rules` through `env.mjs` and run with `node --test`.
- The repo's unit-test count in `RUNNING_TESTS.md` (490) predates later work; see
  [[Tech Stack and Versions]] for the current figures.

### Coverage setup

Kover `0.9.11` is applied in `app/build.gradle.kts:13`. The `kover { }` block (`:248` to `:273`)
excludes generated and wiring classes (`*_`, `*.MyObjectBox`, `*Cursor`, Hilt and Dagger output,
`*ComposableSingletons*`, `*.BuildConfig`) and the whole `com.example.messenger.di` package, then
sets the gate to a 28% minimum. The CI uploads both the XML and the HTML report. The plan that
introduced all this is [[Test Coverage - Implementation Plan]].

## What this note could not verify

- The repository settings: which secrets are actually set (`GOOGLE_SERVICES_JSON`,
  `DEBUG_KEYSTORE_BASE64`, `AGORA_APP_ID` are the names the workflows read) and the branch
  protection rules.
- That the workflows pass: they were read, not run.

## Related

[[Tech Stack and Versions]] - [[E2E Tests]] - [[Feature Flags and Remote Config]] -
[[Test Coverage - Implementation Plan]] - [[Architecture Overview]] - [[Home]]
