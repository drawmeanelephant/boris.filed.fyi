---
title: Standard.site Test Procedures
parent: index
tags: [tests, atproto]
relations: [relates_to=guides/testing-plan]
published_at: 2026-08-27T17:30:00Z
summary: Step-by-step procedures for each ATProto surface in the test matrix, with pass criteria.
---

# Standard.site Test Procedures

Step-by-step procedures for each ATProto surface in the test matrix. All paths
relative to the extracted kit root (the directory containing `bin/boris`).

## T-01 — Offline plan

```text
bin/boris standard-site plan --profile standard-site.json --out plan.json
```

- Produces a deterministic record projection: the `site.standard.publication`
  record, the `.well-known/` verification project path, and document exclusions.
- Documents are excluded until they have a `published_at` date; exclusion reasons
  are recordable and part of the expected output.
- **Pass** when: output is stable across repeated runs with identical input/profile,
  all `at_uri`s and `payload_sha256` values match, and no network/credential is touched.

## T-02 — Canonical records

```text
bin/boris standard-site records --profile standard-site.json --out records.json
```

- Dumps the full canonical record payloads (e.g. `{"collection":"site.standard.publication","rkey":"self",...}`).
- **Pass** when: the payload is byte-stable, `collection`/`rkey`/`at_uri` are correct,
  and it runs fully offline.

## T-03 — Verify built tree

```text
bin/boris standard-site verify --profile standard-site.json --dist dist --out verify.json
```

- Cross-checks the built head links and the well-known file **offline** against the
  profile. Fails on drift between the profile URL and the built output.
- **Pass** when: it reports the expected `.well-known/site.standard.publication`
  project file at the required public URL and a matching set of document links.

## T-04 — Auth, OAuth (primary)

```text
bin/boris standard-site login --did did:plc:TESTID
```

- Browser OAuth requesting granular repo scopes
  (`site.standard.document`, `site.standard.publication`).
- Not granted by `bsky.social`; requires a compliant PDS.
- **Pass** when: session persists to a `0600` store and `sessions` lists it.
- **Skip** if the test PDS does not implement OAuth; fall back to T-05 only for
  a dedicated test identity.

## T-05 — Auth, app-password (opt-in)

```text
bin/boris standard-site login --app-password --handle <test-handle>
```

- Password is read **once from stdin** only — never from argv, profile, or env.
- Works on `bsky.social`. Grants broad account write **in its scope**, so use a
  dedicated, disposable test identity.
- **Pass** when: session persists, `sessions` shows the DID/flavor/PDS with no
  secrets, and `logout` erases it.

## T-06 — Sessions lifecycle

```text
bin/boris standard-site sessions
bin/boris standard-site logout --handle <test-handle>
```

- **Pass** when: `sessions` lists DID/flavor/PDS without exposing keys, `logout`
  securely erases the store, and a later `sessions` no longer lists it.

## T-07 — Publish

```text
bin/boris standard-site publish --profile standard-site.json --out evidence.json
```

- One-shot publish from a stored session; reconcile + rotate-or-die on refresh.
- With a committed `--plan PATH` it is fail-closed on drift against that plan.
- **Pass** when: `evidence.json` is emitted with the created `at_uri`, the record
  is readable back from the PDS, and pruning only occurs with explicit `--prune`
  authority.

## T-08 — Live interop smoke

```text
bin/boris standard-site smoke --did did:plc:TESTID --namespace boris-smoke-test --out smoke.json
```

- Manual, opt-in, **never in CI**. Bounded, leaves no residue.
- Creates two records under unique `boris-smoke-<…>` rkeys, readback-verifies
  them against the PDS, deletes them, and emits a `boris-live-smoke-result`.
- Uses a clock-derived namespace by default; pass `--namespace` for a throwaway prefix.
- Optional: `--surface-url` to verify the served origin, `--indexer` for a
  non-normative AppView observation.
- **Pass** when: both records are written and read back, both are deleted, and the
  result artifact records the whole lifecycle.
