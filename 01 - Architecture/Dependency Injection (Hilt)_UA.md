---
tags:
  - _UA
---
# Впровадження залежностей (Hilt)

## Що і версія

Hilt `2.59.2` (`gradle/libs.versions.toml:19`), обробляється через KSP
(`app/build.gradle.kts:138`, `ksp(libs.hilt.compiler)`). Gradle-плагін Hilt застосовано в
`app/build.gradle.kts:9`. `androidx.hilt:hilt-navigation-compose` `1.3.0` постачає
`hiltViewModel()` для Compose, а `androidx.hilt:hilt-work` + `androidx.hilt:hilt-compiler`
постачають підтримку `@HiltWorker`.

## Точки входу в граф

- `MessengerApplication` - це `@HiltAndroidApp` (`MessengerApplication.kt:34`) - корінь, що
  запускає генерацію компонентів і тримає `SingletonComponent`.
- Активності - це `@AndroidEntryPoint` (`MainActivity`, `IncomingCallActivity`,
  `BubbleActivity`).
- `CallForegroundService` - це `@AndroidEntryPoint` (`data/remote/call/CallForegroundService.kt:45`),
  тож він може `@Inject` `ICallService`, `ICallSignalingService`, `MissedCallRecorder`,
  `AgoraTokenProvider`.
- ViewModels - це `@HiltViewModel` з `@Inject constructor` (`ChatViewModel`, `AuthViewModel`,
  `ConversationsViewModel`, `CallViewModel`); UI отримує їх через `hiltViewModel()`
  (`presentation/navigation/navigation.kt:81`).

## Модулі (`di/`, 10 файлів)

Усі встановлені в `SingletonComponent`, тож прив'язки - singleton рівня процесу.

- `AppModule` (`di/AppModule.kt`) - `@Provides` application `Context` та IO-`CoroutineDispatcher`
  (`Dispatchers.IO`).
- `ObjectBoxModule` (`di/ObjectBoxModule.kt`) - `@Provides @Singleton` `BoxStore` (із
  debug-відновленням схеми, `:29`) та по одному `Box<Obx*>` на сутність (`:42` і далі).
- `RepositoryModule` (`di/RepositoryModule.kt`) - `abstract class`, що через `@Binds @Singleton`
  прив'язує кожен `*Impl` до його інтерфейсу `I*Repository` (`AuthRepositoryImpl -> IAuthRepository`
  тощо, плюс `LocalRepositoryImpl -> ILocalRepository`).
- `CallModule` (`di/CallModule.kt`) - `@Binds` `AgoraCallService -> ICallService` та
  `FirebaseCallSignalingService -> ICallSignalingService`.
- `CryptoModule` (`di/CryptoModule.kt`) - `@Provides` `KeystoreKeyWrapper`, `IdentityKeyStore` з
  коренем у `filesDir/e2ee` та `ObxParticipantResolver`.
- `FirebaseModule` - `@Provides` singleton-и Firebase (`FirebaseAuth`, `FirebaseFirestore`,
  `FirebaseDatabase`, `FirebaseStorage`, `FirebaseMessaging`).
- `RealtimeModule` - прив'язки сервісів присутності/набору.
- `NetworkModule` - прив'язка спостерігача мережі.
- `UseCaseModule` - постачання use case (більшість конструкторно впроваджені).
- `WorkManagerModule` - постачання, пов'язане з WorkManager, для sync-воркера.

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

## Ручний EntryPoint для класів, що створює фреймворк

`MyFirebaseMessagingService` не може чисто використати `@AndroidEntryPoint`, бо ОС конструює
`FirebaseMessagingService`. Натомість він оголошує Hilt `@EntryPoint`-інтерфейс і витягує залежності
з графа на вимогу (`data/remote/firebase/MyFirebaseMessagingService.kt:23`):

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
[[Tech Stack and Versions_UA|Технологічний стек і версії]]
