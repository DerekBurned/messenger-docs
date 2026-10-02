---
tags:
  - _UA
---
# Присутність і набір тексту

Присутність online/away/offline та індикатори набору по розмовах, обидва поверх Firebase Realtime
Database (обрано над Firestore для цього, бо RTDB має нативний `onDisconnect` і дешеві
високочастотні записи).

## Складові

- `RealtimeDbService` (`data/remote/firebase/RealtimeDbService.kt`) - сирі читання/записи RTDB.
- `FirebasePresenceService` (`data/service/FirebasePresenceService.kt`) - `IPresenceService`.
- `FirebaseTypingService` (`data/service/FirebaseTypingService.kt`) - `ITypingService`.
- `PresenceManager` (`data/presence/PresenceManager.kt`) - керує присутністю поточного користувача.
- `AppLifecycleObserver` (`data/presence/AppLifecycleObserver.kt`) - хук foreground/background.
- Use case: `ObserveUserPresenceUseCase`, `ObserveTypingUseCase`.
- UI: `PresenceIndicator`, `TypingIndicator`.

## Модель даних присутності

Присутність записується під `presence/{uid}` у RTDB. `RealtimeDbService.setPresence`
(`RealtimeDbService.kt:37`) пише `{ state, lastSeen, lastChanged: ServerValue.TIMESTAMP }`.
Ключовий механізм надійності - `setupOnDisconnect` (`RealtimeDbService.kt:46`):

```kotlin
fun setupOnDisconnect(userId: String) {
    val offlineData = mapOf("state" to "offline", "lastSeen" to ServerValue.TIMESTAMP, ...)
    presenceRef.child(userId).onDisconnect().setValue(offlineData)
}
```

`onDisconnect()` виконується сервером Firebase, коли сокет клієнта обривається, тож крах чи втрата
мережі все одно перемикають користувача офлайн із серверним timestamp - жодного клієнтського
heartbeat не потрібно. `cancelOnDisconnect` очищає це при чистому виході.

## Керування присутністю поточного користувача

`PresenceManager` (`data/presence/PresenceManager.kt:16`, `@Singleton`) перетворює події життєвого
циклу на записи присутності через конфлюючий канал (`transitions: Channel<PresenceState>`, `:24`).
Збирач (`ensureCollector`, `:77`) застосовує кожен запитаний стан через
`presenceService.setMyPresence(state)`, пропускаючи no-op повтори (`lastAppliedState`), і викликає
`setupOnDisconnect` при переході в ONLINE.

Переходи:
- `goOnline` (`:30`) - ONLINE, скасовує будь-який відкладений таймер офлайн.
- `goAway` (`:37`) - AWAY зараз, потім 5-хвилинний таймер у OFFLINE.
- `goIdleAway` (`:47`) - AWAY без авто-офлайн.
- `goOffline` / `disconnect` (`:54`, `:61`) - OFFLINE / прибрати присутність.

`Channel.CONFLATED` (`:24`) означає, що швидкий сплеск переходів згортається до останнього перед
застосуванням, уникаючи бурі записів RTDB на швидких перемиканнях foreground/background. Менеджер
викликається `AuthViewModel` при вході/виході (`goOnline`/`disconnect`, див.
[[Authentication_UA|Автентифікація]]) та `AppLifecycleObserver` при foreground/background процесу,
зв'язаний у `MessengerApplication.onCreate` (`MessengerApplication.kt:101`). Переданий scope - це
scope викликача (`viewModelScope` з auth, або scope спостерігача), тож менеджер ніколи не володіє
власним scope.

## Спостереження за присутністю інших користувачів

