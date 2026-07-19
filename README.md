# Messenger documentation

An Obsidian vault documenting the Messenger Android app: full architecture plus per-feature
deep dives, written by reading the actual source. Every code reference is a `path:line` into
the app repository; notes are cross-linked with wiki-links.

Open this folder as a vault in Obsidian and start at [Home](Home.md).

## Layout

- `01 - Architecture/` - layers, MVI presentation, domain, data, DI, async/reactivity,
  navigation.
- `02 - Features/` - authentication, conversations list, chat, calls, presence and typing,
  notifications, settings, end-to-end encryption, appearance and theming. Each feature has a
  main note (with a Mermaid `## Visualization`), a `- Code` walkthrough of the real source, an
  `- Explained Simply` companion, and a `- Basic Example` showing the inherited base pattern.
- `03 - Reference/` - tech stack and versions.

This repository contains documentation only. The application source lives in a separate,
private repository and no app code is copied here.
