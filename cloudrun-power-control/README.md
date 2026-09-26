# Pause a Cloud Run service by writing one annotation — and store the value you overwrote

**English** · [Español](README.es.md)

`cloud-run` · `cost` · `2026`

## Context

A portfolio of Cloud Run services across several GCP projects, most of which only need
to be reachable during business hours. We wanted two things from an internal admin panel:
a manual pause/resume button, and a scheduled nightly shutdown.

The constraint that shaped everything: an organization policy blocks `gcloud run deploy`,
because that command issues a `setIamPolicy` call as part of its normal flow. Anything
that redeploys the service to change its scaling is off the table. So is shipping the
full GCP SDK into a Next.js app whose bundle we would rather keep small.

## What failed

**The first attempt scaled services down and could not scale them back up correctly.**
Pausing was easy: set max instances to `0`. Resuming meant choosing a number, and the
code chose a default. Services that had been deliberately capped at 2 came back at 10;
one that had been capped at 1 came back able to fan out. Nothing errored. The bill was
the notification.

**The second attempt, using the Knative API, returned 404 on a service that visibly
existed.**

```
Requested entity was not found
```

Not a permissions error, not a typo in the service name — the URL was wrong in two ways
at once, and each one alone produces the same 404.

**The third attempt worked from a test script and failed from Cloud Scheduler**, with
Scheduler refusing the job configuration rather than the request failing at runtime.

## Why

**The resume bug** is a missing write. Pausing is destructive: `maxScale = 0` overwrites
the only record of what the value used to be. If you do not persist the original before
overwriting it, the information is gone, and "resume" can only ever mean "guess".

**The 404** is the shape of the Knative-flavoured Cloud Run API, which differs from the
`v1` shape in two respects people carry over by habit:

- The **region goes in the hostname**, not in the path: `my-region-run.googleapis.com`.
  A regionless host resolves, authenticates, and reports the service as not found.
- The **namespace is the project number**, not the project ID. A project ID in that slot
  also authenticates and also reports the service as not found.

Two independent mistakes with one identical symptom is why this costs an afternoon.

**The Scheduler rejection** is a supported-methods gap: Cloud Scheduler HTTP targets
support `DELETE`, `GET`, `HEAD`, `POST` and `PUT` — not `PATCH`. Any design that reaches
for a partial update from a cron has to be re-shaped around a `POST` to your own endpoint,
which then performs the `PUT` upstream.

## The pattern

**Persist before you overwrite.** The state machine is symmetric and the write order
matters — the saved value must be durable *before* the destructive call:

```
pause:   GET current maxScale → WRITE power_state/<id> → PUT maxScale = 0
resume:  READ power_state/<id> → PUT maxScale = original → DELETE power_state/<id>
```

**Change one annotation, nothing else.** The `PUT` touches only
`spec.template.metadata.annotations`, so it never calls `setIamPolicy` and stays inside
the organization policy:

```ts
const host = `https://${region}-run.googleapis.com`;
const url =
  `${host}/apis/serving.knative.dev/v1/namespaces/${projectNumber}/services/${serviceName}`;
//            region in the host  ^^^^^^          project NUMBER ^^^^^^^^^^^^^

const svc = await (await fetch(url, { headers: auth })).json();

svc.spec.template.metadata.annotations["autoscaling.knative.dev/maxScale"] = "0";
delete svc.status;          // read-only field; leaving it in makes the PUT fail

await fetch(url, { method: "PUT", headers: auth, body: JSON.stringify(svc) });
```

**Get the token from the metadata server**, not from a key file — which also happens to
be the only option when an organization policy blocks service-account key creation:

```ts
const r = await fetch(
  "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token",
  { headers: { "Metadata-Flavor": "Google" }, cache: "no-store" },
);
const { access_token } = await r.json();
```

**Give the runtime the narrowest role that works.** Writing the service object needs
`roles/run.developer`; only cross-project administration needs `roles/run.admin`:

```bash
# the runtime service account of the service you are power-cycling
SA="$(gcloud run services describe my-service --region=my-region \
  --format='value(spec.template.spec.serviceAccountName)')"
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:${SA}" \
  --role="roles/run.developer" --condition=None
```

**For scheduled shutdown, two crons hitting your own endpoint.** Express the schedule in
UTC and convert once, in a comment, rather than trusting a timezone string you will not
re-read:

```bash
# Local business hours are UTC−4: down at 22:00 local, up at 07:00 local.
gcloud scheduler jobs create http my-service-scale-down \
  --schedule="0 2 * * *" --time-zone="UTC" \
  --uri="https://my-service.example/internal/scaling?mode=down" \
  --http-method=POST --message-body='{}'

gcloud scheduler jobs create http my-service-scale-up \
  --schedule="0 11 * * *" --time-zone="UTC" \
  --uri="https://my-service.example/internal/scaling?mode=up" \
  --http-method=POST --message-body='{}'
```

For the scheduled variant the annotations to flip are `minScale` (`"0"` down, `"1"` up)
and `run.googleapis.com/cpu-throttling` (`"true"` down, `"false"` up) — that pair is what
turns off a warm instance rather than capping a cold one.

**Three UI rules**, because this control is destructive and one click wide:

1. Confirm explicitly, naming the service and the GCP project in the dialog.
2. One action at a time — disable every button while any request is in flight.
3. Model `unknown` as a first-class state. If reading the current scale failed, the
   button is disabled. Never act on a value you could not read.

## How to verify

**Check your URL before blaming permissions.** If this returns the service, the shape is
right; if it 404s, one of the two substitutions is wrong:

```bash
PROJECT_NUMBER=$(gcloud projects describe my-project --format='value(projectNumber)')
TOKEN=$(gcloud auth print-access-token)

curl -s -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer $TOKEN" \
  "https://my-region-run.googleapis.com/apis/serving.knative.dev/v1/namespaces/${PROJECT_NUMBER}/services/my-service"
```

`200` means the host and namespace are both correct. `404` with a valid token means they
are not — re-check the region prefix on the host first, it is the more common of the two.

**Prove resume restores the original, not a default.** Read the value, pause, resume,
read again:

```bash
scale() { gcloud run services describe my-service --region my-region \
  --format="value(spec.template.metadata.annotations['autoscaling.knative.dev/maxScale'])"; }

scale                       # e.g. 2
# ...pause via the panel...
scale                       # 0
# ...resume via the panel...
scale                       # must be 2 again, not a default
```

If the third reading differs from the first, the persist-before-overwrite step is missing
or is racing the `PUT`.

**Confirm you have not tripped the IAM policy.** A pause that fails with
`PERMISSION_DENIED` on `setIamPolicy` rather than on `run.services.update` means
something in the path is still redeploying rather than patching the annotation.

## See also

- [A Cloud Run Service that answers before it finishes is not a durable execution](../cloudrun-job-vs-service/README.md)
- [Cloud Run loses in-memory state on every restart](../cloudrun-sync-status-persistence/README.md) —
  relevant if your panel caches the paused/running state in memory.
