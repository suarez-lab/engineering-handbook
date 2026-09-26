# Five deploys failed on an error message that was true, precise, and pointing at the wrong thing

**English** · [Español](README.es.md)

`node` · `outage` · `2026`

## Context

A Node.js/TypeScript service on a managed container platform. Secrets are injected by the
platform as environment variables, and the service validates them at boot with the usual
guard: a `validateRequiredEnvVars()` that reads `process.env`, and exits loudly if something
is missing.

This entry applies to any runtime where a missing-configuration error is the *first* thing you
see, and where the configuration is demonstrably present.

## What failed

Five consecutive deploys came up dead with the same line:

```
GEMINI_API_KEY not configured
```

The misleading part is that the message was **true**. By the time the validator ran, the
variable really was absent from `process.env`. It was also **precise** — the right variable
name, from the right guard, at the right point in boot. Nothing about it was a red herring in
the usual sense.

And yet the platform was injecting it. The revision's configuration listed the secret. The
secret existed and had a current version. Dumping the environment from a shell in the same
image showed the value.

So the investigation went where the message pointed: secret storage and IAM. A redundant
accessor role was granted to the runtime identity — it changed nothing. A permission was
re-checked, a binding re-applied, the secret re-versioned, the revision redeployed. Five
revisions burned. Roughly ninety minutes.

## Why

Somewhere in the codebase, a getter did this:

```ts
delete process.env.GEMINI_API_KEY;   // "defensive": read once, then remove
```

The intent was to avoid leaking the key through a later environment dump. Harmless in
isolation. The problem is *when* it ran.

The getter was reached from a class property initializer. That class was instantiated at
**module top level** in a routes file:

```ts
// routes/something.ts
const service = new SomeService();   // runs during import, not during startup
```

Module top-level code runs while the module graph is being resolved — during `import`, before
a single line of your `main()` executes. So the sequence was:

1. `import` walks the route modules.
2. `new SomeService()` runs its property initializers.
3. One of them reads the key through the getter, which deletes it from `process.env`.
4. `main()` finally starts and calls `validateRequiredEnvVars()`.
5. The variable is gone. The validator is correct. The message is correct. The diagnosis is not.

The platform did inject the variable. The application removed it, a few milliseconds before
checking for it. No amount of IAM work could have found this, because the error was not where
the error was reported.

## The pattern

The discipline that would have found this in ten minutes instead of ninety:

- **Root cause before any fix.** A fix applied to an unconfirmed cause is a guess, and a guess
  that happens to change behaviour is worse than one that does not — it teaches you something
  false. The IAM grant was a guess. It "worked" in the sense that it applied cleanly, and it
  moved the investigation further from the answer.
- **One hypothesis, one minimal change, per attempt.** If two things change and the symptom
  moves, you have learned nothing about either.
- **Three failed fixes means the model is wrong, not the fix.** Stop patching. The question
  is no longer "why is this variable missing" but "who else touches `process.env` for this
  key, and when". That question is one grep away:

  ```bash
  grep -rn "process\.env\.MY_VAR\|delete process\.env" src/
  ```

- **Trace the value upward, not the error downward.** The error told you where the value was
  *absent*. Find every place it is written, read, or removed, and order those by execution
  time. In Node, "execution time" includes the entire import graph, which runs before your
  entrypoint.
- **Instrument boundaries, not internals.** One log line at the top of the entrypoint and one
  at the top of each module that touches the value would have shown the delete happening
  before the validation, with no further reasoning required.
- **Separate a real regression from a pre-existing failure before debugging it.** When a suite
  goes red while you are mid-change, stash exactly the file you touched and rerun:

  ```bash
  git stash push -- src/path/to/changed-file.ts
  npm test -- path/to/failing.spec.ts
  git stash pop
  ```

  Still red? It was already broken and you are chasing someone else's bug. Green? It is yours.

## How to verify

Two checks, under five minutes.

**Is anything mutating the environment before your validator?** Print the ordering. Put this
as the very first statement of your entrypoint, before every other import:

```ts
// entrypoint: first line, above all other imports
console.log('[boot] pre-import env keys:', Object.keys(process.env).length);
process.on('exit', () => console.log('[boot] exiting'));
```

Then add a second count immediately after the imports. If the number drops, something in the
import graph is deleting variables.

**Is anything instantiating at module scope?** In a codebase you did not write, this finds the
candidates:

```bash
grep -rn "^const .* = new " src/ --include="*.ts"
grep -rn "delete process\.env" src/
```

Any hit from the first command is code that runs during `import`. Any hit from the second, in
a file reachable from the first, is this bug.

## See also

- [A dedupe guard that only reads history cannot see the duplicate it is creating right now](../bounded-agent-loops/README.md)
  — another case where the first instinct ("the data source is wrong") pointed away from the
  actual mechanism.
