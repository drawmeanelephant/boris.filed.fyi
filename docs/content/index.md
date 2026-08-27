---
title: Boris on ATProto
tags: [home]
published_at: 2026-08-27T17:30:00Z
summary: A hands-on technical overview of the Boris Zig content compiler and the AT Protocol surface it publishes to — a site built with Boris, documenting Boris.
---

# Boris on ATProto

This site is a hands-on technical overview of **Boris** — the Zig content
compiler — and **the AT Protocol** surface it publishes to. It is also a
living demo: every page here is compiled by Boris itself, and the whole
site can be projected onto the `standard.site` lexicon and published to an
ATProto PDS. The docs are an artifact of the tool under description.

## What you are looking at

Boris is a single native binary that turns a directory of Markdown files
into a verified static site. It is not a framework and it has no runtime:
it produces frozen HTML, a frozen graph, a frozen search index, and a
**publication proof pack** — a set of artifacts, checks, and claims whose
hashes pin what was built. The AT Protocol gives that frozen site a place
to live on the network: a `site.standard.publication` record that points a
DID at a production URL, and `site.standard.document` records that point
at individual pages.

The three ideas this site explores:

- [[boris]] — what Boris is, how the build pipeline works, what the proof
  pack guarantees, and how IR/RAG/context exports turn the same content
  into machine-readable corpora.
- [[atproto]] — the AT Protocol in the shape Boris cares about: DIDs,
  PDSes, XRPC, records, lexicons, and the two auth paths (OAuth + DPoP vs
  app-password) that gate writes.
- [[standard-site]] — the meta guide: how to take a Boris-built site and
  project, verify, authorize, and publish it to ATProto with the
  `boris standard-site` family. This page is the operator's manual, and
  it is exercised by the very site it documents.

## Why this is interesting

A static site generator that emits a cryptographic proof pack, wired to
a decentralized identity and record store, is a different shape from a
typical "deploy to a CDN" pipeline. The production URL is not an
incidental detail — it is the anchor that ties a DID to a built tree, and
Boris treats it as publication truth. `standard-site plan` projects the
records **offline** and deterministically; `standard-site verify`
cross-checks the built output against that projection **offline**; only
then does an explicit, session-gated `publish` write to the network.

This site is built with two commands (the HTML build with profile +
sitemap, then the standalone `llms.txt` export):

```text
boris build --input content --html-dir dist --theme themes/boris \
  --profile standard-site.json --sitemap --site-url https://boris.filed.fyi/
boris --input content --llms --llms-path dist/llms.txt
```

The build emits `dist/` plus:

- `dist/_boris/proof/` — the publication proof pack (artifacts, checks,
  claims, touches). Inspect it at `dist/_boris/proof/index.html` after a
  build — it renders the artifacts, checks, and claims as a report.
- `dist/_boris/search/search-index.json` — the frozen search index.
- `dist/.well-known/site.standard.publication` — the verification surface
  (the publication record's AT-URI, for readers who fetch the well-known
  URL to confirm the DID points at this origin).
- `dist/sitemap.xml` — a deterministic sitemap of all 9 pages under
  `https://boris.filed.fyi/`.
- `dist/llms.txt` — a deterministic LLM-consumable index of every page
  with its title and summary.

## The meta loop

This site is recursively self-describing:

1. The Markdown in `content/` is the source.
2. `boris build` compiles it to `dist/` and writes the proof pack, the
   well-known file, and the sitemap.
3. `boris --llms` exports `llms.txt` from the same content graph.
4. `boris standard-site plan` projects the records offline using
   `standard-site.json` (the DID + URL that anchor publication).
5. `boris standard-site verify` cross-checks the built tree offline.
6. `boris standard-site login` + `publish` writes to a real PDS, gated by
   a stored session and a committed-plan fail-closed guard.

Steps 1–5 are offline, deterministic, and CI-safe. Step 6 is explicit,
manual, and never implicit. The whole loop is documented in
[[standard-site]] and exercised by the test matrix in [[guides/testing-plan]],
whose results are logged in [[guides/test-runs]].

## Site identity

The `standard-site.json` profile is wired to a real identity,
`did:plc:jiokpoojzqntdpyw5xvfr7rv` (handle `boris.filed.fyi`, PDS
`https://auriporia.us-west.host.bsky.network`, app-password session).
The production origin is `https://boris.filed.fyi/` — the offline
plan/records/verify steps run against it, and a real `publish` writes
the `site.standard.publication/self` record to that PDS.

The earlier placeholder DID (`did:plc:aaaa…`) and the throwaway test
identity (`did:plc:xrqjadveiamk7vq7sfnvfddz`) are both retired from
this site's config; nothing here references the old identities.

## Anatomy of this site

```text
docs/
  content/
    index.md                        this trunk page
    boris.md                        Boris technical overview
    atproto.md                      AT Protocol technical overview
    standard-site.md                the meta guide (operator's manual)
    guides/testing-plan.md          test matrix, parent: index
    guides/standard-site-tests.md   per-surface procedures
    guides/test-runs.md             running log of executed tests
    guides/getting-started.md       authoring a page
    guides/publishing.md            the publication profiles
  themes/boris/                     the Boris theme (layouts + assets)
  boris.json                        GitHub Pages publication profile
  standard-site.json                Atmosphere profile (real DID + boris.filed.fyi origin)
  dist/                             built HTML + proof pack + well-known + sitemap + llms.txt
  evidence/                         offline plan/records/verify + live publish + smoke
```

## Read order

New here? Skim [[boris]] for the mental model, then [[atproto]] for the
network side, then [[standard-site]] for the loop that ties them together.
If you want to reproduce the build, the procedures live in
[[guides/standard-site-tests]] and the run log in [[guides/test-runs]].
