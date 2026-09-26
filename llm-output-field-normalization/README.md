# The model is a fourth code path, and your formatter does not run on it

**English** · [Español](README.es.md)

`llm` · `correctness` · `2026`

## Context

A service that answers user questions over a catalogue. Records are fetched from a
data source, injected into the prompt as context, and the model returns structured
JSON — a list of recommendations, each with an id and a few display fields.

Some of those display fields are not the model's to invent. A title, a formatted price,
a canonical label: these are derived from the record by a deterministic function that
already exists in the codebase and is already used everywhere.

## What failed

Titles in the response came back as raw source strings — `"north-district·apartment·sale"`
— instead of the human-readable form the formatter produces, `"Apartment in North District"`.

What made it confusing: the formatter was demonstrably correct, unit-tested, and called
from every place that builds a title. We audited three call paths and all three were
right. The bug survived a deploy that "fixed" it, because the fix was applied to code
that was never the source of the string.

The raw form was the delimiter-joined header that the upstream data source stores as an
index key. It appeared in the prompt context verbatim. And the response schema's example
value, written to illustrate the field, happened to look like it:

```json
{ "title": "Area·Type·Operation" }
```

So the schema was not describing the field. It was demonstrating the wrong format, in
the one place the model pays most attention to.

## Why

There were not three code paths that build a title. There were four. The fourth is the
model, and it does not call your functions.

When a value appears in the prompt context in a shape that plausibly satisfies a schema
field, generating that field by copying is the cheapest continuation available. It costs
the model nothing and it is locally consistent with everything it can see — including a
schema example that resembles the raw string more than it resembles the intended output.

The failure is silent by construction. The output is valid JSON, it passes schema
validation, the field is a non-empty string of the right type. Nothing downstream has
any way to know that this particular string was copied rather than derived. Prompt
instructions reduce the rate; they do not make it zero, and a field with a canonical
source should not have a rate at all.

## The pattern

Decide, per field, who owns it — and enforce that in code, not in the prompt.

| Field | Owner |
|---|---|
| Derivable from canonical data (title, price, label, URL) | Your code. The model must not produce it. |
| Requires inference (summary, rationale, ranking) | The model. There is no canonical source. |

**Best: do not ask for the field at all.** Remove it from the response schema and build
it after the fact. A field the model cannot emit is a field it cannot get wrong.

**When the schema must keep it**, overwrite it deterministically after validation and
before the response leaves the service:

```ts
// after schema validation, before returning to the caller
const byId = new Map(records.map(r => [r.id, r]));

for (const item of response.items) {
  // the model returns ids as strings even when the source type is numeric
  const id = typeof item.id === 'string' ? Number(item.id) : item.id;
  const record = byId.get(id) ?? byId.get(item.id as never);
  if (record) item.title = buildTitle(record);   // canonical wins, always
}
```

Two details that are load-bearing:

- **Coerce the id type.** A model given numeric ids will frequently return them quoted.
  A `Map` keyed on numbers then misses every lookup, and the normalization pass becomes
  a no-op that fails exactly as silently as the bug it was meant to fix.
- **Do not skip on mismatch.** If the id resolves to no record, that is a problem worth
  logging — the model may have invented an item that is not in the context at all.

And fix the schema example while you are there: an example value is a demonstration, not
documentation. Make it show the output you want.

## How to verify

Run the pipeline over a handful of records and assert equality against the deterministic
function — not a regex, not a "looks reasonable" check:

```ts
for (const item of response.items) {
  const record = byId.get(Number(item.id));
  expect(item.title).toBe(buildTitle(record));   // exact, or the model authored it
}
```

To find the exposure across an existing service without running anything, ask of every
field in your response schema: *does a function in this repository already compute this?*
Every field where the answer is yes, and which the model is still allowed to emit, is
this bug waiting for the right input.

The fastest live probe: pick a record whose raw source string differs visibly from its
formatted output, put it in the context, and inspect the field. If the two forms are
identical for every record in your test data, the test cannot detect this failure at all
— pick different data.

## See also

- The same ownership question applies to ids, currencies and dates in structured model
  output: anything with a canonical representation should be written by code that has
  the canonical value, with the model's version discarded rather than validated.
