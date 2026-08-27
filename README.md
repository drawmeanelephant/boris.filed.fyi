# boris.filed.fyi

A technical overview site for **Boris** (the Zig content compiler) and **ATProto**,
built with Boris itself and published to the AT Protocol via the `standard.site`
lexicon.

## What this is

Every page is compiled by Boris; the site projects onto `site.standard.publication`
and `site.standard.document` records on the AT Protocol; and the whole thing is
served as static HTML from Cloudflare Pages at
[boris.filed.fyi](https://boris.filed.fyi/).

- **Identity:** `did:plc:jiokpoojzqntdpyw5xvfr7rv`
- **PDS:** `https://auriporia.us-west.host.bsky.network`
- **Origin:** `https://boris.filed.fyi/`
- **Publication record:** `at://did:plc:jiokpoojzqntdpyw5xvfr7rv/site.standard.publication/self`

## Repo layout

```text
docs/
  content/      Markdown source (the site pages)
  themes/boris/  the Boris theme (layout + CSS)
  dist/          built HTML + proof pack + well-known + sitemap + llms.txt
  boris.json     GitHub Pages / local publication profile
  standard-site.json   Atmosphere profile (DID + origin)
  evidence/      offline plan/records/verify + live publish/smoke artifacts
```

`dist/` is committed because Boris is a native binary that can't run in the
Cloudflare Pages build environment. The site is pre-built locally and deployed
as-is.

## Build (local)

The build requires the Boris binary (see
[drawmeanelephant/boris](https://github.com/drawmeanelephant/boris)):

```text
boris build --input docs/content --html-dir docs/dist --theme docs/themes/boris \
  --profile docs/standard-site.json --sitemap --site-url https://boris.filed.fyi/
boris --input docs/content --llms --llms-path docs/dist/llms.txt
```

## Deploy

Cloudflare Pages serves `docs/dist/` directly (no build command, since the
output is pre-built). Push to `main` and Cloudflare deploys.
