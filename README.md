# Messenger documentation

An Obsidian vault documenting the Messenger Android app: full architecture plus per-feature
deep dives, written by reading the actual source and checked against release 0.2.2.4 (2026-11-11).
Every code reference is a `path:line` into the app repository, relative to
`app/src/main/java/com/example/messenger/` unless the note says otherwise (repo-root paths such as
`functions/src/index.ts` or `firestore.rules` are used as they are); notes are cross-linked with
wiki-links.

Open this folder as a vault in Obsidian and start at [Home](Home.md).

## Layout

- `01 - Architecture/` - layers, MVI presentation, domain, data, DI, async/reactivity,
  navigation, and feature flags with Firebase Remote Config and Crashlytics. It also holds the
  plan, proposal and history notes (file-layout refactor, test-coverage plan, Paging 3 design,
  multi-device encryption research), each opening with a status line checked against 0.2.2.4.
- `02 - Features/` - authentication, conversations list, search and suggestions, chat,
  voice messages, video messages, chat profile and shared media, calls, presence and typing,
  notifications, settings, end-to-end encryption, appearance and theming. Each feature has a
  main note (with a Mermaid `## Visualization`), a `- Code` walkthrough of the real source, an
  `- Explained Simply` companion, and a `- Basic Example` showing the inherited base pattern.
- `03 - Reference/` - tech stack and versions (including the AGPL-3.0-only license), build, CI and
  releases (the GitHub Actions workflows, versioning and how to run every kind of test), and the
  end-to-end test notes.

Most main notes have a Ukrainian twin (`*_UA.md`, tagged `_UA`) with the same structure, minus the
Mermaid diagrams.

## Added or rewritten for release 0.2.2.4

- Architecture: [[Feature Flags and Remote Config]] (new), plus corrected paths and line numbers
  across [[Architecture Overview]], [[Presentation Layer (MVI)]], [[Domain Layer]],
  [[Data Layer]], [[Dependency Injection (Hilt)]], [[Async and Reactivity]] and [[Navigation]].
- Reference: [[Build, CI and Releases]] (new), [[Tech Stack and Versions]] (versions re-checked, AGPL
  license, Remote Config, Crashlytics, Kover) and [[E2E Tests]].
- Features: [[Chat Profile and Shared Media]] with [[Chat Profile and Shared Media - Code]],
  [[Chat Profile and Shared Media - Explained Simply]] and
  [[Chat Profile and Shared Media - Basic Example]].

This repository contains documentation only. The application source lives in a separate,
private repository and no app code is copied here.
