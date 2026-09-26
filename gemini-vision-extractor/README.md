# A model that answers correctly in a markdown code fence will still take your endpoint down

**English** · [Español](README.es.md)

`gemini` · `outage` · `2026`

## Context

A consumer app in the gaming sector (Spain, 2026) with a photo-capture flow: the user
photographs a paper document with their phone, the image goes to a serverless function as
base64, a multimodal model reads it and returns structured fields, and the app shows a
pre-filled form for the user to confirm.

This entry applies to any "OCR by LLM" path where the model's answer is parsed as JSON and the
parse result decides the HTTP status.

## What failed

Some photos produced a **network error** in the app. Not a validation message, not "we
couldn't read this" — a generic failure, with no form on screen and no way for the user to
enter the data by hand. The flow simply dead-ended.

The misleading part: the model had done its job. The logged raw response was well-formed JSON
with every field correctly extracted. It was just wrapped like this:

````
```json
{"date": "16/07/2026", "amount": "12.50", "error": null}
```
````

`json.loads()` on that string raises. The exception propagated, the function returned 500, and
the frontend's `fetch` handler did what it does with a 500: showed a network error.

Two distinct defects stacked. The model occasionally ignores "no markdown" — that one is
recoverable. The second is not: **the endpoint treated a content-quality outcome as a
transport failure**, and a transport failure has no fallback path in a frontend.

## Why

An instruction-tuned model formatting JSON inside a fence is not malfunctioning. Fenced code
is the overwhelmingly dominant way JSON appears in its training data, so a fence is the high-
probability continuation even when the prompt says not to. You can lower that probability with
prompting; you cannot drive it to zero. Treat the fence as expected output, not as a bug.

The deeper error is a category confusion in the response contract. There are two kinds of
"failure" here and they need opposite handling:

- **Technical failure** — the model is unreachable, the credential is wrong, the function
  crashed. The client should retry or surface an outage. `5xx` is right.
- **Extraction failure** — the photo is blurry, the document is the wrong type, a field is
  genuinely illegible. This is a *normal, expected result* of showing a camera to a member of
  the public. The client should fall back to manual entry. `200` with a flag is right.

Returning `5xx` for the second kind collapses the fallback, because the frontend's error
branch is written for network problems, not for "ask the user to type it".

## The pattern

```python
import json, re

PROMPT = """Extract the fields from this document as strict JSON.
Respond with the JSON object ONLY: no extra text, no markdown, no code fences.

Required shape:
{"date": "DD/MM/YYYY", "amount": "X.XX", "error": null}

Rules:
- If a field cannot be read with certainty, use null for that field.
- If the image is not the expected document, set "error" to a short description.
"""

FENCE = re.compile(r"^```(?:json)?\s*|\s*```$")

def parse_model_output(raw: str) -> dict:
    """Never raises. Returns extracted fields, or a dict carrying an error."""
    try:
        return json.loads(FENCE.sub("", raw.strip()))
    except json.JSONDecodeError as e:
        return {"error": f"unparseable model output: {e}"}

def extract(image_b64: str, mime_type: str) -> dict:
    try:
        raw = model.generate_content([PROMPT, {"inline_data": {
            "mime_type": mime_type, "data": image_b64}}]).text
        logger.info("model raw response: %s", raw)   # keep this: failures are unreproducible
        return parse_model_output(raw)
    except Exception as e:                            # noqa: BLE001 - boundary
        return {"error": f"extraction failed: {e}"}

def handler(request):
    data = extract(request.json["image_b64"], request.json["mime_type"])
    if data.get("error"):
        # Content outcome, not transport failure. 200, with an explicit fallback flag.
        return json.dumps({"ok": False, "use_manual_form": True,
                           "error": data["error"]}), 200
    return json.dumps({"ok": True, "data": data}), 200
```

Three rules that carry beyond this incident:

- **Strip the fence, always.** One regex. Not a prompt-only problem.
- **The HTTP status reflects the transport, not the content.** A model that declines to
  extract has answered you. `200` plus a machine-readable flag keeps the client's fallback
  branch reachable.
- **Always show a confirmation form.** The output is a probabilistic reading of a phone photo.
  Pre-fill it, let the user correct it, never write it straight to your database.

And log the raw response text. When a user reports "it didn't read my document", the image is
usually gone and the run is not reproducible; the raw string is the only evidence you get.

## How to verify

Bypass the model. Feed your parser the shapes the model actually produces:

```python
cases = [
    '{"date": "16/07/2026", "error": null}',                  # clean
    '```json\n{"date": "16/07/2026", "error": null}\n```',    # fenced
    '```\n{"date": "16/07/2026", "error": null}\n```',        # fenced, no language
    'Here is the JSON:\n```json\n{"date": null}\n```',        # preamble
    'I cannot read this image.',                              # refusal
]
for raw in cases:
    out = parse_model_output(raw)      # must never raise
    assert "date" in out or "error" in out
```

The last two are the ones that matter: neither may raise, and both must come back as an
extraction error, never an exception.

Then check the contract end to end with a deliberately unreadable input:

```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST "$ENDPOINT" \
  -H 'content-type: application/json' \
  -d '{"image_b64":"aGVsbG8=","mime_type":"image/jpeg"}'
```

If that prints anything other than `200`, your users have no manual fallback.

## See also

- [Three schemas describe one response, and the strictest one wins silently](../gemini-zod-schema-pipeline/README.md)
  — the next failure in the same pipeline: the model extracts a field correctly and your
  validator throws it away without a word.
