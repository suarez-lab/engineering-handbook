# Cloud Run loses in-memory state on every restart — your status endpoint is lying

**English** · [Español](README.es.md)

`cloud-run` · `correctness` · `2026`

## Context

An admin panel with a "last synchronised" indicator. A background sync writes a JSON
file to object storage every few hours; a module-level variable tracks whether it is
`idle`, `running` or `error`, and when it last succeeded; a `GET /api/sync/<x>/status`
endpoint returns that variable.

Any Cloud Run service that scales to zero has this shape somewhere, usually written
early, when the service was never idle long enough to notice.

## What failed

The panel showed the sync as never having run. Blank timestamp, state `idle`.

The sync had in fact run, on schedule, hours earlier — the output file in object storage
was correct and current, with a fresh `syncedAt` inside it. The data was fine. Only the
report of the data was wrong.

The misleading part is that this looks like the sync failing intermittently. It
reproduces "randomly": fine right after a deploy, blank the next morning, fine again
after someone clicks around. Two engineers can disagree about whether the bug exists,
because whoever hits the endpoint first pays the cold start and whoever hits it second
does not.

## Why

Cloud Run terminates instances when there is no traffic, and replaces them on every
deploy. A `let` at module scope lives exactly as long as the instance does.

So `idle` is overloaded. It means two different things that the code cannot distinguish:

- *this sync is not currently running, and here is when it last finished*, and
- *this process has no idea, because it started thirty seconds ago*

Both serialise to the same JSON. The endpoint is not reporting the state of the sync; it
is reporting the age of the container.

The fix is not to add a database. The durable record of when the sync last succeeded
already exists — it is the output file the sync wrote. It just was not being read.

## The pattern

When in-memory state says `idle`, fall back to the artefact. Read only the first bytes
of the output file: the timestamp lives near the top, and downloading a multi-megabyte
export to learn one field is how a status endpoint becomes the slowest route in the app.

```ts
type SyncStatus = {
  state: "idle" | "running" | "error";
  syncedAt?: string;
};

async function resolveSyncStatus(
  memState: SyncStatus,
  objectName: string,
): Promise<SyncStatus> {
  // 'running' and 'error' are only knowable in memory — trust them.
  if (memState.state !== "idle") return memState;

  try {
    const file = storage.bucket(process.env.OUTPUT_BUCKET!).file(objectName);
    const [exists] = await file.exists();
    if (!exists) return memState;

    // Range read: the first 200 bytes, not the whole export.
    const [head] = await file.download({ start: 0, end: 199 });
    const match = head.toString().match(/"syncedAt"\s*:\s*"([^"]+)"/);
    if (match) return { state: "idle", syncedAt: match[1] };
  } catch {
    // Storage unreachable is not a sync failure — degrade to what we know.
  }
  return memState;
}
```

The handler becomes async, which is the change most likely to be forgotten:

```ts
// Before — blank after any restart
app.get("/api/sync/reports/status", (req, res) => {
  res.json(getReportsSyncStatus());
});

// After
app.get("/api/sync/reports/status", async (req, res) => {
  res.json(await resolveSyncStatus(getReportsSyncStatus(), "reports.json"));
});
```

Three conditions make this work, and all three are easy to break later:

- **The sync must write `syncedAt` near the top of the output.** If a future change moves
  it after a large array, the range read silently stops finding it and you are back to
  blanks — with no error.
- **Fall back to the *output* file, not a separate metadata file.** A metadata file can
  be written when the sync starts, or fail to be written when the sync dies halfway;
  the output file exists only if the sync actually produced something.
- **If the state already lives in a database, skip all of this.** Firestore or Cloud SQL
  already survive restarts. This pattern is for state whose only durable trace is a file.

## How to verify

Force the condition instead of waiting for it. Deploying a new revision replaces the
instance, which is the same event as a scale-to-zero:

```bash
gcloud run services update my-service --region my-region \
  --update-env-vars "CACHE_BUST=$(date +%s)"

curl -s https://my-service.example/api/sync/reports/status
```

If the response has no `syncedAt` while the output file does, you are exposed:

```bash
gcloud storage cat gs://my-bucket/reports.json | head -c 200
```

Two matching timestamps means the fallback works. A timestamp in the file and a blank in
the endpoint is the bug, reproduced on demand in under a minute.

To find every instance of it in a codebase, look for status routes that are not `async`:

```bash
grep -rn "sync.*status" src/ | grep -v "async"
```

## See also

- [A Cloud Run Service that answers before it finishes is not a durable execution](../cloudrun-job-vs-service/README.md) —
  the same instance lifecycle, seen from the writer's side instead of the reader's.
- [Scale a Cloud Run service to zero without touching IAM](../cloudrun-power-control/README.md)
