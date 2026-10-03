---
tags:
  - _UA
---
# Впровадження залежностей (Hilt)

## Що і версія

Hilt `2.59.2` (`gradle/libs.versions.toml:19`), обробляється через KSP
(`app/build.gradle.kts:131`, `ksp(libs.hilt.compiler)`). Gradle-плагін Hilt застосовано в
`app/build.gradle.kts:9` (`alias(libs.plugins.hilt.android)`). `androidx.hilt:hilt-navigation-compose` `1.3.0` постачає
`hiltViewModel()` для Compose, а `androidx.hilt:hilt-work` + `androidx.hilt:hilt-compiler`
постачають підтримку `@HiltWorker`.

## Точки входу в граф

- `MessengerApplication` - це `@HiltAndroidApp` (`MessengerApplication.kt:34`) - корінь, що
  запускає генерацію компонентів і тримає `SingletonComponent`.
- Активності - це `@AndroidEntryPoint` (`MainActivity`, `IncomingCallActivity`,
  `BubbleActivity`).
- `CallForegroundService` - це `@AndroidEntryPoint` (`data/remote/call/CallForegroundService.kt:50`),
  тож він може `@Inject` `ICallService`, `ICallSignalingService`, `MissedCallRecorder`,
  `AgoraTokenProvider`.
- ViewModels - це `@HiltViewModel` з `@Inject constructor` (`ChatViewModel`, `AuthViewModel`,
  `ChatsViewModel`, `CallViewModel`, `ClipPlayerViewModel`, ...); UI отримує їх через
  `hiltViewModel()` (`presentation/navigation/graph/AppNavigation.kt:27`).
- Класам з `@Inject constructor` і `@Singleton` взагалі не потрібен запис у модулі. Так зібрано
  голосовий конвеєр: `VoiceRecorder` (`data/media/VoiceRecorder.kt:77`), `ClipPlaybackController`
  (`data/media/ClipPlaybackController.kt:79`) і `VoiceDraftStore` (`data/media/VoiceDraftStore.kt:32`).

## Модулі (`di/`, 11 файлів)

Усі встановлені в `SingletonComponent`, тож прив'язки - singleton рівня процесу.

- `AppModule` (`di/AppModule.kt`) - `@Provides` application `Context` та IO-`CoroutineDispatcher`
  (`Dispatchers.IO`).
- `ObjectBoxModule` (`di/ObjectBoxModule.kt`) - `@Provides @Singleton` `BoxStore` (із
  debug-відновленням схеми, `:30`) та по одному `Box<Obx*>` на сутність (`:43` і далі).
- `RepositoryModule` (`di/RepositoryModule.kt`) - `abstract class`, що через `@Binds @Singleton`
  прив'язує кожен `*Impl` до його інтерфейсу `I*Repository` (`AuthRepositoryImpl -> IAuthRepository`,
  `ClipRepositoryImpl -> IClipRepository` на `:54` тощо, плюс
  `LocalRepositoryImpl -> ILocalRepository`).
- `CallModule` (`di/CallModule.kt`) - `@Binds` `AgoraCallService -> ICallService` та
  `FirebaseCallSignalingService -> ICallSignalingService`.
- `CryptoModule` (`di/CryptoModule.kt`) - `@Provides` `KeystoreKeyWrapper`, `IdentityKeyStore` з
  коренем у `filesDir/e2ee` та `ObxParticipantResolver`.
- `FirebaseModule` - `@Provides` singleton-и Firebase (`FirebaseAuth`, `FirebaseFirestore`,
  `FirebaseDatabase`, `FirebaseStorage`, `FirebaseMessaging`) і `FirebaseFunctions`, закріплений за
  `europe-west1` (`di/FirebaseModule.kt:51`), який використовують `AccountDeletionService`
  (`deleteAccount`) і `AgoraTokenProvider` (`getRtcToken`).
- `RealtimeModule` - прив'язки сервісів присутності/набору, плюс
  `InMemoryAuthFlowGate -> IAuthFlowGate` (`di/RealtimeModule.kt:35`) для шлюзу сесії.
