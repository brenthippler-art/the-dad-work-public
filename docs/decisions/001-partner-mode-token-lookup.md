# 001: Partner Mode uses a two-step token lookup

## Context

Partner Mode lets a dad share his progress with his partner through a private link. The partner should see only what the dad chooses to share (wins, milestones, reflection prompts), without creating an account.

My first version stored the link's token on the user's document and had the partner's page **query** the `users` collection for a matching token. It failed. For a visitor who isn't signed in, Firestore security rules block the entire query before any document is checked. There was no way to write a rule that allowed "find the user with this token" without also allowing much wider access.

## Options considered

1. **Require the partner to create an account** and link it. Secure, but it adds friction for someone who just wants to see a few updates.
2. **Loosen the rules** so signed-out visitors can query users. This would expose far more data than Partner Mode needs.
3. **Store each token as its own document** that points to the user, turning the lookup from a query into a direct read by ID.

## Decision

Option 3. A separate `partnerTokens` collection maps each token to a user ID.

## How it works

1. When a dad turns on Partner Mode, the app generates a random token and writes `partnerTokens/{token}` containing his user ID.
2. The partner opens `/partner/{token}`. The page reads that one document directly by its ID.
3. With the user ID, the page reads the dad's shared data. Security rules allow it only while his Partner Mode is turned on.

Firestore rules can't safely allow a signed-out *query*, but they can allow a direct *read* of a document whose exact ID you already know. Since the token is the document ID, knowing the token is what grants access.

A simplified version of the token handling:

```js
// Simplified example, not production code
function generateToken() {
  const bytes = crypto.getRandomValues(new Uint8Array(18));
  return Array.from(bytes)
    .map((b) => b.toString(36).padStart(2, "0"))
    .join("")
    .slice(0, 24);
}

async function lookUpPartner(token) {
  const tokenSnap = await getDoc(doc(db, "partnerTokens", token));
  if (!tokenSnap.exists()) return null;

  const { uid } = tokenSnap.data();
  return getDoc(doc(db, "users", uid));
}
```

The token comes from `crypto.getRandomValues`, the browser's secure random number generator, rather than `Math.random`, because the token works like a password. It needs to be impossible to guess.

## Consequences

- **The link is the key.** Anyone who has it can see the shared view, so it's treated like a password.
- **Revoking is simple.** Regenerating the link deletes the old token document and writes a new one, so the old link stops working immediately.
- **The dad controls access.** Turning Partner Mode off blocks all partner reads through the security rules, even if someone still has the link.
- **Partners can't write anything.** The partner view is read-only.