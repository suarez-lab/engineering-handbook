# A Cloud Run Service that answers before it finishes is not a durable execution

**English** · [Español](README.es.md)

`cloud-run` · `cloud-scheduler` · `data loss` · `2026`

## Context

Scheduled batch work on Google Cloud Run: nightly exports, scrapers, ETL, report
generation. The code is a script — it starts, does the work, writes an output file,
exits. The question is which Cloud Run primitive runs it, and the answer is not
obvious because both a **Service** and a **Job** will appear to work on the first try.

This applies to anything triggered by Cloud Scheduler that must *complete*, as opposed
to anything triggered by a user that must *respond*.

## What failed

Three symptoms, in the order we hit them.

**1. The script deployed as a Service never started.**

```
container failed to start and listen on the port defined by the PORT environment variable
```

Misleading, because the script ran fine: it did the work and called `exit(0)`. Cloud Run
Services treat a process that stops listening as a failed health check. A clean exit is
the failure.

**2. Rewritten as an HTTP Service that returned `202 Accepted` and continued in a
background promise, the work silently truncated.**

No error. No log line. The output file in object storage was written, but incomplete —
partial on some runs, current on others. It looked like a flaky upstream API.

**3. Once it did work, Cloud Scheduler reported the job as failed anyway.**

The Scheduler execution history showed failures. The service logs showed `200 OK` on
every single invocation over a week. Both were telling the truth.

## Why

**Symptom 1** is a primitive mismatch. A Service is a server: its contract is *listen on
`PORT` and keep listening*. A Job is a process: its contract is *run to completion and
exit*. Deploying batch code as a Service inverts the success condition.

**Symptom 2** is CPU throttling. By default, a Cloud Run Service's CPU is throttled once
the response has been sent. A promise left running after `res.status(202).send()` does
not get a guaranteed CPU slice — it may be paused mid-read or mid-upload and never
resumed, because from the platform's point of view the request is over. The work is not
durable; it is a best-effort continuation. That is fine for fire-and-forget telemetry
and wrong for anything whose absence you would notice.

**Symptom 3** is two independent timeouts. Cloud Scheduler's `attempt-deadline` and the
Service's `timeoutSeconds` are not related. Scheduler cuts the HTTP connection and marks
its own attempt failed at *its* deadline, regardless of the service continuing to run
and finishing successfully under its own, longer timeout. The result is a false alarm in
the observability layer: the work is fine, the dashboard is not.

We have now seen symptom 3 twice, in unrelated systems. Both times the real runtime had
grown past a deadline that was set when the workload was smaller — once because a
third-party geocoding API required per-request throttling, once because an upstream read
timeout plus a cold start pushed a run just over the line. Neither is caught by any lint
or deploy gate.

A detail that costs an hour when diagnosing symptom 3: 2nd-gen Cloud Functions run on
Cloud Run. Query their logs with `resource.type="cloud_run_revision"`. The older
`resource.type="cloud_function"` filter is 1st-gen only and returns an empty result set
**without an error**, which reads as "no logs, so it never ran".

## The pattern

Choose the primitive from the shape of the work, not from what you already have deployed:

| | Cloud Run **Service** | Cloud Run **Job** |
|---|---|---|
| Workload | HTTP server / API | Batch script |
| Trigger | HTTP request | Manual, Scheduler, Pub/Sub |
| Lifetime | Indefinite, scales to zero | Bounded (max 24h) |
| Must listen on `PORT` | Yes | No |
| Success signal | Keeps serving | Exits `0` |

Deploy batch work as a Job:

```bash
gcloud run jobs create my-job \
  --image "$IMAGE_URL" \
  --project my-project --region my-region \
  --task-timeout 900 --max-retries 1 --memory 2Gi
```

If you genuinely need a Service — because the batch path shares a codebase with an API —
then make the handler `await` the work and respond at the end. An explicit
`?wait=true` query parameter makes the contract visible in the Scheduler target itself:

```ts
app.post("/tasks/export", async (req, res) => {
  const durable = req.query.wait === "true";
  if (!durable) {
    // Only safe if a queue or Job has already taken ownership of the work.
    res.status(202).json({ accepted: true });
    return;
  }
  const result = await runExport();   // CPU is guaranteed while the request is open
  res.status(200).json(result);       // respond only once the work is on disk
});
```

Then size the deadlines so the outer one is the larger:

```
Cloud Scheduler attempt-deadline  >  Service timeoutSeconds  >  worst-case runtime
```

Three rules that fall out of this, each of which cost us a production incident:

- **A Job does not inherit anything from the Service next to it.** If the Service has a
  `--vpc-connector`, the Job needs its own, explicitly. Without it you get `ETIMEDOUT`
  connecting to a private database while the Service, in the same project, works fine.
- **`describe && update || create` must repeat every flag in the `update` branch**, not
  just `--image`. Secrets, VPC connector, memory — anything omitted from `update` is
  dropped from the Job on the next deploy.
- **A Node.js Job must close its connection pools.** The work finishes, the logs say
  "done", and the execution is still reported as failed with
  `exit code: 0 and message: The configured timeout was reached`, because an open pool
  keeps the event loop alive.

```ts
async function main() {
  await doWork();
  await pool.end();                 // without this the process never exits
}
main().catch(async (err) => {
  await pool.end().catch(() => {});
  process.exit(1);
});
```

## How to verify

**Are your Scheduler deadlines actually above your runtimes?** For each scheduled job,
compare the configured deadline against the p99 of real executions:

```bash
gcloud scheduler jobs describe my-scheduler-job \
  --location my-region --format="value(attemptDeadline)"

gcloud run services describe my-service \
  --region my-region --format="value(spec.template.spec.timeoutSeconds)"
```

Then read the service's own logs for the same window — not the Scheduler status:

```bash
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="my-service"' \
  --freshness=7d --format="value(httpRequest.status, httpRequest.latency)"
```

If the Scheduler says failed and this says `200` with a latency above the attempt
deadline, you have symptom 3 and nothing is actually broken except the alerting.

**Is any handler doing durable work after responding?** This grep finds the shape:

```bash
grep -rn "res.status(202)\|res.sendStatus(202)" src/
```

For each hit, check whether anything after that line writes to a database or object
store. If it does, and no queue or Job took ownership first, it is not durable.

**Does your Job exit?** Run it and watch for a completion that is reported as a timeout:

```bash
gcloud run jobs execute my-job --region my-region --wait
```

A log line saying the work completed, followed by a timeout failure, is an open handle —
almost always a connection pool.

## See also

- [Cloud Run loses in-memory state on every restart](../cloudrun-sync-status-persistence/README.md) —
  the same "scales to zero" property that throttles your background promise also wipes
  your status variables.
- [Scale a Cloud Run service to zero without touching IAM](../cloudrun-power-control/README.md)
