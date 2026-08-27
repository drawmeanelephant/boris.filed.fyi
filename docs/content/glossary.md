---
title: ATProto glossary
parent: index
tags: [reference, atproto]
relations: [relates_to=atproto, relates_to=boris]
published_at: 2026-08-27T19:05:00Z
summary: A quick-reference glossary of AT Protocol terms — DID, PDS, XRPC, record, lexicon, AT-URI, AppView, DPoP, Bearer — each linked back to the overview pages.
---

# ATProto glossary

A quick-reference glossary of the AT Protocol terms this site uses. Each
entry links to the deeper treatment in [[atproto]]; the Boris-side view
is [[boris]], the publication loop is [[standard-site]], and failure
modes are in [[troubleshooting]].

## Identity

**DID** — Decentralized Identifier. The portable, self-certifying
identifier that owns records. ATProto uses `did:plc:` (the dominant
production method) and `did:web:` (DNS-anchored). A DID resolves to a
DID document that points at the account's PDS and authorization server.
Portable: the owner can move the account to a different PDS without
changing the identifier or the records. This site's identity is
`did:plc:jiokpoojzqntdpyw5xvfr7rv`.

**Handle** — A human-readable name that resolves to a DID (e.g.
`boris.filed.fyi`). Handles are the friendly alias; DIDs are the
canonical identifier. Boris accepts either for login
(`--did` or `--handle`).

## Hosting and data

**PDS** — Personal Data Server. The server that hosts the account's
repository of records and speaks the HTTP API (XRPC) for reading and
writing them. The PDS is where records physically live. This site's PDS
is `https://auriporia.us-west.host.bsky.network`.

**XRPC** — The AT Protocol's HTTP API surface for reading and writing
records (`com.atproto.repo.*` etc.). Boris talks to the PDS over XRPC
for `login`, `publish`, and `smoke`.

**Record** — A typed JSON value stored in a repository under a
`collection` + `rkey`, owned by a DID. The `standard.site` family uses
two: `site.standard.publication` (rkey `self`, the root pointer) and
`site.standard.document` (one rkey per page). See [[standard-site]] for
how Boris projects and publishes them.

**Lexicon** — The schema that defines a record's shape and validation
rules. The collection name namespaces the lexicon (e.g.
`site.standard.publication`). `standard.site` is a small, deliberate
lexicon family.

**AT-URI** — The canonical address of a record:
`at://<did>/<collection>/<rkey>`. It is stable across PDS moves because
it is keyed by the DID, not the server. Example:
`at://did:plc:jiokpoojzqntdpyw5xvfr7rv/site.standard.publication/self`.

## Network and discovery

**AppView / indexer** — Read-side services that aggregate records across
the network for discovery (e.g. a Bluesky feed). Non-normative for
publication: Boris's `smoke` can observe an AppView with `--indexer`,
but publication itself never depends on one.

**Verification surface** — The `.well-known/site.standard.publication`
file served at the site's origin. It carries the publication record's
AT-URI so a reader can confirm the hosted site claims the DID it points
at. This is the bridge between the hosting layer (Cloudflare) and the
network layer (the PDS) — see [[operations]].

## Authorization

**DPoP** — Demonstrating Proof of Possession (RFC 9449). The mechanism
that binds an OAuth access token to a key the client holds; each request
carries a signed proof with a server-issued nonce. This is the OAuth +
DPoP write path — the one that fails with `InvalidNonce` against
bsky.network (see [[troubleshooting]]).

**Bearer** — The simple token-carrying auth scheme used by the
app-password path. The credential itself is the bearer of authority
(no key binding). This is the path that completes live writes against
bsky.social today.

## How they fit together

```text
handle ──resolves to──▶ DID ──owns──▶ records on a PDS (XRPC)
                                        │
                              site.standard.publication/self
                                        │  points at
                                        ▼
                              https://boris.filed.fyi/
                                        │  confirms via
                                        ▼
                              .well-known/site.standard.publication
                                        │  (the AT-URI)
                                        ▼
                              a reader resolves DID → URL → well-known → claim confirmed
```

For the full picture: [[atproto]] explains the pieces, [[boris]] the
tool that builds the site, [[standard-site]] the loop that publishes it,
[[operations]] how this site is run, and [[troubleshooting]] what to do
when it breaks.
