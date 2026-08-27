---
title: Boris
parent: index
tags: [overview, boris]
published_at: 2026-08-27T17:30:00Z
summary: What Boris is — a Zig content compiler that produces verified static HTML, a frozen graph, a search index, and a publication proof pack from Markdown.
---

# Boris

**Boris** is a single native binary — a content compiler written in Zig
(currently `boris/0.8.1`, built with Zig 0.16.0 for Darwin-arm64). It
takes a directory of Markdown (or Textile/Cooklang) and produces, in one
pass, a verified static HTML site, a frozen content graph, a search index,
and a publication proof pack. It is not a framework, has no runtime, and
runs no JavaScript on the output. Everything is frozen at build time.

This page is the technical overview. The operator's manual for the
ATProto side is [[standard-site]]; the AT Protocol background is
[[atproto]].

## What Boris compiles

A Boris site is a directory tree with:

- **`content/`** — one Markdown file per page, each with YAML frontmatter.
- **`themes/<name>/`** — a layout (`layouts/main.html`) and assets
  (`assets/...`). Layouts are closed-slot templates: `{{content}}`,
  `{{nav}}`, `{{breadcrumb}}`, `{{toc}}`, `{{head}}`, `{{footer}}`.
- **a publication profile** — `boris.json` (HTML target) and optionally
  `standard-site.json` (the ATProto/DID/URL projection).

A page is a Markdown file whose frontmatter shapes the graph:

```yaml
---
title: Boris
parent: index
tags: [overview, boris]
relations: [relates_to=atproto]   # optional semantic edges
---

# Boris

Body in Markdown. [[atproto]] is a real graph edge, not dead prose.
```

- `parent` — the structural parent's entity id (forms the nav forest and
  breadcrumb chain).
- `tags` — free-form list rendered into page metadata.
- `relations` — semantic edges like `relates_to=<target>`; the frozen
  graph exposes them for impact analysis.
- `[[wiki-links]]` — a link to a missing page **fails the build** rather
  than rendering as dead prose. The graph is strict.

## Build pipeline

```text
boris --input content --html-dir dist --theme themes/boris
```

A build pass produces, for one HTML target:

- `dist/**/*.html` — the rendered pages, each wrapped in the layout.
- `dist/assets/**` — copied theme assets (CSS, images, etc.).
- `dist/_boris/search/search-index.json` — a frozen search index derived
  from the committed HTML page set.
- `dist/_boris/proof/` — the publication proof pack (see below).

The build is **deterministic**: the same input + profile + environment
produces byte-identical output, which is what makes the proof pack's
SHA-256 digests meaningful.

## Modes

Boris has several top-level modes, all invoked as `boris <command>`:

