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

## Start here if you are new

Plain-language explainers, no background needed. Each has annotated code (with teaching
comments), a step-by-step data trace, a diagram, and a worked example.

Start with the patterns, then any feature:

- [[Reading the Code - Simple Patterns]] - the six patterns that repeat everywhere (the
  `_state`/`state` backing field, use cases, repositories, cold Flow).

Every feature has an "Explained Simply" companion:

- [[Authentication - Explained Simply]]
- [[Conversations List - Explained Simply]]
- [[Chat and Messaging - Explained Simply]]
- [[Calls (Voice and Video) - Explained Simply]]
- [[Presence and Typing - Explained Simply]]
- [[Notifications - Explained Simply]]
- [[Settings - Explained Simply]]
- [[End-to-End Encryption - Explained Simply]]
- [[Appearance and Theming - Explained Simply]]

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
- [[Appearance and Theming]] - light/dark/system theme mode, gradient and imported wallpapers,
  background intensity, the design-token layer, and the reusable motion modifiers.

## Feature code walkthroughs

Each feature also has a `- Code` companion that embeds and explains that feature's real source
(verified against the repo), block by block:

- [[Authentication - Code]]
- [[Conversations List - Code]]
- [[Chat and Messaging - Code]]
- [[Calls (Voice and Video) - Code]]
- [[Presence and Typing - Code]]
- [[Notifications - Code]]
- [[Settings - Code]]
- [[End-to-End Encryption - Code]]
- [[Appearance and Theming - Code]]

## Basic examples

Each feature also has a `- Basic Example` companion. It shows the *inherited* pattern the feature is
built on - the minimal, canonical, textbook form of the underlying library or base pattern (from
[[Reading the Code - Simple Patterns]]) - and then points at where the real, specialised source
differs. Read the basic example first to see the skeleton, then the `- Code` note for the real
thing.

- [[Authentication - Basic Example]]
- [[Conversations List - Basic Example]]
- [[Chat and Messaging - Basic Example]]
- [[Calls (Voice and Video) - Basic Example]]
- [[Presence and Typing - Basic Example]]
- [[Notifications - Basic Example]]
- [[Settings - Basic Example]]
- [[End-to-End Encryption - Basic Example]]
- [[Appearance and Theming - Basic Example]]

Every feature note also carries a `## Visualization` section with a Mermaid diagram of its core
data flow.

## Reference

- [[Tech Stack and Versions]] - every library and version from `gradle/libs.versions.toml`
  and `app/build.gradle.kts`, with why it is here.

## Conventions used in this vault

- Four-part notes where a technology decision is involved: What / How we use it / Why this
  choice / Alternatives considered.
- Function explanations state the goal first (why this signature, why this Flow, why this
  scope) and then walk the mechanics.
- No emojis. Absolute dates. `path:line` for code, wiki-links between notes.
