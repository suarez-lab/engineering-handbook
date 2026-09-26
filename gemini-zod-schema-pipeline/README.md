# Three schemas describe one response, and the strictest one wins silently

**English** · [Español](README.es.md)

`gemini` · `correctness` · `2026`

## Context

A real-estate assistant (Latin America, 2026) that answers queries with structured data: the
model returns a JSON object under constrained decoding, the backend validates it with a Zod
schema, and the frontend renders the validated object.

This entry applies to any pipeline where an LLM response passes through a runtime validator
before reaching the UI — which, if you are doing it properly, is all of them.

## What failed

A new field was added to the assistant's answers: a link to each listing. The work looked
trivial — extend the prompt, extend the model's response schema, ship.

In the UI the field was `undefined`. Every time.

The misleading part: **every component reported success.** The model's raw output contained
the field, correctly populated. No exception was thrown. No validation error was logged.
Nothing appeared in the error tracker. The request returned `200` with a well-formed body that
simply did not have the field in it.

The first hypothesis was the model — a prompt problem, a decoding problem, a caching problem.
It was none of those. The value was generated, and then deleted, by our own code, on purpose,
silently.

## Why

A response schema is not one artefact. It is three, maintained in three different places, and
they have to agree:

1. **The model's `responseSchema`** — constrains what the model is allowed to emit.
2. **The validator schema** (Zod, Pydantic, whatever you use) — decides what survives into the
   application.
3. **The JSON example inside the prompt** — in practice, the strongest signal about output
   shape. A field described in prose but absent from the example is often simply not produced.

Here, artefacts 1 and 3 had the field and artefact 2 did not. And the default behaviour of
most object validators is to **strip** unknown keys, not to reject them. That is a reasonable
default — it is what makes validators safe against injected fields — but it means an
undeclared key produces no error, no warning, and no trace. It is a silent delete.

Each artefact fails differently, which is why the symptom is confusing:

| Missing from | Symptom |
|---|---|
| Prompt example | Model usually omits the field; output looks "incomplete" |
| Model response schema | Model may emit it, or may be blocked from emitting it |
| Validator schema | Model emits it correctly; it vanishes between backend and UI |

A related version of the same class: a field typed `NUMBER` in the model schema while the real
values are alphanumeric identifiers from third-party sources (`ABC1011102482`). The model
returns a string, the validator expects a number, and the *entire* response is rejected — a
loud failure caused by the same root, a type declared in one place that does not match the
data in another.

## The pattern

Declare the three together, in one file, so a field cannot be added to one and forgotten in
the others:

```ts
import { z } from 'zod';
import { SchemaType } from '@google/generative-ai';

// 1. What the model is allowed to emit.
export const MODEL_SCHEMA = {
  type: SchemaType.OBJECT,
  properties: {
    id:          { type: SchemaType.STRING },   // STRING accepts numeric-looking ids too
    title:       { type: SchemaType.STRING },
    listing_url: { type: SchemaType.STRING },
  },
};

// 2. What survives into the application.
export const ResponseSchema = z.object({
  id:          z.union([z.number(), z.string()]),   // external ids are not always numeric
  title:       z.string(),
  listing_url: z.string().optional(),               // optional, not absent
});

// 3. What the prompt shows. Real-looking values, never empty ones.
export const PROMPT_EXAMPLE = JSON.stringify({
  id: 'ABC1011102482',
  title: 'Two-bedroom apartment',
  listing_url: 'https://example.com/listing/123',
}, null, 2);
```

Two rules worth internalising:

- **A new field is three edits, never one.** If your checklist has one line, it is wrong.
- **Prefer `STRING` in the model schema for any identifier that comes from outside your
  system.** External ids are opaque. `STRING` accepts both shapes; `NUMBER` rejects half of
  reality and takes the whole response down with it.

## How to verify

The three artefacts live in the same file, so compare their key sets directly — no model call,
no network, runs as a unit test:

```ts
const modelKeys  = new Set(Object.keys(MODEL_SCHEMA.properties));
const zodKeys    = new Set(Object.keys(ResponseSchema.shape));
const promptKeys = new Set(Object.keys(JSON.parse(PROMPT_EXAMPLE)));

const diff = (a: Set<string>, b: Set<string>) => [...a].filter(k => !b.has(k));

assert.deepEqual(diff(modelKeys, zodKeys),    [], 'model emits fields the validator drops');
assert.deepEqual(diff(zodKeys, modelKeys),    [], 'validator expects fields the model cannot emit');
assert.deepEqual(diff(modelKeys, promptKeys), [], 'schema declares fields the prompt never shows');
```

Add it once and every future field gets checked for free. If you are chasing a missing field
right now, the sixty-second version is to log the raw model output next to the validated
object and diff the keys — the gap between the two is your answer, and it points at the
validator, not the model.

## See also

- [A model that answers correctly in a markdown code fence will still take your endpoint down](../gemini-vision-extractor/README.md)
  — the earlier failure in the same pipeline, where the model's output never reaches the
  validator at all.