`FirebasePresenceService.observePresence` (`FirebasePresenceService.kt:21`) маппить RTDB-потік
`ValueEventListener` у `UserPresence(state, lastSeen)`. `observeMultiplePresence` (`:30`)
`combine`-ить список потоків по користувачах в єдиний `Map<String, UserPresence>` - використовується
списком розмов, щоб підсвітити кожен рядок (див. [[Conversations List_UA|Список розмов]]). У чаті
`ChatViewModel.observePartnerPresence` (`presentation/viewmodel/ChatViewModel.kt:400`) `combine`-ить
потік присутності з 60-секундним ticker, тож напис "останній раз у мережі N хвилин тому"
оновлюється, навіть коли присутність сама не змінилася. Прибирання `callbackFlow` + `awaitClose` в
[[Async and Reactivity_UA|Асинхронність і реактивність]].

## Набір тексту

Набір записується під `typing/{conversationId}/{uid}` у RTDB. Як і присутність, вузол
самоочищується: він прибирається, чи то користувач виходить з чату, згортає застосунок, чи процес
убито - тож співрозмовник ніколи не бачить назавжди застряглий "набирає...".

- Вихідний: `ChatViewModel.onTextChanged` (`presentation/viewmodel/ChatViewModel.kt:445`) дебаунсить
  500 мс перед `typingService.setTyping`, із 5-секундним TTL-очищенням та миттєвим очищенням при
  порожньому вводі. `FirebaseTypingService.setTyping` (`FirebaseTypingService.kt:28`) пише вузол
  timestamp для поточного користувача.
- Самоочищення при розриві сокета: `RealtimeDbService.setTyping` (`RealtimeDbService.kt:79`)
  реєструє `onDisconnect().removeValue()` на вузлі *перед* записом, дзеркалячи `setupOnDisconnect`
  для присутності. Збій, втрата мережі чи вбивство процесу змушують сервер Firebase прибрати вузол
  набору. `clearTyping` (`:85`) скасовує цей обробник перед видаленням вузла при чистому очищенні.
- Очищення при виході з екрана: `ChatViewModel.onCleared` (`ChatViewModel.kt:561`) викликає
  несуспендний `typingService.clearTypingNow` (`FirebaseTypingService.kt:38` ->
  `RealtimeDbService.kt:91`). `viewModelScope` там уже скасовано, тож очищення не може бути
  suspend-викликом у ньому; `clearTypingNow` виконує `removeValue()` без очікування й скасовує
  обробник розриву. Це покриває вихід із чату, коли застосунок лишається під'єднаним - випадок, де
  `onDisconnect` ніколи б не спрацював.
- Вхідний: `FirebaseTypingService.observeTyping` (`FirebaseTypingService.kt:17`) маппить дочірні
  вузли RTDB у множину uid, що набирають, відфільтровує поточного користувача й захисно відкидає
  будь-який вузол, чий `timestamp` старший за 15 с (`isFresh`, `:43`) - страховка для застарілих
  записів з часів до самоочищення:
  ```kotlin
  data.entries
      .filter { it.key != currentUserId && isFresh(it.value, now) }
      .map { it.key }
      .toSet()
  ```
  `ChatViewModel.observeTyping` (`ChatViewModel.kt:430`) збирає це в `isPartnerTyping` /
  `typingUsernames`, показане як напис "набирає..." у шапці чату.

## Чому Realtime Database (а не Firestore) для цього

- `onDisconnect()` дає серверне очищення безкоштовно, чого Firestore не пропонує - критично, щоб
  убитий застосунок усе одно показувався офлайн *і* припиняв показувати застарілий індикатор набору.
- Присутність і набір - це високочастотні, крихітні, ефемерні записи; модель RTDB для цього дешевша
  й з нижчою затримкою, ніж записи документів Firestore.
- Firestore лишається сховищем запису для повідомлень і розмов; RTDB використовується лише для
  тимчасових сигналів (присутність, набір і сигналізація дзвінків у [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]]).

## Пов'язане

[[Conversations List_UA|Список розмов]] - [[Chat and Messaging_UA|Чат і обмін повідомленнями]] -
[[Authentication_UA|Автентифікація]] - [[Calls (Voice and Video)_UA|Дзвінки (голос і відео)]] -
[[Async and Reactivity_UA|Асинхронність і реактивність]] - [[Data Layer_UA|Шар даних]]
