---
title: Test Runs
parent: index
tags: [runs, status]
relations: [relates_to=guides/testing-plan, relates_to=standard-site]
published_at: 2026-08-27T17:30:00Z
summary: A running log of executed tests against the boris-agent-kit, with commands, results, and evidence artifacts.
---

# Test Runs

A running log of executed tests against the `ef23e37785ce` kit. Each entry
records the procedure area, the command used, the observed result, and the
evidence artifact. The loop these tests exercise is documented in
[[standard-site]]; the kit under test is described in [[boris]].

## Run log

| Date | ID | Command | Result | Evidence |
|------|----|---------|--------|----------|
| 2026-08-27 | T-01 | `standard-site plan` | **passed** | `evidence/plan.json` (offline, stable, real DID) |
| 2026-08-27 | T-02 | `standard-site records` | **passed** | `evidence/records.json` (offline, real DID) |
| 2026-08-27 | T-03 | `standard-site verify` | **passed** (well-known match, 9 docs verified) | `evidence/verify.json` |
| 2026-08-27 | T-00 | `boris build` (site) | **passed** | `dist/**/*.html` |
| 2026-08-27 | T-04 | `standard-site login --did` (OAuth) | **partial** — read-side OK, writes fail | `sessions` |
| 2026-08-27 | T-05 | `standard-site login --app-password` | **passed** (stdin-only secret) | `sessions` (flavor `app-password`) |
| 2026-08-27 | T-08 | `standard-site smoke` (app-pw, throwaway identity) | **passed — full live round-trip** | `evidence/live-smoke.json` |
| 2026-08-27 | T-07 | `standard-site publish` (app-pw, `boris.filed.fyi`) | **passed — `self` record created live** | `evidence/publish.json` |
| 2026-08-27 | T-07b | `standard-site publish` (9 doc records) | **passed — 9 docs created, 1 self updated** | `evidence/publish-docs.json` |
| 2026-08-27 | T-09 | Cloudflare Pages deploy (`boris-product`) | **passed — `boris.filed.fyi` serving** | `https://boris.filed.fyi/` |

## Notes

- **T-01 insight** — Documents are excluded until they carry a `published_at`;
  the starter's pages appear in `exclusions` with reason `missing-date`. That is
  expected and worth recording as documented behavior, not a bug.
- **T-02 insight** — The canonical record for `site.standard.publication` with
  rkey `self` is `{"url":..., "name":...}`; the plan adds the derived `at_uri`,
  description, and `payload_sha256`.
- **T-03 insight (important)** — `verify` initially reported
  `overall_passed: false` from `evidence/verify.json`: the well-known file was
  `missing` and `documents` was empty. This was the expected fail-closed
  behavior before a real deployment URL and published documents existed. It
  was **not a failure of the tool** — it was the verification gate doing its
  job.

**Update 2026-08-27 (T-03 now PASSED):** once the build was run with
`--profile standard-site.json` (which emits the
`.well-known/site.standard.publication` file) and every page carried a
`published_at` date + `summary`, `verify` now reports `overall_passed: true`:
well-known `match`, all 9 documents `verified`. See
`evidence/verify.json`.

## Pending / blocked

- **T-08 smoke (live)** — attempted 2026-08-27 against the stored
  `did:plc:xrqjadveiamk7vq7sfnvfddz` OAuth session (PDS `pholiota.us-west.host.bsky.network`)
  with `--namespace boris-smoke-test`. Result: **blocked — `InvalidNonce` (exit 3)**.
  Both stored OAuth access tokens are expired (obtained 2026-08-21 / 2026-08-26,
  1-hour lifetime, well past `expires_in`), and the DPoP refresh is failing at the
  network layer rather than returning a clean "session expired" state. The live path
  needs a fresh authorization before it can create/readback/delete the two temporary
  records.

**Update 2026-08-27 (post re-auth):** `standard-site login --did
did:plc:xrqjadveiamk7vq7sfnvfddz` re-signed-in successfully and the session was
refreshed (`sessions` confirms the PDS binding). **T-08 still failed with
`InvalidNonce` (exit 3)** on **both** test DIDs / PDS instances
(`pholiota.us-west.host.bsky.network` and `morel.us-east.host.bsky.network`).

### Finding (T-08, OAuth path)

The live smoke's **OAuth + DPoP write path fails with `InvalidNonce` against
bsky.network PDS instances, reproducibly, on a freshly signed-in session**.
`login` itself succeeds (read-side discovery + token exchange), but the smoke's
record writes are rejected at the DPoP nonce layer. This is consistent with the
upstream Boris note that bsky.social does not reliably grant the Standard.site
write scope via the OAuth/DPoP path, and that the **recorded passing live smoke
against bsky.social used the opt-in app-password (`Bearer`) path instead**
(2026-08-16 fixture).

### Resolution (T-05 + T-08: app-password path) — PASSED

**2026-08-27, `did:plc:xrqjadveiamk7vq7sfnvfddz`, PDS `pholiota.us-west.host.bsky.network`.**
Logged in with a dedicated throwaway test identity app-password
(`standard-site login --app-password`, secret read once from stdin — never
argv/env/profile), then ran `standard-site smoke --namespace boris-smoke-test`.
**Overall verdict: passed (exit 0).** This throwaway identity is now retired;
the production identity for this site is `did:plc:jiokpoojzqntdpyw5xvfr7rv`
(`boris.filed.fyi`).

From `evidence/live-smoke.json` (`boris-live-smoke-result`, `boris/0.8.1`):

| Phase | Result |
|-------|--------|
| discovery | passed |
| authorization | passed (Bearer app-password path) |
| write | passed — `site.standard.publication` + `site.standard.document`, unique rkeys |
| readback | passed — AT-URI + value + CID verified for **both** records |
| verification surface | skipped (no `--surface-url`) |
| indexer | skipped (no `--indexer`) |
| cleanup | passed — **both records deleted, zero residue** |

**Net finding:** the OAuth/DPoP write path is non-pairity on bsky.network
(`InvalidNonce`), but the **app-password (`Bearer`) path completes the full live
interop round-trip cleanly against bsky.social**. This matches the upstream
recorded fixture. Granular `repo:site.standard.*` scopes are not what the
app-password path relies on; app-password grants broad account write (revoke
after use), so it must stay on a dedicated, disposable test identity.

**Implication for the plan:** the supported live parity target on bsky.social is
the app-password path. OAuth writes need a self-hosted PDS that grants the
granular collection scopes.

- **T-07 (publish) — PASSED** — `standard-site publish --profile standard-site.json`
  wrote the `site.standard.publication/self` record to
  `did:plc:jiokpoojzqntdpyw5xvfr7rv` (PDS `auriporia.us-west.host.bsky.network`),
  intent `create`, verified via write response. The record is now live at
  `at://did:plc:jiokpoojzqntdpyw5xvfr7rv/site.standard.publication/self`
  (CID `bafyreibsgzcmlq37gspdgz2avdiizr6erfp5dx4gqwslsu7w5j2t6yzjgu`). See
  `evidence/publish.json`.
- **T-04 / T-05 / T-06 (live auth)** — `sessions` (T-06) runs and lists all
  stored DIDs/flavors/PDS with no secrets, including the production identity
  `did:plc:jiokpoojzqntdpyw5xvfr7rv` (`boris.filed.fyi`). OAuth re-login works
  (T-04 partial: read-side authorization succeeds). Live `publish`/`smoke` OAuth
  writes are non-parity on bsky.network per the finding above; the production
  identity uses the app-password path.
- **T-09 (context)** and **T-10 (nostr)** — offline or scheduler-only; not yet run.
