---
title: The AT Protocol
parent: index
tags: [overview, atproto]
relations: [relates_to=boris]
published_at: 2026-08-27T17:30:00Z
summary: The AT Protocol in the shape Boris cares about — DIDs, PDSes, XRPC, records, lexicons, and the two write paths that gate publication.
---

# The AT Protocol

The **AT Protocol** (ATProto) is a decentralized networking protocol for
social applications. It provides portable identity, interoperation
between services, and a shared data model so that content and
relationships are not siloed per app. This page is the technical
overview in the shape that **Boris** cares about: the pieces that let a
Boris-built static site become an on-network publication.

For the Boris side, see [[boris]]; for the loop that ties them together,
see [[standard-site]].

## The pieces Boris touches

ATProto has several layers; the ones that matter for a published site
are:

- **DIDs** — the portable, verifiable identifier that owns records.
- **PDS** — the Personal Data Server that hosts the account's repository
  of records and speaks the HTTP API.
- **Records** — typed JSON blobs stored in the repository under a
  `collection` + `rkey`.
- **Lexicons** — the schemas that define record shapes (the
  `site.standard.*` family is what Boris publishes).
- **Auth** — two write paths: OAuth + DPoP (granular scopes) and the
  app-password (broad account write, Bearer).
- **AppView / indexer** — read-side services that aggregate records for
  discovery (non-normative for publication).

### DIDs

A DID is a self-certifying, portable identifier. ATProto uses
`did:plc:` (the PLC method, the dominant production method) and
`did:web:` (DNS-anchored). A DID resolves to a DID document that points
at the account's PDS and authorization server. Portability means the
owner can move the account to a different PDS while keeping the same
identifier and records.

This site's profile (`standard-site.json`) is wired to the real
identity `did:plc:jiokpoojzqntdpyw5xvfr7rv` (handle `boris.filed.fyi`),
which resolves to the PDS at
`https://auriporia.us-west.host.bsky.network` and the authorization
server `https://bsky.social`.

### PDS and the HTTP API

The **Personal Data Server** hosts the account's repository. It speaks an
HTTP API (XRPC) for reading and writing records. A write to a PDS goes
through:

1. **Discovery** — resolve the DID to a PDS origin and authorization
   server metadata (RFC 9728).
2. **Authorization** — obtain a credential that authorizes the write.
3. **The write** — `com.atproto.repo.createRecord` / `putRecord` /
   `deleteRecord` against the PDS.
4. **Readback** — fetch the record by its AT-URI to confirm it landed.

An **AT-URI** (`at://<did>/<collection>/<rkey>`) is the canonical
address of a record. It is stable across moves because it is keyed by the
DID, not the PDS host.

### Records and lexicons

A record is a typed JSON value stored at `collection` + `rkey` under the
owner's DID. The collection name namespaces the record's lexicon (e.g.
`site.standard.publication`, `site.standard.document`). A lexicon defines
the record's fields and validation rules. The `standard.site` family is
small and deliberate:

- **`site.standard.publication`** (rkey `self`) — the root pointer. It
  carries the site's production `url` and `name`, and ties the DID to a
  deployed origin.
- **`site.standard.document`** (one rkey per page) — a per-page record
  that points into the published tree.

This is the shape Boris projects and publishes. `boris standard-site
records` dumps the canonical payloads offline (see the
[[standard-site]] guide).

### Verification surface

Because a `site.standard.publication` record points a DID at a URL,
publication is verifiable: a reader fetches the DID's record, learns
the URL, and checks that the URL serves what the record claims. Boris's
`standard-site verify` cross-checks the built tree against this
projection **offline** — it confirms the head links and the
`.well-known/site.standard.publication` file are consistent before any
network write.

## The two write paths

ATProto offers two ways to authorize a write, and Boris supports both.
They are not interchangeable, and the choice matters operationally.

### OAuth + DPoP (granular scopes)

The primary, scoped path. A client requests granular repository scopes
(e.g. `repo:site.standard.document`, `repo:site.standard.publication`)
via OAuth, and proves possession of the token with DPoP (RFC 9449,
Demonstrating Proof of Possession). Each write is signed with a DPoP
proof and a server-issued nonce.

- **Pros:** least privilege — the grant is scoped to specific
  collections, and the token binds to a key the client holds.
- **Cons:** requires a PDS and authorization server that implement the
  granular scopes and the DPoP nonce flow correctly.

**Operational note (recorded 2026-08-27):** against `bsky.social` /
`bsky.network` PDS instances, the OAuth + DPoP write path fails at the
DPoP nonce layer with `InvalidNonce`, reproducibly, on a freshly
signed-in session. Read-side discovery and token exchange succeed; the
record writes are rejected. See [[guides/test-runs]] for the full
finding. This means OAuth writes currently need a self-hosted PDS that
grants the granular collection scopes.

### App-password (broad write, Bearer)

The opt-in fallback. An app-password is a long-lived, broad-write
credential (a Bearer token) that works on `bsky.social`. Boris reads it
**once from stdin only** — never from argv, profile, or environment.

- **Pros:** works against the dominant production PDS today; completes
  the full create/readback/delete round-trip.
- **Cons:** broad account write within its scope — the grant is not
  collection-scoped, so it must be used with a dedicated, disposable
  test identity and revoked after use.

**Operational note (recorded 2026-08-27):** the app-password path
fully passed a live smoke against a throwaway test identity
(`did:plc:xrqjadveiamk7vq7sfnvfddz` /
`pholiota.us-west.host.bsky.network`): discovery, authorization, write
(publication + document, unique rkeys), readback (AT-URI + value + CID
verified for both records), and cleanup (both deleted, zero residue).
See `evidence/live-smoke.json` and [[guides/test-runs]]. The production
identity for this site is now `did:plc:jiokpoojzqntdpyw5xvfr7rv`
(`boris.filed.fyi`, PDS `auriporia.us-west.host.bsky.network`).

### Which path to use

| | OAuth + DPoP | App-password |
|---|---|---|
| Scope | granular (`site.standard.*`) | broad account write |
| Token binding | DPoP key binding | Bearer |
| Works on bsky.social | ✗ (writes fail with `InvalidNonce`) | ✓ |
| Credential entry | browser OAuth | stdin only |
| Least privilege | yes | no — use a disposable identity |

The supported live parity target on `bsky.social` today is the
app-password path. OAuth writes need a compliant PDS.

## Standards referenced

The live smoke's `spec.baseline` records the authoritative surfaces the
behavior was validated against:

- AT Protocol DID — the identifier method.
- AT Protocol OAuth — the authorization flow.
- AT Protocol Handle — handle resolution.
- AT Protocol HTTP API / XRPC — the record read/write surface.
- RFC 9728 (OAuth 2.0 Protected Resource Metadata, April 2025) — PDS
  resource metadata discovery.
- RFC 9449 (OAuth 2.0 Demonstrating Proof of Possession, September
  2023) — the DPoP token-binding layer.
- Standard.site permission contract — the `site.standard.*` collection
  scope model.

The next page, [[standard-site]], shows how these pieces combine into
the publish loop that turns a Boris build into an on-network site.
