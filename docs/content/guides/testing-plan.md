---
title: Testing Plan
parent: index
tags: [plan]
relations: [relates_to=guides/standard-site-tests, relates_to=standard-site]
published_at: 2026-08-27T17:30:00Z
summary: The test matrix, acceptance criteria, and identity plan for validating the Boris agent kit end to end against the AT Protocol.
---

# ATProto Testing Plan

The goal is to validate the **Boris agent kit** end to end against the
AT Protocol, focusing on the `boris standard-site` publication family. This
page is the master plan; each area links to its detailed procedure. The
operator's manual for the loop is [[standard-site]]; the ATProto and Boris
overviews are [[atproto]] and [[boris]].

## Kit under test

- Repository `drawmeanelephant/boris`, commit `ef23e37785ced4405191c233da216168aea8ff55` (`main`).
- Binaries `boris` + 10 tools, Darwin-arm64, Zig 0.16.0. Binary version `boris/0.8.1`.
- Platform is **non-portable** (Darwin-arm64 only) — recorded as a test constraint.
- Archived as `boris-agent-kit-ef23e37785ce.tar.gz` with `MANIFEST.json` + `SHA256SUMS`.

## Principle of operation

Boris treats the production URL as publication truth. ATProto publication is
**explicit, never implicit** — nothing is written to the network without an
explicit publish command and a stored session. Tests must reflect that: the
offline projection and verify steps run in CI-like fashion; the live smoke is
manual and never in CI.

## Test matrix

| ID | Area | Surface | Offline | Live | Priority | Status |
|----|------|---------|:-------:|:----:|:--------:|:------:|
| T-01 | Plan | `standard-site plan` | yes | no | high | passed |
| T-02 | Records | `standard-site records` | yes | no | high | passed |
| T-03 | Verify | `standard-site verify` | yes | no | high | passed |
| T-04 | Auth (OAuth) | `standard-site login --did` | no | yes | high | partial |
| T-05 | Auth (app-pw) | `standard-site login --app-password` | no | yes | medium | passed |
| T-06 | Sessions | `standard-site sessions` / `logout` | no | yes | medium | passed |
| T-07 | Publish | `standard-site publish` | no | yes | high | passed |
| T-08 | Smoke | `standard-site smoke` | no | yes | high | passed |
| T-09 | Context | `context` bundle | yes | no | low | planned |
| T-10 | Nostr | `nostr plan` / `sign` | yes | no | low | scheduled |

## Acceptance criteria

- **T-01/T-02** — Deterministic output: two runs with the same input and profile
  must produce byte-identical plans and canonical record payloads. All hashes and
  `at_uri`s are stable across runs.
- **T-03** — `verify` cross-checks the built head links and the
  `.well-known/site.standard.publication` project file offline, and fails on drift.
- **T-04/T-05** — A dedicated test identity authorizes without touching real account
  data. Credentials are accepted **only via stdin**, never argv/profile/env.
- **T-06** — `sessions` lists DID/flavor/PDS without secrets; `logout` securely
  erases the stored session.
- **T-07** — One-shot publish from the stored session, reconcile + rotate-or-die on
  refresh, and a committed-plan fail-closed guard.
- **T-08** — Smoke creates two records under unique `boris-smoke-<…>` rkeys,
  readback-verifies them, deletes them, and emits a `boris-live-smoke-result`
  artifact. Runs against a throwaway namespace and leaves no residue.

## Identity / handles

All live tests use a **dedicated test identity**, never a personal account.
This site is wired to `did:plc:jiokpoojzqntdpyw5xvfr7rv` (handle
`boris.filed.fyi`, PDS `auriporia.us-west.host.bsky.network`,
app-password session) — the production identity. An earlier throwaway
identity (`did:plc:xrqjadveiamk7vq7sfnvfddz`) passed the live smoke
and is retired. App passwords grant broad account write in its scope,
so the credential is scoped and revocable. See
[[guides/standard-site-tests]] for the auth flows and [[atproto]] for the
OAuth/DPoP vs app-password comparison.

## Status legend

- **planned** — not yet run; documented and ready.
- **scheduled** — queued after higher-priority items.
- **passed** / **failed** — recorded in [[guides/test-runs]].
- **blocked** — documented but cannot reach a passing live state in the current
  environment (see [[guides/test-runs]] finding on the OAuth DPoP path).
- **partial** — a subset of the area validated; remainder blocked.
