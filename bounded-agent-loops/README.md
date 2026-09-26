# A dedupe guard that only reads history cannot see the duplicate it is creating right now

**English** · [Español](README.es.md)

`agent-loops` · `correctness` · `2026`

## Context

A scheduled loop in a consumer-lending platform (Latin America, 2026): each cycle it reads
the set of accounts due for contact, runs them through a rules engine, and pushes the
resulting actions onto an outbound messaging queue.

Every action carries an idempotency key — something like `D15_2026-07-16` — and, before
enqueueing, a guard asks the obvious question: *has this key already been sent?* It answers
by querying the persisted history of sent messages.

This is the shape of system the entry applies to: **one pass of a loop can emit more than
one action, and the guard against repetition lives in durable storage.**

## What failed

Some contacts received the same reminder twice, minutes apart, on the same day.

The misleading part: **the guard was working correctly.** It was enabled, it ran on both
actions, and it returned "not previously sent" both times — truthfully. There was no
history record to find. Nothing in the logs looked like a bypass, a race against the
database, or a retry.

The first instinct was to suspect the data source: a duplicated account row, the scheduler
firing twice, two workers picking up the same batch. All three were checked. All three were
clean. The queue simply contained two rows with the same idempotency key, written
milliseconds apart, by the same process, in the same pass.

## Why

Two rules overlapped at a boundary value. One fired on `days_overdue == 15`; another, added
later for recurring follow-up, fired on `days_overdue >= 15`. Both used the same template,
so both derived the same idempotency key. On exactly day 15 — and only on day 15 — both
matched.

The guard compared against **persisted** history. Both actions were born inside the same
iteration of the same loop, before either had been persisted. Neither was in history,
because history is written *after* the loop finishes deciding.

Historical dedupe and intra-cycle dedupe are different problems. The first asks "did a past
run already do this?"; the second asks "did *this* run already decide to do this?" A store
query can only answer the first. The second needs state that lives inside the pass.

A second defect surfaced while fixing the first. The naive repair — keep a set of keys seen
in the current batch and skip repeats — silently dropped legitimate work, because the key is
not unique across entities. Many contacts share `D15_2026-07-16`. Deduping by key alone
discards other people's messages. **The identity used for deduplication must be composite:
`(entity, key, channel)`.**

## The pattern

```js
// Two layers, and a composite identity in both.
async function planCycle(accounts, rules, history) {
  const planned = [];
  const seenThisCycle = new Set();          // layer 2: intra-cycle

  for (const account of accounts) {
    for (const rule of rules) {
      if (!rule.matches(account)) continue;

      const action = rule.buildAction(account);
      // Identity includes the entity. The key alone is shared across entities.
      const identity = `${action.entityId}|${action.idempotencyKey}|${action.channel}`;

      if (await history.alreadySent(identity)) continue;  // layer 1: historical
      if (seenThisCycle.has(identity)) continue;          // layer 2: intra-cycle

      seenThisCycle.add(identity);
      planned.push(action);
    }
  }
  return planned;
}
```

Three rules that generalise beyond this incident:

- **Two layers, always.** A durable check plus an in-pass check. Either one alone leaves a
  hole, and the holes are on opposite sides.
- **Composite identity everywhere.** The same string must be used by the historical check,
  the in-pass set, and the merge that persists the batch. If any of the three uses a
  narrower identity, it is either letting duplicates through or eating valid work.
- **Suspect overlapping rules before suspecting the data.** Duplicates at a boundary value
  (`== N` next to `>= N`) are a predicate-design problem, not an ingestion problem. Check
  the predicates first; it is a five-minute check and it was the answer here.

## How to verify

You do not need the production loop. Run one pass over a synthetic input built to hit the
boundary, and assert on the plan:

```js
const accounts = [{ entityId: 'a1', daysOverdue: 15 }, { entityId: 'a2', daysOverdue: 15 }];
const rules = [
  { matches: a => a.daysOverdue === 15,  buildAction: a => act(a) },
  { matches: a => a.daysOverdue >= 15,   buildAction: a => act(a) },
];
const act = a => ({ entityId: a.entityId, idempotencyKey: 'D15_2026-07-16', channel: 'wa' });

const planned = await planCycle(accounts, rules, { alreadySent: async () => false });

// Exactly one action per entity — not two, and not one in total.
assert.equal(planned.length, 2);
assert.equal(new Set(planned.map(p => p.entityId)).size, 2);
```

Both assertions matter. `length === 2` catches the original duplicate bug; the distinct-entity
assertion catches the over-eager fix that dedupes by key alone. A test that only checks the
first will happily pass on a loop that has started swallowing other people's messages.

Against a live queue, the equivalent check is a grouping query over recently enqueued rows by
`(entity, key, channel)` with `count > 1`. If that returns anything, the second layer is
missing or its identity is too narrow.

## See also

- [Bounded autonomous loop](../../reference-architectures/bounded-autonomous-loop.md) — the
  general contract this incident is one instance of. Invariant 5 (idempotency and mutual
  exclusion) and invariant 2 (observe fresh state) are the two this loop violated. Read the
  architecture for the shape; read this entry for what the violation looks like in a log.
- [Root cause first](../systematic-debugging/README.md) — why "check the data source" was the
  wrong first move here, and what to do instead.