- `NetworkModule` - прив'язка спостерігача мережі.
- `UseCaseModule` - `@Provides` для use case, чиї конструктори потребують обв'язки (26 функцій;
  наприклад `DeleteMessageUseCase(messageRepository, clipRepository)` і use case сесії/акаунта);
  решта впроваджуються конструктором.
- `WorkManagerModule` - постачання, пов'язане з WorkManager, для sync-воркера.
- `UiEntryPoints` (`di/UiEntryPoints.kt`) - Hilt `@EntryPoint`, що дає composable без ViewModel
  (`ActiveCallBar`, `CallBubble`, `WallpaperBackground`) доступ до `IUserRepository`,
  `ThemePreferenceStore` та `ICallService`.

## `@Binds` проти `@Provides` - правило, що тут застосовується

- `@Binds` (у модулях `abstract class`) використовується, коли надаване - це інтерфейс, реалізацію
  якого Hilt здатен сконструювати сам - не потрібен код фабрики, лише "коли хтось просить
  `IMessageRepository`, дай йому впроваджений `MessageRepositoryImpl`". Це `RepositoryModule` і
  `CallModule`.
- `@Provides` (у модулях `object`) використовується, коли конструювання потребує логіки або типу,
  який Hilt не може збудувати сам: `BoxStore` (потрібен білдер і обробка помилок), окремі `Box<T>`
  (будуються зі стору), singleton-и Firebase (отримуються зі статичних геттерів), `IdentityKeyStore`
  (потрібен шлях `File`). Це `ObjectBoxModule`, `AppModule`, `CryptoModule`, `FirebaseModule`.

## Скоупи

Усе - `@Singleton` у `SingletonComponent`, що відповідає дизайну застосунку: `BoxStore`,
репозиторії, сервіси Firebase, `AgoraCallService` і crypto-кодек - усі довгоживучі й спільні.
ViewModels скоуповані до свого навігаційного запису через `@HiltViewModel` + декоратор ViewModel-стору
Navigation 3 (див. [[Navigation_UA|Навігація]]), а не через Hilt-скоуп.

`dagger.Lazy` відкладає конструювання, коли singleton потрібен лише на рідкісному шляху.
`AndroidSessionCleaner` впроваджує `Lazy<ClipPlaybackController>`, `Lazy<VoiceRecorder>` і
`Lazy<VoiceDraftStore>` та викликає `.get()` лише всередині `clearOnLogout`, тож побудова use case
виходу не конструює голосові singleton-и, чиї блоки `init` запускають збирачі рівня застосунку
(стан дзвінка, життєвий цикл процесу, видалені кліпи).

## Ручний EntryPoint для класів, що створює фреймворк

`MyFirebaseMessagingService` не може чисто використати `@AndroidEntryPoint`, бо ОС конструює
`FirebaseMessagingService`. Натомість він оголошує Hilt `@EntryPoint`-інтерфейс і витягує залежності
з графа на вимогу (`data/remote/firebase/MyFirebaseMessagingService.kt:25`):

```kotlin
@EntryPoint
@InstallIn(SingletonComponent::class)
interface FcmEntryPoint {
    fun firestoreService(): FirestoreService
    fun telecomCallManager(): TelecomCallManager
    fun messageRepository(): IMessageRepository
    fun e2eeCodec(): E2eeMessageCodec
}
// ...
val entryPoint = EntryPointAccessors.fromApplication(applicationContext, FcmEntryPoint::class.java)
```

Це стандартний "запасний вихід" Hilt для компонента, який ви не конструюєте. Див.
[[Notifications_UA|Сповіщення]].

## Пов'язане

[[Architecture Overview_UA|Огляд архітектури]] - [[Data Layer_UA|Шар даних]] -
[[Domain Layer_UA|Доменний шар]] - [[Navigation_UA|Навігація]] - [[Notifications_UA|Сповіщення]] -
[[Voice Messages_UA|Голосові повідомлення]] - [[Tech Stack and Versions_UA|Технологічний стек і версії]]
