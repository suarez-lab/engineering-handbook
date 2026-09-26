# Collection-group reports fail loudly on the index and silently on the limit

**English** · [Español](README.es.md)

`firestore` · `correctness` · `2026`

## Context

A multi-tenant application that stores each tenant's records under its own document
path — `tenants/{tenantId}/deals/{dealId}` — and needs cross-tenant reporting: funnels,
activity counts, velocity. The natural tool is `collectionGroup('deals')` with a
`tenantId` equality filter.

Two different failures wait there. One stops you at the door. The other lets you
through and hands you a wrong number.

## What failed

**The loud one.** A single-field collection-group query threw on the first call:

```ts
db.collectionGroup('deals').where('tenantId', '==', t).get();
// FAILED_PRECONDITION: the query requires an index
```

The misleading part is what happens next. The composite indexes already existed and
were `READY`, the console's suggested-index link did not resolve the problem, and
`firebase deploy --only firestore:indexes` reported success. The index the query wanted
was not a composite index at all.

A variant of the same error is worse, because the index *does* exist: a composite
collection-group index deployed with the range field `DESCENDING`, queried with a bare
`.orderBy(field)` — which defaults to `ASCENDING`. `READY` index, matching fields,
`FAILED_PRECONDITION` anyway.

**The quiet one.** A report that never threw, ran fast, and returned plausible numbers
that were too low. No error, no warning, no truncation notice:

```ts
const snap = await db.collectionGroup('deals')
  .where('tenantId', '==', t)
  .limit(5000)                                  // no date range in the query
  .get();
const rows = snap.docs.filter(d => d.createdAt >= from && d.createdAt <= to);
```

For any tenant under 5000 records this is correct. It stays correct in staging, in
tests, and for the first months of production. Then the largest tenant crosses the cap
and the report starts lying — to them specifically, and to nobody else.

## Why

**The index.** Firestore indexes a field automatically at `COLLECTION` scope. It does
*not* index it automatically at `COLLECTION_GROUP` scope; that is off by default, and
it is a single-field index, so no composite index can satisfy it. Single-field index
configuration lives in a separate API surface — `fieldOverrides` — that the `gcloud`
and `firebase` CLIs do not expose for `queryScope=COLLECTION_GROUP`. Hence: a real
missing index that no CLI can create and that no amount of composite-index deployment
fixes.

Declaring a `fieldOverride` also *replaces* the automatic indexing for that field, so
the `COLLECTION`-scope entries you were getting for free have to be re-declared
alongside the one you actually wanted.

**The ordering direction.** An index direction is an exact contract with `.orderBy()`,
not a hint. If another query created the index `DESCENDING` first, an `ASCENDING`
`.orderBy()` has no index — even though a human reading both would call them "the
same index".

**The silent cap.** `.limit(n)` is applied by the server *before* your in-memory filter
runs, and the documents it returns are an arbitrary subset, not the most recent ones
unless you asked for an order. Filtering after the cap therefore computes a correct
aggregate over the wrong population. Firestore has no reason to warn: the query it was
given succeeded exactly as specified.

## The pattern

**Push every filter that narrows the population into the query.** The `.limit()` stops
being a filter and becomes a circuit breaker that must announce itself when it trips.

```ts
const HARD_CAP = 50000;

let q = db.collectionGroup('deals').where('tenantId', '==', t);
if (status) q = q.where('status', '==', status);
if (from)   q = q.where('createdAt', '>=', Timestamp.fromDate(from));
if (to)     q = q.where('createdAt', '<=', Timestamp.fromDate(to));
if (from || to) q = q.orderBy('createdAt', 'desc');   // direction explicit — never the default
q = q.limit(HARD_CAP);

const snap = await q.get();
if (snap.size >= HARD_CAP) {
  console.warn(`[reports] hard cap hit (${snap.size}); result may be truncated`);
}
```

An optional equality filter that would double your index combinatorics — one index with
it, one without — is the exception. Once the range has bounded the set, filter that one
in memory: it is exact, cheap, and costs no index.

```ts
for (const doc of snap.docs) {
  if (!doc.ref.path.startsWith(`tenants/${t}/`)) continue;  // defensive: collection-group spans tenants
  const d = doc.data();
  if (pipelineId && d.pipelineId !== pipelineId) continue;
  // ...
}
```

That path check is not paranoia. A collection group matches *every* subcollection with
that id, anywhere in the database. If one document is ever written outside the expected
path, or `tenantId` is ever missing, it lands in another tenant's report.

**For the index itself**, pick by query shape:

```bash
# Equality + range/order → composite, COLLECTION_GROUP scope. The CLI can do this.
# Equalities first, range/order last.
gcloud firestore indexes composite create \
  --collection-group=deals --query-scope=COLLECTION_GROUP \
  --field-config=field-path=tenantId,order=ascending \
  --field-config=field-path=createdAt,order=descending

# Single field, no range → fieldOverride via the REST API. The CLI cannot do this.
curl -X PATCH \
  "https://firestore.googleapis.com/v1/projects/PROJECT/databases/(default)/collectionGroups/deals/fields/tenantId?updateMask.fieldPaths=indexConfig" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  -d '{"indexConfig":{"indexes":[
        {"queryScope":"COLLECTION","fields":[{"fieldPath":"tenantId","order":"ASCENDING"}]},
        {"queryScope":"COLLECTION","fields":[{"fieldPath":"tenantId","order":"DESCENDING"}]},
        {"queryScope":"COLLECTION","fields":[{"fieldPath":"tenantId","arrayConfig":"CONTAINS"}]},
        {"queryScope":"COLLECTION_GROUP","fields":[{"fieldPath":"tenantId","order":"ASCENDING"}]}
      ]}}'
```

The three `COLLECTION` entries are the automatic indexing you are replacing. Drop them
and you break every ordinary query on that field.

## How to verify

**Is any report filtering after a cap?** This is the query that finds the silent bug,
and it takes one command:

```bash
grep -rn -A6 "collectionGroup(" src/ | grep -B3 "\.filter("
```

Read every hit and ask one question: *could this collection ever exceed the limit for a
single tenant?* If yes, the report is already wrong for that tenant or will be.

**Is the truncation observable?** Temporarily lower `HARD_CAP` to a number below a known
tenant's document count and run the report. If no warning is logged and no caller sees a
truncation flag, the circuit breaker is decorative — fix that before touching indexes.

**Is the index direction what your code assumes?** Do not infer it from the config file;
read what is actually deployed:

```bash
gcloud firestore indexes composite list --format="table(name,queryScope,fields)"
```

Then make every `.orderBy()` state its direction explicitly, matching that output.

Two operational notes while you are in there. `gcloud` and `firebase` hold separate
credential stores — re-authenticating one does nothing for the other, and you only need
`gcloud` for all of the above. And an index creation whose client-side poll dies may
well have succeeded server-side: list before you retry. Wait for `READY`, never
`CREATING`, before telling anyone the report works.

## See also

- Aggregate views over shared collections have a cost failure mode as well as a
  correctness one — precompute them rather than subscribing every client.
