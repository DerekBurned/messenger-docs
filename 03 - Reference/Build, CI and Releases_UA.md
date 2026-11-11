---
tags:
  - _UA
---
# Збірка, CI і релізи

Як збирається застосунок, як задається номер його версії, що роблять два воркфлоу GitHub Actions, як
випускається реліз і як запустити кожен вид тестів, що є в репозиторії. Усе тут прочитано з
репозиторію на релізі 0.2.2.4 (2026-11-11); команди скопійовано з воркфлоу,
`app/src/test/RUNNING_TESTS.md`, `e2e/run.sh` і файлів `package.json`.

Шляхи в цій нотатці - відносно кореня репозиторію (`.github/workflows/release.yml`,
`gradle.properties`, `app/build.gradle.kts`). Значень секретів тут немає: секрети лише названо, а їхній
вміст лежить у налаштуваннях репозиторію GitHub, `local.properties` чи `app/google-services.json`, жоден
із яких не комітиться.

## Збірка

- Один Gradle-модуль, `:app` (`settings.gradle.kts`), Gradle `9.3.1`
  (`gradle/wrapper/gradle-wrapper.properties`), Android Gradle Plugin `9.1.1`, Kotlin `2.3.10`, JDK 21 для
  джерел і цілі (`app/build.gradle.kts:83`, `:84`). Версії бібліотек - у
  [[Tech Stack and Versions_UA|Технологічний стек і версії]].
- Три типи збірки (`app/build.gradle.kts:59` - `:80`): `debug`, `release` (R8-minify і стиснення
  ресурсів увімкнені, `:67`, `:68`; поки що підписується debug-конфігурацією підпису, `:69`) та `e2e`, що
  починається з `debug`, встановлюється як `com.example.messenger.e2e` і є `testBuildType` (`:81`).
  Збір Crashlytics вимкнено для `debug` та `e2e` і ввімкнено для `release` через manifest-плейсхолдер
  (`:61`, `:65`), а список функцій, вимкнених за замовчуванням, різний для кожного типу збірки (`:62`,
  `:66`); див. [[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]].
- `gradle.properties` містить `android.onlyEnableUnitTestForTheTestedBuildType=false`. Оскільки
  `testBuildType` - це `e2e`, Android Gradle Plugin інакше створив би завдання юніт-тестів лише для
  `e2e`; без цього прапорця `testDebugUnitTest` не існує, і CI падає з "Task 'testDebugUnitTest' not
  found". Злиття develop колись загубило цей рядок і зламало CI (виправлення `d82f95c`, 2026-11-10).
- Лише локальні вхідні дані, усі в gitignore: `local.properties` (також місце для `AGORA_APP_ID`),
  `app/google-services.json` (`.gitignore:155`, `:156`), сховища ключів. `app/src/e2e/google-services.json` -
  єдиний закомічений виняток (`.gitignore:157`): фальшивий проєкт `demo-messenger`, що використовується
  лише збіркою `e2e`, див. [[E2E Tests]]. `AGORA_APP_ID` береться з `local.properties`, потім із Gradle
  project property, потім порожній рядок (`app/build.gradle.kts:20` - `:27`); збірка без нього
  компілюється й запускається, але дзвінки не з'єднуються.
- `app/objectbox-models/default.json` закріплено на кінцях рядків LF у `.gitattributes`, а завдання
  юніт-тестів оголошує `remoteconfig.template.json` своїм входом (`app/build.gradle.kts:113`).

## Версіонування

Версія живе в одному місці, `gradle.properties`: `VERSION_NAME=0.2.2.4` і `VERSION_CODE=20204`
(рядки 34 і 35). `app/build.gradle.kts` читає їх через `providers.gradleProperty("VERSION_CODE")` і
`("VERSION_NAME")` (`:45`, `:46`); у build-файлі більше нічого не захардкоджено (попередні рядки
`versionCode = 11301` / `versionName = "0.1.13.1"` замінив коміт `fb2b0f5` 2026-10-18).

Код виводиться з імені: `major * 1_000_000 + minor * 10_000 + patch * 100 + hotfix`, тож `0.2.2.4` - це
`0 + 20_000 + 200 + 4 = 20204`, і кожна частина після major має бути менша за 100. Воркфлоу релізу
перевіряє це, коли задає версію (`.github/workflows/release.yml:72` - `:101`): приймає
`major.minor.patch` або `major.minor.patch.hotfix`, обчислює код (`:90`) і падає, якщо новий код не
вищий за той, що в `gradle.properties` (`:92`), або якщо тег уже існує (`:96`).

## CI-воркфлоу

