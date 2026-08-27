---
title: Troubleshooting
parent: index
tags: [guide, atproto, ops]
relations: [relates_to=standard-site, relates_to=guides/test-runs]
published_at: 2026-08-27T18:55:00Z
summary: How to diagnose a failed standard-site publish or smoke — the InvalidNonce OAuth finding, the app-password vs OAuth tradeoff, and a step-by-step failure checklist.
---

# Troubleshooting

This page collects the failure modes observed while operating Boris's
`standard-site` family against bsky.social / bsky.network PDS instances,
and how to diagnose them. The full run history is in
[[guides/test-runs]]; the happy-path procedures are in
[[guides/standard-site-tests]] and [[standard-site]].

## The InvalidNonce finding (OAuth + DPoP writes)

**Symptom:** `boris standard-site smoke` (or any OAuth write) fails with
`InvalidNonce` (exit 3), reproducibly, on a freshly signed-in session,
against bsky.network PDS instances.

**What happens:** `standard-site login --did <did>` succeeds — the
read-side discovery and token exchange complete — but the record writes
are rejected at the DPoP nonce layer. The OAuth access token is bound to
the DPoP key, and each request must carry a proof with a server-issued
nonce; the PDS rejects the proof. The refresh path compounds it: an
expired access token (1-hour lifetime) triggers a DPoP refresh that
fails at the network layer rather than surfacing a clean "session
expired" state.

**What it means:** on `bsky.social` / `bsky.network`, the OAuth + DPoP
write path is **non-parity** — `login` works but writes are rejected.
This matches the upstream Boris note that bsky.social does not reliably
grant the Standard.site write scope via the OAuth/DPoP path.

**Workaround:** use the **app-password path** for live writes (see
below). The recorded passing live smoke against bsky.social used the
opt-in app-password (`Bearer`) path.

**If OAuth writes are a hard requirement:** they need a self-hosted PDS
that implements and grants the granular collection scopes
(`repo:site.standard.document`, `repo:site.standard.publication`). The
OAuth path is not the supported live target on bsky.social today.

## App-password vs OAuth — the tradeoff

| | OAuth + DPoP | App-password |
|---|---|---|
| Scope | granular (`site.standard.*`) | broad account write |
| Token binding | DPoP key binding | Bearer |
| Works on bsky.social | ✗ (writes fail with `InvalidNonce`) | ✓ |
| Credential entry | browser OAuth | stdin only |
| Least privilege | yes | no — use a dedicated identity |
| CI-friendly | no (needs interactive grant) | yes, via a repo secret |

- **OAuth** is the *right* path in principle (least privilege, key
  binding) but doesn't complete writes against bsky.network today.
- **App-password** is the *working* path against bsky.social. It grants
  broad account write within its scope, so it must be kept on a
  dedicated identity and revocable — and the credential is accepted
  **only via stdin**, never argv/profile/env.

Decision rule: if you're publishing to bsky.social, use app-password.
If you control a compliant PDS and need granular scopes, use OAuth.

## Diagnosing a failed publish

When `boris standard-site publish` (or `smoke`) fails, walk this
checklist in order:

### 1. Did it fail offline or live?

- **Offline failure** (`plan`, `records`, `verify`) — no network, no
  credentials involved. The problem is the input or the profile.
- **Live failure** (`login`, `publish`, `smoke`) — network + auth are
  involved. The problem is usually the session, the PDS, or the plan
  drift.

### 2. Check the session

```text
boris standard-site sessions
```

- **No session for the identity** → `login` first.
- **Session present but stale** → the access token may be expired (1-hour
  lifetime for OAuth). Re-run `login` (OAuth) or `login --app-password`
  (app-password). For OAuth sessions past expiry, expect the DPoP
  refresh failure described above; re-auth cleanly.
- **"sessions" is empty** → the session store is elsewhere or was
  erased. Use `--session-root` to confirm which store you're reading.

### 3. Check the plan matches the profile (fail-closed guard)

```text
boris standard-site plan --profile standard-site.json --out /tmp/plan.json
boris standard-site publish --profile standard-site.json --plan /tmp/plan.json
```

Publishing with a committed `--plan` is **fail-closed on drift**: if the
profile's DID, origin, or record projection diverges from the committed
plan, publish refuses. If publish dies on drift, regenerate the plan,
review the diff, and commit the new plan — then publish against it.

### 4. Check the identity + origin in the profile

```text
grep -E '"(did|base_url|origin)"' standard-site.json
```

- **Wrong DID** → the publish targets the wrong account.
- **Placeholder origin** (`https://boris.example.net/` or similar) →
  the plan projects a fake URL; `verify` will report
  `overall_passed: false` with the well-known file `missing`. Fill in
  the real origin and rebuild.
- **Old placeholder DID** (`did:plc:aaaa…`) → retired; replace with the
  real DID.

### 5. Check verify against the built tree

```text
boris standard-site verify --profile standard-site.json --dist dist --out verify.json
```

- `overall_passed: false` with well-known `missing` → the build wasn't
  run with `--profile`, or the origin doesn't match. Rebuild with the
  profile flag (this is what emits `.well-known/site.standard.publication`).
- `overall_passed: false` with `documents` empty → pages lack
  `published_at` + `summary`; they're excluded with reason
  `missing-date`. Add both to the frontmatter (full UTC:
  `YYYY-MM-DDTHH:MM:SSZ`).
- `overall_passed: true` → the offline gate is green; the failure is
  live-side.

### 6. Live-side failures

- **`InvalidNonce` (exit 3)** → OAuth + DPoP write path; switch to
  app-password (see above).
- **Auth rejection on publish** → the app-password may be revoked or
  wrong. Revoke and re-mint if it was ever exposed (e.g. pasted in a
  chat), then re-login.
- **Network error during refresh** → the access token expired and the
  DPoP refresh failed at the transport layer. Re-login to get a fresh
  session rather than fighting the refresh.
- **Record already exists / conflict** → the publish reconciles against
  the live repository; a stale committed plan may describe records that
  no longer match. Re-plan against the current profile and re-publish.

### 7. The evidence trail

Every command writes an artifact:

| Command | Artifact | Tells you |
|---------|----------|-----------|
| `plan` | `evidence/plan.json` | the projection (documents, exclusions, well-known path) |
| `records` | `evidence/records.json` | the canonical payloads |
| `verify` | `evidence/verify.json` | offline gate status |
| `publish` | `evidence/publish*.json` | per-record outcome (`created`/`updated`/`failed`) + CIDs |
| `smoke` | `evidence/live-smoke.json` | per-phase result (discovery → cleanup) |

Check the artifact for the failing step first — the diagnosis is usually
written down in it.

## Still stuck?

- Confirm which identity you're actually publishing as: `sessions`.
- Confirm the PDS is reachable: resolve the DID and check the origin.
- Read [[guides/test-runs]] for the recorded findings, including the
  InvalidNonce investigation and the app-password resolution.
- The upstream project docs at
  [drawmeanelephant/boris](https://github.com/drawmeanelephant/boris)
  are the canonical source for the tool itself.
