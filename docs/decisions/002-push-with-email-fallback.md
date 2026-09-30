# 002: Push notifications with an email fallback for iOS

## Context

The Dad Work depends on a daily habit, and streaks only work if dads remember to come back. A daily reminder is core to the product, not an extra.

Web push has uneven support:

- **Android** handles web push well through Firebase Cloud Messaging.
- **iPhones** only support web push on iOS 16.4 or later, and only when the app is installed to the home screen from Safari. A dad using the site in the browser can't receive push at all.
- **Push tokens go stale** when users clear browser data, uninstall, or reinstall, and sending to a stale token fails silently from the user's point of view.

## Options considered

1. **Push only.** Simple, but a large share of iPhone users would never get reminders.
2. **Email only.** Reaches everyone, but it's easier to ignore and feels less like an app.
3. **Wrap the app as a native iOS app.** Solves push, but adds App Store review, fees, and a second release process for a solo developer.
4. **Push when possible, email when not,** decided per user every time a reminder is sent.

## Decision

Option 4, run from a single scheduled Cloud Function.

## How it works

The function runs every hour and, for each user whose reminder time has arrived:

1. **Skips anyone who already finished today's content,** checked against the date in *their* timezone rather than the server's.
2. **Sends a push notification** through Firebase Cloud Messaging if the user has a push token saved.
3. **Sends an email reminder instead** if there's no push token.
4. **Cleans up stale tokens.** If FCM reports that a token is invalid or no longer registered, the function deletes it from the user's profile, so that user automatically gets email from then on.

A simplified version of the per-user logic:

```js
// Simplified example, not production code
async function remindUser(user) {
  if (completedToday(user.lastActiveAt, user.timezone)) return "skipped";

  if (!user.pushToken) {
    await sendReminderEmail(user.email, user.currentDay);
    return "emailed";
  }

  try {
    await messaging.send({ token: user.pushToken, notification: buildReminder(user) });
    return "pushed";
  } catch (err) {
    if (isStaleTokenError(err)) {
      await clearPushToken(user.id); // next reminder falls back to email
    }
    return "failed";
  }
}
```

Each user is processed independently with `Promise.allSettled`, so one failed send doesn't stop the rest of the batch.

## Consequences

- **Every user gets a reminder** regardless of device or whether they installed the app.
- **The fallback heals itself.** A broken push token turns into email reminders without any manual cleanup.
- **Nobody gets reminded for a day they already finished,** which keeps reminders from feeling like nagging.
- **Tradeoffs:** a function running every hour costs a little more than one daily run, and email reminders are easier to miss than push. To close that gap, the app encourages iPhone users to install it to the home screen so they can switch to push.