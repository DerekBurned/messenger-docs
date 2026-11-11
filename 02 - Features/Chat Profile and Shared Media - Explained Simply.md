# Chat Profile and Shared Media - Explained Simply

The plain-language version of [[Chat Profile and Shared Media]]. How the pictures, videos and voice
messages you shared with someone show up on their profile page. `path:line` refs are relative to
`app/src/main/java/com/example/messenger/`.

(Teaching comments only; the real source has none.)

---

## The big idea in one sentence

The profile page keeps a photo album for one chat: it shows what your phone already has right
away, quietly asks the server for older things 60 at a time as you scroll, and mixes both into
four tabs - Photos, Videos, Voice and Files.

Analogy: a shoebox of photos on your desk (what is already on the phone) and an archive in the
basement (the server). You flip through the shoebox first. When you reach the bottom you send
someone down to fetch the next batch of 60 from the archive, and the new ones are slotted into the
same sorted pile. When you leave, the archive batches are put back - the shoebox stays.

---

## Step by step: opening Alice's profile and looking at photos

1. You tap Alice's name at the top of the chat (or swipe left on the chat). The app opens her
   profile and tells it which conversation you came from
   (`presentation/navigation/graph/MainDisplay.kt:511`).

2. The profile screen's "brain" (`ChatUserProfileViewModel`,
   `presentation/screens/chatuserprofile/ChatUserProfileViewModel.kt:38`) wakes up. First it checks
   the on/off switch for this section (`partner_profile_shared_media`). If the switch is off it does
   nothing more and the tabs never appear.

3. It works out which conversation to look in: the one it was told about, or else the first
   one-to-one chat it can find with Alice (`:140`). No chat at all means "nothing to show", and the
   tab says so.

4. It starts watching the messages already saved on the phone for that chat (`:130`). Anything
   there appears at once, and anything new that arrives while the page is open appears by itself.

5. At the same time it asks the server for the newest page of 60 photo-or-voice messages
   (`loadMoreMedia`, `:147`). Those are kept only in memory.

6. A small sorter (`sharedMediaOf`, `presentation/screens/chatuserprofile/SharedMedia.kt:21`) takes
   the combined pile, throws away deleted messages, puts the newest first and deals them into three
   stacks: photos, videos, voice.

7. The Photos tab shows the stack as a grid, four squares per row
   (`presentation/components/profile/ProfileSharedMedia.kt:110`). A square that is not on the phone
   yet shows a blurry preview and a download arrow.

8. You tap a downloaded photo and it opens full screen; you can swipe to the next downloaded photo
   or video. You tap a Voice row and it plays, using the same player as the chat.

9. You scroll to the bottom. A hidden marker at the end of the list becomes visible and says
   "fetch the next 60" (`ProfileSharedMedia.kt:102`). When the batch lands, the marker checks again;
   when a batch comes back shorter than 60 the album knows it has reached the beginning of the chat
   and stops asking.

---

## Picture: how the album is filled

```mermaid
flowchart LR
    PHONE[(Saved on the phone)] --> MIX[Mix: the phone copy wins]
    SERVER[Server, 60 at a time] --> MEM[Kept in memory only] --> MIX
    MIX --> SORT[Sort: photos, videos, voice]
    SORT --> TABS[Four tabs]
    TABS -->|scroll to the end| SERVER
```

---

## The code, with plain-English comments

Deciding what goes in which tab (`SharedMedia.kt:21`):

```kotlin
// Hide deleted messages and put the newest first.
val newestFirst = messages.filterNot { it.deleted }.sortedByDescending { it.timestamp }
// photos = pictures, videos = videos, voice = voice messages only (not video notes)
```

Mixing what the phone has with what the server sent (`ChatUserProfileViewModel.kt:130`):

```kotlin
// "remote + local" - if the same message is in both, the phone's copy replaces the server's.
sharedMediaOf((remote + local.associateBy { it.id }).values)
```

Deciding when to stop asking (`ChatUserProfileViewModel.kt:160`):

```kotlin
// A page shorter than 60 messages means there is nothing older left.
mediaEndReached = page.size < MEDIA_PAGE_SIZE
```

---

## A worked example

- You and Alice have sent 130 photos over a year. You open her profile.
- The phone already has the last 20 saved, so the grid appears instantly. The server adds its
  newest 60 and the grid grows to about 60 (the 20 overlap and are counted once).
- You scroll down: the marker appears, 60 more arrive, then the last 10. That batch is shorter than
  60, so the album stops asking.
- You tap a photo that is only a blurry preview: it downloads, and if the chat is end-to-end
  encrypted it is unlocked on the way in. Now the full picture opens.
- You leave and come back: the server batches are gone from memory and are fetched again.

---

## Why, in one line each

- Why show the phone's copy first? It is instant and works offline.
- Why only 60 at a time? A chat can hold thousands of messages; fetching all of them to look at
  a few would be slow and costly.
- Why keep the server batches in memory only? So browsing an old album does not fill the phone's
  database with things you may never open again.
- Why does the Files tab always say "No files yet"? The app cannot send files, so there is
  nothing to list.
- Why can you only swipe through photos you have downloaded? The full-screen viewer reads the
  file saved on the phone; a photo not yet downloaded has no file to show.
- Why is there an on/off switch? So the team can hide the whole section from every phone without
  shipping a new app. See [[Feature Flags and Remote Config]].

---

## Related

[[Chat Profile and Shared Media]] - [[Chat Profile and Shared Media - Code]] -
[[Chat Profile and Shared Media - Basic Example]] - [[Chat and Messaging - Explained Simply]] -
[[Voice Messages - Explained Simply]] - [[End-to-End Encryption - Explained Simply]] -
[[Reading the Code - Simple Patterns]] - [[Home]]
