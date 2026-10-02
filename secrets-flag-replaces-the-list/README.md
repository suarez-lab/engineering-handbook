# `--set-secrets` replaces the whole list — one deploy can silently drop every other secret

**English** · [Español](README.es.md)

`cloud-run` · `outage` · `2026`

## Context

A Cloud Run service that reads several secrets from Secret Manager, exposed to the
container as environment variables. The deploy pipeline lives in a build file, and each
change that needs one more secret adds it to the deploy command.

## What failed

**The deploy succeeded, the service started, and a feature stopped working days later.**
A change needed one new secret, so the build file passed only that secret to
`--set-secrets`. The new revision came up healthy — the container started and answered
its health check. Nothing in the deploy output suggested anything was missing.

The secrets that were *not* in the command were gone from the new revision. Code paths
that read them at request time failed only when somebody used that path. We found it days later, from user-facing
errors, not from a deploy log.

## Why

The `set` family of flags means *replace the entire set*, not *add to it*:

| Flag | Behaviour |
|---|---|
| `--set-secrets` | The listed secrets become the **only** secrets. Everything else is removed. |
| `--update-secrets` | The listed secrets are added or changed. Everything else is kept. |
| `--set-env-vars` | Same trap, for plain environment variables. |
| `--update-env-vars` | Merge. |

Because the container starts fine without them, the platform has nothing to complain
about. A missing environment variable is an application error, and the application only
notices when it needs the value.

## The pattern

**Use the merging flag for incremental changes.**

```bash
gcloud run services update my-service --region=my-region \
  --update-secrets=NEW_SECRET=new-secret-name:latest
```

**If the pipeline must use `--set-secrets`, the build file is the full source of truth** —
every secret the service needs, every time, reviewed like code:

```bash
gcloud run services update my-service --region=my-region \
  --set-secrets=SECRET_A=secret-a:latest,SECRET_B=secret-b:latest,SECRET_C=secret-c:latest
```

**Fail the pipeline when a name disappears.** Compare the names before and after:

```bash
names() {
  gcloud run services describe my-service --region=my-region --format=json \
    | jq -r '.spec.template.spec.containers[0].env[].name' | sort
}

names > /tmp/before.txt
# ... deploy ...
names > /tmp/after.txt
comm -23 /tmp/before.txt /tmp/after.txt   # names that existed and no longer do
```

## How to verify

Run the `names` function above against any service you deploy today, and diff it against
the list the application actually reads. Any name the code uses that is not in the output
is a latent failure. Then search your build files for `--set-secrets` and `--set-env-vars`
and check that each one lists everything the service needs.

## See also

- [`cloudrun-job-vs-service`](../cloudrun-job-vs-service/README.md) — another case where a deploy can look healthy and be wrong.
