---
title: Publishing
parent: index
tags: [guides]
relations: [relates_to=standard-site]
published_at: 2026-08-27T17:30:00Z
summary: The publication profiles — how Boris treats the production URL as publication truth.
---

# Publishing

Boris treats the deployment URL as publication truth, not an incidental
detail. This page covers the publication profiles; the full ATProto
publish loop (plan → verify → login → publish) is in
[[standard-site]], and the ATProto background in [[atproto]].

The starter profile declares one public HTML target:

```json
{
  "format": "boris-publication-profile",
  "schema_version": 1,
  "input": "content",
  "targets": [
    { "name": "public", "output": "dist", "public": true, "theme": "themes/boris" }
  ]
}
```

Inspect the normalized plan before publishing:

```text
boris plan --profile boris.json
boris standard-site plan --profile standard-site.json
```

The official GitHub Pages workflow (see the repository's
`docs/github-pages.md`) builds a verified target: it resolves the Pages
location from `actions/configure-pages`, fails on any URL projection
that disagrees with it, uploads only inventory-verified files, and
retains a separate evidence artifact. Atmosphere publication uses
`standard-site.json`, wired to the real production identity
`did:plc:jiokpoojzqntdpyw5xvfr7rv` (handle `boris.filed.fyi`, origin
`https://boris.filed.fyi/`). The earlier placeholder DID and the
throwaway test identity are both retired. This page is
related to [[standard-site]] so the semantic graph has an edge to inspect.
