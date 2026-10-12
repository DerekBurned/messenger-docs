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
[[Voice Messages - Explained Simply]] - [[Home]]