У `.github/workflows/` лежать два воркфлоу. Обидва використовують JDK 21 (Temurin),
`gradle/actions/setup-gradle@v4` і `actions/checkout@v4` на `ubuntu-latest`.

### `blank.yml` - "Android CI"

Файл називається `blank.yml`, але його `name:` - `Android CI` (`blank.yml:1`). Він запускається на pull
request у `main` чи `develop` і на ручний запуск (`:3` - `:6`); новіший запуск для того ж ref скасовує
старший (`:8` - `:10`). З "кожного push" його звузили до самих pull request у PR #36 (коміт `6b32fce`,
2026-10-21).

Кроки по порядку:

1. Записати `app/google-services.json`: із секрету `GOOGLE_SERVICES_JSON` (base64), коли він
   доступний, інакше - файл-плейсхолдер із фіктивним id проєкту, бо pull request з форків не отримують
   секретів, а юніт-тести ніколи не звертаються до Firebase (`:32` - `:71`).
2. `./gradlew testDebugUnitTest` (`:74`), потім вивантажити `app/build/reports/tests/testDebugUnitTest/`
   як артефакт `test-report`, навіть якщо тести впали (`:76` - `:81`).
3. `./gradlew koverXmlReportDebug koverHtmlReportDebug` (`:84`), потім вивантажити
   `app/build/reports/kover/` як `coverage-report` (`:86` - `:90`).
4. `./gradlew koverVerifyDebug` (`:93`): поріг покриття. Правило - `minBound(28)` у
   `app/build.gradle.kts:268` - `:272`, тож CI падає, коли покриття рядків опускається нижче 28%.

Цей воркфлоу запускає лише JVM-тести. Він не запускає тести Cloud Functions, тести правил Firestore,
тести на пристрої чи наскрізні сценарії (останнє - відомий наступний крок, див. [[E2E Tests]]).

### `release.yml` - "Release"

Тригери (`:3` - `:12`): push тегу, що відповідає `[0-9]+.[0-9]+.[0-9]+*`, або ручний запуск із
необов'язковим входом `version` ("Leave empty for a test build without a release"). Запуски ніколи не
скасовують один одного (`cancel-in-progress: false`, `:14` - `:16`), а завдання може писати вміст
репозиторію (`:18`, `:19`).

Кроки по порядку (`:26` - `:180`):

1. Checkout, налаштувати JDK 21 і Gradle, зробити `gradlew` виконуваним.
2. Налаштувати пам'ять Gradle: дописати `org.gradle.jvmargs=-Xmx4g -XX:MaxMetaspaceSize=1g` і
   `kotlin.compiler.execution.strategy=in-process` у Gradle user home раннера (`:41` - `:48`), щоб
   тести релізу та збірка ділили один демон Gradle в межах бюджету пам'яті (налаштування пам'яті
   з'явилися з воркфлоу на тегах у `fb2b0f5`; PR #16, `7616d52`, - це зміна "share one Gradle daemon").
3. `GOOGLE_SERVICES_JSON`: тут обов'язковий, запуск падає без секрету (`:50` - `:58`).
4. `DEBUG_KEYSTORE_BASE64`: обов'язковий; записується в `debug.keystore` раннера, щоб CI-збірки мали
   той самий підпис, що й локальні (`:60` - `:70`).
5. Якщо задано вхід `version`: відмовити, якщо запуск не на гілці за замовчуванням (`:78`),
   перевірити й перетворити версію, перевірити, що тег вільний, і переписати `VERSION_CODE` та
   `VERSION_NAME` у `gradle.properties` через `sed` (`:100`).
6. Прочитати версію назад із `gradle.properties` і визначити тег релізу: надісланий тег або запитана
   версія (`:103` - `:122`).
7. Для push тегу впасти, якщо тег не дорівнює `VERSION_NAME` (`:124` - `:130`).
8. `./gradlew testDebugUnitTest` (`:132` - `:133`), потім `./gradlew assembleRelease bundleRelease` з
   `AGORA_APP_ID`, переданим із секрету з такою назвою як `ORG_GRADLE_PROJECT_AGORA_APP_ID`
   (попередження, якщо він не заданий: у цій збірці дзвінки не працюватимуть) (`:135` - `:142`).
9. Скопіювати результати в `dist/Switch-<version>.apk` і `dist/Switch-<version>.aab` (`:147`, `:148`).
10. Лише коли запитано версію: закомітити підвищення версії як `chore(release): bump version to <tag>`,
    створити анотований тег і надіслати гілку та тег одним атомарним push (`:151` - `:161`). Тож тег
    існує лише якщо тести й збірка пройшли.
