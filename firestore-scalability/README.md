# A real-time listener is a per-user subscription to everybody else's writes

**English** · [Español](README.es.md)

`firestore` · `cost` · `2026`

## Context

An internal dashboard on Firestore. Several hundred to a few thousand users, all
authenticated, all opening roughly the same screens during the same working hours.
One of those screens is an aggregate view — a heat map, a ranking, a counters panel —
built the obvious way: `onSnapshot` over a collection with a `limit`, so the view
updates itself as new documents arrive.

If your system looks like that, this applies to you. If your concurrency is in the
dozens, it does not, and you should keep the simpler code.

## What failed

Nothing threw. No quota error, no 429, no degraded status page. The read count on the
billing dashboard simply grew at a rate nobody could explain from the traffic numbers,
and the aggregate view got slower as the day went on.

The misleading part: profiling a single session made the view look cheap. One user
opening it read the documents the `limit` allowed and nothing else. The cost was not
in the session; it was in the product of sessions and writes, and no single trace
contains that product.

The shape of the code that caused it:

```javascript
// aggregate view — updates itself, therefore "free"
onSnapshot(
  query(collection(db, 'events'), orderBy('createdAt', 'desc'), limit(200)),
  (snap) => render(aggregate(snap.docs))
);
```

## Why

`onSnapshot` bills in two moments, and the second one is the one people forget.

1. **On attach**, the listener receives the full result set. That is one read per
   document returned, per listener. A view with `limit(200)` open on 1000 clients is
   200000 reads, and it is 200000 reads again every time those clients reload.

2. **On every subsequent write that matches the query**, the changed document is
   delivered to *every* attached listener, and each delivery is a read. A background
   job that appends 60 documents while 1000 listeners are attached bills 60000 reads
   for one batch of 60 writes.

So the cost is not driven by how many documents you have or how many users you have.
It is driven by their product — `documents_changed × listeners_attached` — and that
term only exists at runtime, which is why it never shows up in a code review.

The second thing worth naming: almost no aggregate view actually needs real-time.
A heat map that is four hours stale is a heat map. A price ticker is not. The
`onSnapshot` was chosen because it was the shortest code, not because the product
required live updates.

## The pattern

Three rules, in order of how much they save.

**1. Precompute the aggregate into a single document.** A scheduled job reads the
collection once, on the server, and writes one small document. Every client does one
`getDoc`. Reads go from `listeners × documents` to `listeners × 1`.

```typescript
// server — runs on a schedule, not per request
export async function computeSnapshot() {
  const since = Timestamp.fromMillis(Date.now() - 72 * 60 * 60 * 1000);
  const snap = await db.collection('events')
    .where('createdAt', '>=', since)
    .orderBy('createdAt', 'desc')
    .get();

  const byBucket = new Map<string, number>();
  for (const doc of snap.docs) {
    const d = doc.data();
    if (d.kind !== 'SEARCH') continue;          // extra filters in memory:
    const key = (d.bucket ?? '').trim();        // one index, not a combinatorial set
    if (key) byBucket.set(key, (byBucket.get(key) ?? 0) + 1);
  }

  await db.doc('aggregates/heatmap').set({
    buckets: [...byBucket].map(([bucket, count]) => ({ bucket, count })),
    total: snap.size,
    generatedAt: FieldValue.serverTimestamp(),
    windowHours: 72,
  });
}
```

**2. Split the subscription by view type, not by page.** Real-time and aggregate are
different contracts and must not share a listener. The list view — filtered, paginated,
user-specific — keeps `onSnapshot`. The aggregate view reads the precomputed document
and attaches nothing.

**3. Make staleness explicit, with a fallback.** A precomputed document that silently
stops being regenerated is worse than the expensive version, because it is wrong and
quiet. Check the age on read and fall back rather than render stale data as fresh.

```javascript
const doc = await getDoc(docRef(db, 'aggregates', 'heatmap'));
const ageMs = Date.now() - (doc.data()?.generatedAt?.toMillis() ?? 0);

// job runs every 4h; allow one missed run, then stop trusting it
if (!doc.exists() || ageMs > 5 * 60 * 60 * 1000) return renderFromLiveQuery();
renderFromSnapshot(doc.data());
```

The tolerance window should be strictly larger than the job interval and strictly
smaller than two intervals. Equal to the interval and every slow run trips the
fallback; unbounded and a dead job goes unnoticed for a week.

## How to verify

You do not need load testing. Two checks, both under five minutes.

**Find the exposure.** Any listener with a large `limit` on a shared collection is a
candidate:

```bash
grep -rn "onSnapshot" src/ | grep -v "\.test\."
# then, for each hit: is the query user-specific, or does every user attach the same one?
```

A listener whose query contains no user identifier is attached by everyone, and its
cost multiplies.

**Watch the fan-out directly.** Log the delivered changes per snapshot, open the view
in two tabs, then write one matching document:

```javascript
onSnapshot(q, (snap) => {
  console.log('delivered docs:', snap.docChanges().length, 'from cache:', snap.metadata.fromCache);
});
```

One write, two tabs, two deliveries, two billed reads. Multiply by your real concurrency
and by your real write rate — that product is the number to compare against the
precomputed version, which is one read per client per page load regardless of write rate.

## See also

- The same fan-out reasoning applies to any push-based subscription billed per
  delivery, not only Firestore.
- A scheduled job that regenerates a shared document needs an overlap guard of its
  own: two concurrent runs of the same job will happily compute and write twice. A
  transactional lock document with a TTL is the smallest fix; a non-transactional
  read-then-write check is not a fix at all.
