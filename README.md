# Engineering Handbook

**English** · [Español](README.es.md)

Lessons from running production systems. Each entry was written **after** an incident,
a surprise, or a bill — never as a tutorial.

The format is fixed on purpose: *context → what failed → why → the pattern → how to
verify*. If an entry cannot answer "how would I check whether I have this problem right
now", it does not get published.

These come out of AI-assisted business platforms on Google Cloud: messaging pipelines,
LLM cost control, Firestore at awkward scale, Cloud Run's less obvious semantics,
and the organizational policies that quietly shape what you are allowed to deploy.

## Index

| Entry | Lesson |
|---|---|
| [`accepted-is-not-delivered`](accepted-is-not-delivered/README.md)<br><sub>`messaging` · `correctness` · `2026`</sub> | A `200` from a messaging gateway is not delivery — validate the recipient where you store it |
| [`bounded-agent-loops`](bounded-agent-loops/README.md)<br><sub>`agent-loops` · `correctness` · `2026`</sub> | A dedupe guard that only reads history cannot see the duplicate it is creating right now |
| [`chrome-headless-pdf-fixed-footer`](chrome-headless-pdf-fixed-footer/README.md)<br><sub>`chrome-headless` · `correctness` · `2026`</sub> | A negative `bottom` pushes a fixed footer *into* your content, not off the page |
| [`cloudrun-job-vs-service`](cloudrun-job-vs-service/README.md)<br><sub>`cloud-run` · `cloud-scheduler` · `data loss` · `2026`</sub> | A Cloud Run Service that answers before it finishes is not a durable execution |
| [`cloudrun-power-control`](cloudrun-power-control/README.md)<br><sub>`cloud-run` · `cost` · `2026`</sub> | Pause a Cloud Run service by writing one annotation — and store the value you overwrote |
| [`cloudrun-sync-status-persistence`](cloudrun-sync-status-persistence/README.md)<br><sub>`cloud-run` · `correctness` · `2026`</sub> | Cloud Run loses in-memory state on every restart — your status endpoint is lying |
| [`firestore-collectiongroup-indexes`](firestore-collectiongroup-indexes/README.md)<br><sub>`firestore` · `correctness` · `2026`</sub> | Collection-group reports fail loudly on the index and silently on the limit |
| [`firestore-scalability`](firestore-scalability/README.md)<br><sub>`firestore` · `cost` · `2026`</sub> | A real-time listener is a per-user subscription to everybody else's writes |
| [`gemini-vision-extractor`](gemini-vision-extractor/README.md)<br><sub>`gemini` · `outage` · `2026`</sub> | A model that answers correctly in a markdown code fence will still take your endpoint down |
| [`gemini-zod-schema-pipeline`](gemini-zod-schema-pipeline/README.md)<br><sub>`gemini` · `correctness` · `2026`</sub> | Three schemas describe one response, and the strictest one wins silently |
| [`jest-promisify-mock-pattern`](jest-promisify-mock-pattern/README.md)<br><sub>`jest` · `node` · `correctness` · `2026`</sub> | `promisify` captures the function reference at import time — your mock arrives too late |
| [`llm-output-field-normalization`](llm-output-field-normalization/README.md)<br><sub>`llm` · `correctness` · `2026`</sub> | The model is a fourth code path, and your formatter does not run on it |
| [`secret-with-trailing-newline`](secret-with-trailing-newline/README.md)<br><sub>`secret-manager` · `correctness` · `2026`</sub> | A secret stored with a trailing newline passes every check you wrote and fails in production |
| [`secrets-flag-replaces-the-list`](secrets-flag-replaces-the-list/README.md)<br><sub>`cloud-run` · `outage` · `2026`</sub> | `--set-secrets` replaces the whole list — one deploy can silently drop every other secret |
| [`systematic-debugging`](systematic-debugging/README.md)<br><sub>`node` · `outage` · `2026`</sub> | Five deploys failed on an error message that was true, precise, and pointing at the wrong thing |

---

<sub>Generic patterns only. No client code, credentials or identifiers.</sub>