| Mode | What it does |
|------|--------------|
| `build` (default) | Compile the HTML site + proof pack. |
| `validate` | Validate selected HTML source/config without publication. |
| `watch` | Build, then rebuild on content/theme change. |
| `check` | Read-only graph health report (findings don't fail by default). |
| `impact <ID>` | Read-only transitive impact report for a page. |
| `plan` | Emit the normalized publication plan (no publication). |
| `standard-site *` | The ATProto family — see [[standard-site]]. |
| `nostr *` | The Nostr NIP-23 publication family (plan/sign/publish). |
| `init [DIR]` | Write a starter site (content, theme, profile) into DIR. |

The `standard-site` subfamily is the ATProto surface: `plan`, `records`,
`verify`, `login`, `sessions`, `logout`, `publish`, `smoke`. These are
covered in [[standard-site]] and [[guides/standard-site-tests]].

## The proof pack

Every HTML build writes `dist/_boris/proof/`. This is the heart of
Boris's integrity model — it pins what was built, what was checked, and
what can be claimed as a result.

The pack has four artifacts plus an `index.html` presentation:

- **`artifacts.json`** — the inventory. Every committed file in the
  target gets a `path`, `kind` (`html-page`, `theme-asset`,
  `rendered-search`, `sitemap`, `rss`, `llms`), `bytes`, and `sha256`.
- **`checks.json`** — the checks that ran, their status, coverage, and
  findings. The three core checks are `artifact-integrity` (every
  artifact's bytes+sha256 match), `rendered-html` (structural/route/
  fragment/duplicate-ID audit of HTML pages), and `rendered-search`
  (the search index matches the committed HTML page set).
- **`claims.json`** — the verifiable claims, each grounded in a check's
  evidence and explicitly limited by a set of limitations. A claim is
  `verified` only when its supporting check passed with complete
  coverage.
- **`touches.json`** — the full relationship graph (nodes + edges) that
  connects targets → artifacts → checks → claims → limitations.

The presentation at `_boris/proof/index.html` renders the pack as a
human-readable report with status banners and tables.

### What the claims mean — and don't

A verified claim is deliberately scoped. The limitations are part of
the claim, not a footnote:

- **`target-local-only`** — the claim describes one local target after
  commit; it says nothing about other targets or environments.
- **`no-deployment-verification`** — local generation cannot verify
  deployed behavior.
- **`no-accessibility-verification`** — no a11y audit was performed.
- **`no-prose-quality-verification`** — no writing-quality judgment.
- **`no-universal-reproducibility-claim`** — deterministic bytes on one
  recorded environment are not universal reproducibility.
- **`omitted-projections-not-certified`** — an omitted projection is
  explicitly unverified, not certified.

This is the "conductor rule": what was not inspected remains explicitly
unverified. The proof pack never overclaims.

## IR, RAG, and context exports

The same content can be exported as machine-readable corpora without
re-parsing:

- **IR mode** (`--out <DIR>`, default `.boris`) — JSON intermediate
  representation under the output directory.
- **RAG mode** (`--rag --rag-dir <DIR>`) — working-context packs sized
  to a target byte budget (`--split-size`, default 256 KiB). `--complete`
  exports the entire validated corpus (system + per-page + graph +
  catalog).
- **Context mode** (`--context --context-dir <DIR>`) — a single bounded
  context bundle (byte-capped via `--split-size`).
- **`--llms`** — a deterministic `llms.txt` export for LLM consumption.
- **`--rss`** — a deterministic RSS 2.0 feed.
- **`--sitemap`** — a deterministic `sitemap.xml` in the HTML target.
- **`--scope VALUE`** — restrict RAG/context to an entity id or
  collection prefix.

The standalone `boris-source-rag` tool (in the kit) packs project
**source files** for LLM upload — a different concern from the content
RAG above. It emits per-file documents, combined bundles, and a catalog.

## The agent kit

The `boris-agent-kit` directory is a transport artifact: native binaries
built from `drawmeanelephant/boris` at a recorded commit, verified with
`SHA256SUMS` against `MANIFEST.json`. The kit includes the `boris`
compiler plus ten standalone tools:

`boris-content-audit`, `boris-docs-maintenance`,
`boris-github-pages-audit`, `boris-migration-lab`,
`boris-scale-smoke`, `boris-search-index`, `boris-source-rag`,
`boris-testdata`, `boris-package`, and `boris`.

Verify before use:

```text
cd boris-agent-kit && shasum -a 256 -c SHA256SUMS
```

This site was built with the kit at commit
`ef23e37785ced4405191c233da216168aea8ff55` (`main`, Darwin-arm64,
Zig 0.16.0, binary `boris/0.8.1`).

## Boris's design stance

Three commitments shape the tool:

1. **Publication truth is the production URL.** The deployment origin is
   not incidental — it is the anchor that ties identity to a built tree.
   `standard-site.json` records it; `verify` enforces it offline.
2. **Explicit, never implicit.** Nothing is written to the network
   without an explicit command and a stored session. The build and proof
   pack are offline and deterministic; publication is a separate,
   gated, manual act.
3. **Fail closed on drift.** A committed `--plan` makes `publish`
   fail-closed against drift. A `[[wiki-link]]` to a missing page fails
   the build. An omitted projection is explicitly unverified, not
   silently certified.

These are why the proof pack's limitations read the way they do: Boris
does not claim what it did not inspect. The next page, [[atproto]],
explains the network side that the standard-site family publishes to.
