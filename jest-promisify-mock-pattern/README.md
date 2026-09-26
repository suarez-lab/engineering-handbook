# `promisify` captures the function reference at import time — your mock arrives too late

**English** · [Español](README.es.md)

`jest` · `node` · `correctness` · `2026`

## Context

A Node module that wraps a command-line tool — an OCR binary, `ffmpeg`, `git`, anything
you shell out to. The idiomatic way to write it puts a `promisify` call at module scope:

```ts
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const execFileAsync = promisify(execFile);   // runs once, at import

export async function isToolAvailable(): Promise<boolean> {
  try {
    await execFileAsync("mytool", ["-v"]);
    return true;
  } catch {
    return false;
  }
}
```

Then a test needs to assert what happens when the binary is missing from the runtime —
a real scenario, because the container image and the developer's laptop rarely agree on
which binaries exist. The obvious move is to mock `child_process`.

## What failed

The test mocked `child_process` so that `execFile` fails, and asserted a rejection. The
function resolved instead — with a real value, from a real subprocess.

```
expect(received).rejects.toThrow()
Received promise resolved instead of rejected
```

The misleading part is that the mock is not broken and Jest is not misbehaving. If you
log inside the test, `require("node:child_process").execFile` really is the mock. The
assertion is correct. The module under test simply is not using it.

Worse on a machine where the binary *is* installed: the test does not fail loudly, it
quietly shells out to the real tool. On CI, where the binary may be absent, the same test
passes for the wrong reason. It is a test that measures the host, not the code.

## Why

`promisify(execFile)` is a call, not a reference. It executes the instant the module is
first imported, and what it returns is a new function that has closed over *the value of
`execFile` at that moment*.

Ordering, then:

1. Test file imports the module under test (directly, or transitively).
2. `const execFileAsync = promisify(execFile)` runs. The original is captured.
3. `jest.doMock("node:child_process", …)` replaces the module's export.
4. Calls go through `execFileAsync`, which still holds the original.

Jest can replace what the `child_process` module *exports*. It cannot reach into a
closure that already copied the old value out. The same trap applies to any
`const x = someModule.fn` at module scope — `promisify` is just the most common way to
write one without noticing you wrote one.

Hoisting hides the ordering. `jest.mock` is hoisted above the imports and would work
here; `jest.doMock` is not hoisted, which is precisely why it loses this race. Switching
between them to make a test pass is treating the symptom.

## The pattern

Mock the module that *owns* the wrapper, not the primitive it wraps. That is the boundary
your code actually depends on:

```ts
test("rejects when the tool is not installed on the runtime", async () => {
  jest.doMock("../src/services/tool.service", () => ({
    isToolAvailable: jest.fn().mockResolvedValue(false),
    runTool: jest.fn().mockRejectedValue(
      new Error("mytool is not installed in this runtime"),
    ),
  }));

  // Import AFTER the doMock — this is what makes doMock work at all.
  const mod = await import("../src/services/tool.service");

  await expect(mod.runTool(Buffer.from("input")))
    .rejects.toThrow("mytool is not installed");
});
```

If you genuinely need to exercise the wrapper's own logic — retries, argument building,
stderr parsing — then do not capture the reference at module scope in the first place.
Resolve it per call, which is mockable and costs nothing measurable:

```ts
import { execFile } from "node:child_process";
import { promisify } from "node:util";

export async function runTool(args: string[]) {
  // promisify at call time: reads whatever child_process exports right now
  return promisify(execFile)("mytool", args);
}
```

Or inject it, which makes the dependency visible in the signature and removes the need
for module mocking entirely:

```ts
export async function runTool(
  args: string[],
  exec = promisify(execFile),   // default argument, evaluated per call
) {
  return exec("mytool", args);
}
```

The rule, stated generally: **anything captured into a `const` at module scope is frozen
before your test runs.** Mock at the level of that `const`'s owner, or stop capturing.

## How to verify

**Find the pattern in a codebase.** A `promisify` at module scope is a single grep — the
tell is no indentation:

```bash
grep -rn "^const .* = promisify(" src/
```

Every hit is a module whose `child_process` mocks will silently do nothing.

**Prove a suspect test is not testing what it claims.** Add a marker to the mock that can
only appear if the mock was used:

```ts
jest.doMock("node:child_process", () => ({
  execFile: (...args: unknown[]) => {
    throw new Error("MOCK_WAS_REACHED");
  },
}));
```

Run the test. If it fails with `MOCK_WAS_REACHED`, the mock is wired correctly and the
test is real. If it fails with anything else — or passes — the mock was never consulted,
and the test has been passing on the host's installed binaries.

**Catch the CI-versus-laptop divergence.** Run the suite with the binary removed from
`PATH`; a test that behaves differently is testing the environment:

```bash
PATH=/usr/bin:/bin npx jest path/to/tool.service.test.ts
```

## See also

- [Cloud Run loses in-memory state on every restart](../cloudrun-sync-status-persistence/README.md) —
  a different consequence of the same habit: values captured once, at module scope, that
  everything downstream assumes are live.
