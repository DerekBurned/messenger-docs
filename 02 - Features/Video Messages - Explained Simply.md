# Video Messages - Explained Simply

The plain-language version of [[Video Messages]]. How a round video note gets from your front
camera to the other person's chat.

(The comments in the code are teaching comments for this doc only. The real source has none.)

---

## The big idea in one sentence

The mic button doubles as a camera button: tap it to switch, hold it to film yourself in a
circle, let go to send - and if anything interrupts you, the clip is kept as a draft instead of
being lost.

Analogy: a photo booth with a curtain. Tapping the button flips the sign from "voice" to "video".
Holding it starts the camera. Letting go hands the film to the post office. If someone pulls the
curtain open mid-take (a call, switching apps, leaving the chat), the film is not thrown away; it
is put in your drawer for later.

---

## Step by step

Alice sends Bob a 10-second video note.

1. She taps the mic once. Because she let go within 200 ms, the app treats it as a tap, not a
   recording: the icon turns into a camera and a hint says "Hold to record a video message".

2. She holds the button. After 200 ms the app checks camera and microphone permission and
   starts the recorder (`VideoNoteRecorder.start`, `data/media/VideoNoteRecorder.kt:154`).
   The screen dims and a circle with her front-camera preview appears; a red ring around it fills
   up as the 2 minutes run out. The composer shows a red dot, a timer and "Slide to cancel".

3. She can slide left to throw the clip away, or slide up along the lock rail to lock it and lift
   her finger. Locked, she gets a trash button, a stop button and a send button.

4. She lets go. The camera finishes writing the file - this takes a moment, so the app waits for
   CameraX to say "done" (`Finalize`). Then the clip goes into the same safe outbox as a voice
   note and is uploaded, encrypted when the chat is encrypted.

5. If Alice had left the chat or switched apps before the file was finished, the clip would
   become a draft in that chat instead of vanishing. The draft shows a small round thumbnail
   with Send and Discard.

6. Bob sees a round bubble with the first frame of the video, its length and the time. He taps
   it: the circle grows, the video plays inside it, and a white ring around the edge shows how
   far it has played.

7. Bob drags his finger along the ring to jump around in the clip. A tap in the middle pauses.
   A long press still opens the message menu, and a swipe from the middle still starts a reply.

8. When the clip ends it goes back to its thumbnail and is marked as watched. Unlike voice
   notes, it does not start the next message automatically.

---

## The code, with plain-English comments

What happens to the film when recording ends (`videoFinalizeOutcome`,
`data/media/VideoNoteRecorder.kt:87`):

```kotlin
when {
    // Alice pressed the trash / slid left: always throw it away.
    requested == VideoEnd.DISCARD -> VideoEnd.DISCARD
    // Empty file, shorter than 1 second, or the camera reported a real error: nothing worth keeping.
    fileBytes <= 0L || durationMs < MIN_VIDEO_DURATION_MS || error !in USABLE_FINALIZE_ERRORS -> VideoEnd.DISCARD
    // Otherwise do what was asked; if nobody asked (the 2-minute limit hit by itself), keep it as a draft.
    else -> requested ?: VideoEnd.DRAFT
}
```

Turning a finger position on the ring into "how far through the clip"
(`ringFraction`, `presentation/screens/chat/VideoNoteBubble.kt:87`):

```kotlin
// Angle of the finger around the circle's centre, measured from 12 o'clock, clockwise: 0 .. 1.
val turns = atan2(x - cx, -(y - cy)) / (2f * PI.toFloat())
return (turns + 1f) % 1f
```

Fitting a wide frame into a round window (`centerCropScale`, `VideoNoteBubble.kt:95`):

```kotlin
// Scale so the SMALLER side fills the window, then let the longer side overflow and be cropped.
val scale = max(viewW.toFloat() / videoW, viewH.toFloat() / videoH)
```

---

## A worked example

- Alice holds the mic for 0.6 seconds and lets go with her finger still. The film is shorter than 1
  second, so `videoFinalizeOutcome` says "throw away" and nothing is sent.
- She tries again and films 10 seconds, then slides up to lock, then taps send. The request is
  `SEND`, the file is usable, so the clip goes into the outbox and is uploaded.
- Halfway through a third clip a phone call comes in. The request becomes `DRAFT`; Alice finds
  the clip waiting in the chat with Send and Discard buttons after the call.
- If the team switches the `video_notes` flag off in Remote Config, the camera mode stops being
  offered and the app refuses to start the camera even from an old screen - see
  [[Feature Flags and Remote Config]].

---

## Why, in one line each

- Why wait for `Finalize`? The file is not complete until CameraX says so.
- Why throw away clips under a second? An accidental touch should not send an empty circle.
- Why keep a clip as a draft when the 2-minute limit hits by itself? The person never chose to
  stop, so losing the take would be the worse surprise.
- Why crop at display time? A normal 4:3 recording works on every phone; the circle is just a mask.
- Why a ring you drag? The circle has no room for a seek bar, so its edge is the seek bar.

---

## Why the camera is handled so carefully

- The camera is shared by the whole app (the photo/video capture screen uses it too). Cancelling
  its start-up the wrong way would break it until the app restarts, so the start-up is never
  cancelled - if Alice lets go too early, the app simply ignores the result.
- The file has to fit the server's 40 MB limit, so the recorder asks for standard definition at
  1.2 Mbps and stops at 38 MB at the latest.
- The video is filmed in a normal 4:3 frame and cut into a circle only when it is shown, which
  works the same on every phone.

## Related

[[Video Messages]] - [[Video Messages - Code]] - [[Video Messages - Basic Example]] -
[[Voice Messages - Explained Simply]] - [[Feature Flags and Remote Config]] - [[Home]]
