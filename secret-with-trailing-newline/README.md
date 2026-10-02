# A secret stored with a trailing newline passes every check you wrote and fails in production

**English** · [Español](README.es.md)

`secret-manager` · `correctness` · `2026`

## Context

A service whose credential is generated with a command-line tool and stored in a secret
manager, then read by the application as an environment variable. A token check, a signed
callback or a login flow compares the stored value against what a client sends.

## What failed

**Every client got `401` — and the checks we ran by hand said the secret was correct.** The
same happened a second time in a different service: an OAuth sign-in failed, and we spent
minutes comparing hashes of the secret in two places that looked identical.

## Why

Command-line tools end their output with a newline. Piped into the secret manager, that
newline is stored as part of the value:

```bash
openssl rand -hex 32 | gcloud secrets versions add my-secret --data-file=-
#                      ^ the stored value is 65 bytes: 64 characters plus "\n"
```

The application reads 65 bytes and compares them with the 64 the client holds. They never
match.

What hid it was the shell itself. **Command substitution strips trailing newlines**, so
any check written as `$(...)` silently removes the very byte that is wrong:

```bash
# both sides lose the newline, so the hashes agree — and the stored secret is still broken
[ "$(gcloud secrets versions access latest --secret=my-secret | sha256sum)" = \
  "$(printf '%s' "$EXPECTED" | sha256sum)" ] && echo "looks fine"
```

The verification was structurally unable to see the defect it was meant to find.

## The pattern

**Write the value without the newline.**

```bash
openssl rand -hex 32 | tr -d '\n' | gcloud secrets versions add my-secret --data-file=-

# or, for a value you already hold in a variable
printf '%s' "$VALUE" | gcloud secrets versions add my-secret --data-file=-
```

**Compare raw bytes, never a `$(...)` result.** Look at the last byte, or count:

```bash
gcloud secrets versions access latest --secret=my-secret | xxd | tail -n 1
# a trailing "0a" is the newline

gcloud secrets versions access latest --secret=my-secret | wc -c
# 64 expected for a 32-byte hex value; 65 means a newline is stored
```

**Decide on purpose whether the application trims.** Trimming on read hides this class of
bug, which is convenient and also how you stop noticing that the stored value is wrong.
Either trim everywhere and document it, or trim nowhere and keep the stored value exact.

## How to verify

For every secret you generate by script, run the `wc -c` check above and compare it with
the length you expect. A mismatch of exactly one byte is this bug.

## See also

- [`secrets-flag-replaces-the-list`](../secrets-flag-replaces-the-list/README.md) — the other way a deploy leaves a secret wrong without any error.
