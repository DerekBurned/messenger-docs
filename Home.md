# Messenger documentation - Home

Map of content for the Messenger Android app. Every note is verified against the actual
source at `D:\StudioProjects\Messenger` (base package `com.example.messenger`,
single `:app` module). Code references are written as `path:line`; other notes are linked
with Obsidian wiki-links.

This vault documents the code that exists, not the code the spec imagines. When a rationale
was not recoverable from the source or history, the note says so instead of inventing one.

## What the app is

A one-to-one messenger: phone-number auth, real-time text and media chat, voice and video
calls, presence and typing indicators, offline-first local storage, push notifications with
inline reply and conversation bubbles, and optional end-to-end encryption. Kotlin, Jetpack
Compose UI, MVI presentation, Hilt DI, ObjectBox local persistence, Firebase backend
(Firestore + Realtime Database + Auth + Storage + Cloud Messaging + Functions), Agora RTC for
calls.

## Architecture

- [[Architecture Overview]] - the three layers, module layout, the dependency graph, how a
  message travels end to end, and app startup wiring.
- [[Presentation Layer (MVI)]] - the MVI contract (State / Intent / Effect), the
  `MviViewModel` base class, the two ViewModel styles actually in the code, the Root/Screen
  composable split, and how the UI collects state and events.
- [[Domain Layer]] - use cases as invokable classes, repository interfaces, service
  interfaces, and the domain models.
- [[Data Layer]] - repositories, ObjectBox persistence, the Firebase services, DTOs and
  mappers, the offline sync queue, and media transfer/caching.
- [[Dependency Injection (Hilt)]] - the module set, `@Binds` vs `@Provides`, scoping, and the
  manual `EntryPoint` used from framework-instantiated components.
- [[Async and Reactivity]] - Flow vs StateFlow vs SharedFlow, `callbackFlow` bridges,
  dispatcher and scope choices, with the real call site for each.
- [[Navigation]] - Navigation 3 typed routes, the `NavDisplay` back stack, and how nav
  arguments reach a ViewModel's `SavedStateHandle`.

## Features

- [[Authentication]] - phone-number OTP sign-in, account linking, session bootstrap.
- [[Conversations List]] - the chat list, offline-first stream, per-participant presence and
  avatars, delete-for-me vs delete-for-everyone.
- [[Chat and Messaging]] - send/receive, replies, media attachments, read receipts, paging,
  message deletion rules.
- [[Calls (Voice and Video)]] - Agora RTC, the foreground-service call state machine, the
  active-call bar with swipe/bubble/snap-back gestures, PiP video, and call history.
- [[Presence and Typing]] - online/away/offline presence and typing indicators over Realtime
  Database.
- [[Notifications]] - FCM handling, offline message persistence, MessagingStyle, conversation
  shortcuts and bubbles, inline reply.
- [[Settings]] - the settings screens and the DataStore-backed theme preference.
- [[End-to-End Encryption]] - X25519 + HKDF + AES-GCM message encryption, key registry, epoch
  rotation, safety numbers.

## Reference

- [[Tech Stack and Versions]] - every library and version from `gradle/libs.versions.toml`
  and `app/build.gradle.kts`, with why it is here.

## Conventions used in this vault

- Four-part notes where a technology decision is involved: What / How we use it / Why this
  choice / Alternatives considered.
- Function explanations state the goal first (why this signature, why this Flow, why this
  scope) and then walk the mechanics.
- No emojis. Absolute dates. `path:line` for code, wiki-links between notes.