11. Щоразу, коли є тег релізу: `gh release create <tag> dist/*` зі згенерованими нотатками та
    `--verify-tag` (`:163` - `:173`).
12. Без тегу (ручний запуск із порожньою версією): натомість вивантажити `dist/` як артефакт
    `Switch-<version>` (`:175` - `:180`) - тестова збірка, що нічого не публікує.

## Потік релізу

Реліз дають два шляхи; другий додано в PR #36 (`bbc6a71`, 2026-10-21).

1. Позначити підготовлений коміт тегом. Підняти `gradle.properties` у гілці, злити в `main`, надіслати
   тег, що дорівнює `VERSION_NAME`. Воркфлоу перевіряє збіг (крок 7), збирає й публікує.
2. Запустити з `main` із версією. Запустити "Release" на `main` і ввести `0.2.3.0`: воркфлоу редагує
   `gradle.properties`, збирає, комітить підвищення, ставить тег і публікує. Коміт підвищення версії
   `9dcf57f` (`chore(release): bump version to 0.2.1.0`, 2026-10-21) зроблено саме так, від імені
   `github-actions[bot]`.

Що показує історія для лінії 0.2.x (`git log`, теги): `0.2.0.0` прийшов із `release/0.2.0.0` (PR #29);
`0.2.2.0` - із `chore/release-0.2.2.0` (PR #55 і #56); `0.2.2.1` та `0.2.2.2` піднімали у власних гілках
`chore/release-*` і влили в `chore/release-0.2.2.3` (PR #77); реліз 0.2.2.4 - це `chore/release-0.2.2.4`,
злита як PR #87 (`d6374fb`), єдина зміна якої - два рядки `gradle.properties` (`1af6676`). Ці коміти
підвищення має за автора людина, а не бот, тож їх зроблено вручну, а теги існують для
`0.1.12`, `0.1.13`, `0.1.13.1`, `0.2.0.0` і `0.2.2.0` - `0.2.2.3`. `git ls-remote --tags origin`
перелічує саме їх плюс `backup/main-clean-slate`: тега `0.2.1.0` немає, і станом на 2026-11-11
**тега `0.2.2.4` теж немає**. Тобто 0.2.2.4 існує лише як підвищення версії, влите в `main`, а
оскільки саме тег запускає реліз-воркфлоу, керований тегом шлях для нього не виконувався і реліз
на GitHub не публікувався. Тег `0.2.2.4`, запушений пізніше, усе одно збігатиметься з `VERSION_NAME`
і випустить реліз у звичайний спосіб. Конвенція назв гілок - `chore/release-<version>` для підвищення версії та `release/<version>` для
першого зрізу 0.2.0.0.

Release-збірки поки що підписуються debug-ключем (`app/build.gradle.kts:69`), а `main` - гілка за
замовчуванням (тригер CI перейменовано з `master` 2026-07-31).

## Запуск тестів

У репозиторії шість видів тестів. `app/src/test/RUNNING_TESTS.md` - розгорнута форма для перших трьох;
нижче точні команди. На машині з малою пам'яттю спершу закрийте Android Studio й додайте прапорці
низької пам'яті, які документ використовує всюди; нижче `GRADLE` означає

```
./gradlew -Pkotlin.compiler.execution.strategy=in-process -Dorg.gradle.jvmargs="-Xmx3072m -XX:MaxMetaspaceSize=768m -Dfile.encoding=UTF-8" --max-workers=2
```

| Вид | Де | Команда |
|---|---|---|
| JVM-юніт, інтеграційні з базою та екранні тести Robolectric | `app/src/test/` (130 файлів Kotlin) | `GRADLE :app:testDebugUnitTest` (CI використовує простий `./gradlew testDebugUnitTest`) |
| Один клас | те саме | `GRADLE :app:testDebugUnitTest --tests "*ChatViewModelTest"`; звузити можна через `--tests "*ChatViewModelTest.typing*"` |
| Звіт покриття та поріг | Kover | `GRADLE :app:testDebugUnitTest :app:koverHtmlReportDebug :app:koverVerifyDebug`; звіт у `app/build/reports/kover/htmlDebug/index.html` |
| Тести на пристрої (контраст WCAG, жести, Keystore) | `app/src/androidTest/` (27 файлів Kotlin) | зібрати `GRADLE :app:assembleE2e :app:assembleE2eAndroidTest`, потім `adb install -r` обох APK і `adb shell am instrument -w -r ...` (нижче) |
| Наскрізні сценарії проти емуляторів Firebase | `app/src/androidTest/.../e2e/`, `e2e/run.sh` | `bash e2e/run.sh all`, або `bash e2e/run.sh signin` та інші назви сценаріїв |
| Cloud Functions | `functions/src/push.test.ts`, `functions/test/callAccess.test.js` | `cd functions && npm ci && npm test` |
| Правила безпеки Firestore | `firestore-tests/*.test.mjs` | `cd firestore-tests && npm ci && npm run emulator` (потрібен Firebase CLI; Node 20 або новіший) |

Що варто знати:

- Шаблони фільтрів із `RUNNING_TESTS.md`: `"*IntegrationTest"` (справжнє сховище ObjectBox у тимчасовій
  теці, `app/src/test/.../testutil/ObjectBoxExtension.kt`), `"*ContentTest"` (екрани Robolectric,
  JUnit 4 через рушій Vintage, `sdk=35` у `app/src/test/resources/robolectric.properties`),
  `"*ViewModelTest"`, `"*UseCaseTest"`. Фейки лежать у `app/src/test/.../testutil/fakes/`.
- Тести на пристрої встановлюються як окремий застосунок `com.example.messenger.e2e`, тож не можуть
  зачепити справжню інсталяцію чи її ключі. Один клас запускають так:
  `adb shell am instrument -w -r -e class <fully.qualified.Class> com.example.messenger.e2e.test/androidx.test.runner.AndroidJUnitRunner`,
  а весь набір для пристрою без наскрізного пакета - з `-e notPackage com.example.messenger.e2e`. APK
  тестів - `app/build/outputs/apk/androidTest/e2e/app-e2e-androidTest.apk`, APK застосунку -
  `app/build/outputs/apk/e2e/app-e2e.apk`. Ніколи не запускайте `connectedDebugAndroidTest`: він
  видаляє застосунок і стирає його дані та ключі шифрування. `RUNNING_TESTS.md` згадує 21 відомий збій
  контрасту в наборі для пристрою на момент написання.
- `e2e/run.sh` збирає обидва APK, запускає емулятори Auth, Firestore і Realtime Database
  (`firebase emulators:exec --only auth,firestore,database --project demo-messenger`), пробрасує їхні
  порти на пристрій і запускає сценарії `signin`, `messages`, `reinstall-blockstore`, `reinstall-code` і
  `reinstall-fresh`; `stale-key` виключено з `all`, бо він падає через відому помилку. Потрібні
  Firebase CLI і розблокований пристрій. `bash e2e/run.sh emulators` та `bash e2e/run.sh reset` існують
  для запуску з Android Studio.
- Cloud Functions: `functions/package.json` визначає `npm test` як `npm run build && node --test
  lib/push.test.js test/callAccess.test.js`, тож спершу компілюється TypeScript; Node 20.
- Правила Firestore: `npm run emulator` обгортає `firebase --config ../firebase.json emulators:exec
  --only firestore --project demo-messenger "npm test"`; тести (`devices.test.mjs`, `keys.test.mjs`)
  завантажують `firestore.rules` через `env.mjs` і запускаються через `node --test`.
- Кількість юніт-тестів у `RUNNING_TESTS.md` (490) випереджена подальшою роботою; поточні цифри - у
  [[Tech Stack and Versions_UA|Технологічний стек і версії]].

### Налаштування покриття

Kover `0.9.11` застосовано в `app/build.gradle.kts:13`. Блок `kover { }` (`:248` - `:273`) виключає
згенеровані та "зв'язувальні" класи (`*_`, `*.MyObjectBox`, `*Cursor`, вихід Hilt і Dagger,
`*ComposableSingletons*`, `*.BuildConfig`) і весь пакет `com.example.messenger.di`, а потім задає поріг
у мінімум 28%. CI вивантажує і XML-, і HTML-звіт. План, що все це запровадив, -
[[Test Coverage - Implementation Plan]].

## Що ця нотатка не змогла перевірити

- Налаштування репозиторію: які секрети справді задані (`GOOGLE_SERVICES_JSON`,
  `DEBUG_KEYSTORE_BASE64`, `AGORA_APP_ID` - назви, які читають воркфлоу) і правила захисту гілок.
- Чи проходять воркфлоу: їх прочитано, а не запущено.

## Пов'язане

[[Tech Stack and Versions_UA|Технологічний стек і версії]] - [[E2E Tests]] -
[[Feature Flags and Remote Config_UA|Feature flag-и та Remote Config]] -
[[Test Coverage - Implementation Plan]] - [[Architecture Overview_UA|Огляд архітектури]] -
[[Home_UA|Головна]]
