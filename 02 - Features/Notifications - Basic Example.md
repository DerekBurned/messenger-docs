# Notifications - Basic Example

The inherited pattern behind [[Notifications]], in its plain textbook form - not the real
`MyFirebaseMessagingService`/`ChatNotifier`, but the platform APIs they specialise. Then read
[[Notifications - Code]] for the real source.

(Teaching comments only; the real source has none.)

---

## What is inherited

Push is FCM's own `FirebaseMessagingService` (override `onMessageReceived` / `onNewToken`) plus
the platform's **`MessagingStyle`** notification with a **`RemoteInput`** action for inline reply.
These are the standard Android chat-notification building blocks; the app adds offline persistence
and bubbles on top.

## The basic example (canonical FCM receiver + inline-reply notification)

```kotlin
// FCM constructs this service for you; there is no long-lived scope, so work is short-lived.
class MyMessagingService : FirebaseMessagingService() {
    override fun onNewToken(token: String) { /* upload token so senders can target this device */ }

    override fun onMessageReceived(message: RemoteMessage) {
        val data = message.data
        if (data["type"] == "message") notify(data["sender"].orEmpty(), data["body"].orEmpty())
    }
}

// A conversation-style notification with a reply box, tied to a channel.
fun Context.notify(sender: String, body: String) {
    val remoteInput = RemoteInput.Builder("key_reply").setLabel("Reply").build()
    val replyAction = NotificationCompat.Action.Builder(0, "Reply", replyPendingIntent)
        .addRemoteInput(remoteInput)
        .setSemanticAction(NotificationCompat.Action.SEMANTIC_ACTION_REPLY)
        .build()

    val style = NotificationCompat.MessagingStyle(Person.Builder().setName("You").build())
        .addMessage(body, System.currentTimeMillis(), Person.Builder().setName(sender).build())

    val n = NotificationCompat.Builder(this, "chat_messages")
        .setSmallIcon(R.drawable.ic_msg)
        .setStyle(style)
        .addAction(replyAction)
        .build()
    NotificationManagerCompat.from(this).notify(sender.hashCode(), n)
}
```

## How the real code specialises it

- The receiver is `MyFirebaseMessagingService` (`data/remote/firebase/MyFirebaseMessagingService.kt:32`);
  because the OS constructs it, it reaches the Hilt graph through a manual `@EntryPoint`
  (`FcmEntryPoint`, `:25`) - see [[Dependency Injection (Hilt)]].
- `onMessageReceived` (`:53`) dispatches on `data["type"]`: `message` decrypts (see
  [[End-to-End Encryption]]) and **persists** the message to ObjectBox for offline delivery
  before notifying, with a voice note's body read as "Voice message"; `call` starts the foreground-service call path (see
  [[Calls (Voice and Video)]]).
- `ChatNotifier` (`presentation/notification/ChatNotifier.kt:30`) adds per-conversation history,
  avatar `Person` icons, dynamic shortcuts + `LocusId`, an `InboxStyle` group summary, and
  conversation **bubbles** (`bubbleMetadata`, `:177`) on top of the base `MessagingStyle`.

## Related

[[Notifications]] - [[Notifications - Code]] - [[Notifications - Explained Simply]] -
[[Reading the Code - Simple Patterns]] - [[Dependency Injection (Hilt)]] - [[Home]]
