# A `200` from a messaging gateway is not delivery — validate the recipient where you store it

**English** · [Español](README.es.md)

`messaging` · `correctness` · `2026`

## Context

A system that sends notifications through a third-party WhatsApp gateway to phone numbers
that come from configuration or a form — not from an inbound webhook. The numbers are typed
by a person, in whatever format they use locally.

## What failed

**Notifications stopped arriving, and for days nothing anywhere said so.** The application
logged success on every send. The gateway answered `200`. There was no error, no exception
and no failed status. The messages sat in a `pending` state permanently, and the recipient's
owner concluded the product did not work.

The misleading part: the contact lookup for the same number answered `valid`.

## Why

Two endpoints of the same gateway treat the same input differently:

- The **contact lookup** normalizes the number — it adds the missing country code — and
  reports it as valid, returning the normalized identifier.
- The **send** endpoint does **not** normalize. Given a locally formatted number, it creates
  a chat under that malformed identifier, accepts the message, and queues it forever.

So "valid" from the lookup and "accepted" from the send are both true, and neither means
the message can reach a phone. The application's behaviour is identical in the good and
the bad case, which is what makes it silent.

The only reliable signal is the one the lookup *returns*: **the normalized identifier
differs from the input.** The lookup never says "invalid" for these numbers, so waiting for
a status that says so waits forever.

## The pattern

**Validate in the endpoint that saves the number, not in the one that uses it.** By the
time you send, the person who typed it is gone.

```ts
async function saveRecipient(raw: string) {
  const digits = raw.replace(/\D/g, "");
  const res = await gateway.checkContact(digits);   // the lookup normalizes
  const normalized = res.id.replace(/\D/g, "");

  if (normalized !== digits) {
    // the gateway had to fix it: store what it returned, never what was typed
    return store(normalized);
  }
  return store(digits);
}
```

**Treat `pending` as a state with a deadline.** Anything still pending after a few minutes
is a failure and needs its own alert — see
[alert on the fallback path](../../reference-architectures/alert-on-the-fallback-path.md).

```sql
-- messages accepted by the gateway but never confirmed
SELECT id, recipient, created_at
FROM messages
WHERE status = 'pending' AND created_at < now() - interval '10 minutes';
```

## How to verify

Take every recipient number currently stored by the system and run it through the lookup.
Any whose returned identifier differs from the stored value is already failing silently.
Then check the age of the oldest `pending` message — if it is measured in days, you have
been losing messages without knowing.

## See also

- [Alerting on the fallback path](../../reference-architectures/alert-on-the-fallback-path.md)
