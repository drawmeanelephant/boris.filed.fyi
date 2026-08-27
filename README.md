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
output is pre-built). Push to `main` and the GitHub Action deploys it.

## Publishing to ATProto (manual)

The ATProto records on the PDS are **not** deployed by CI. They are a stable
pointer ("this DID owns the site at this URL"), published manually and only
when the site *structure* changes — adding/renaming pages, changing the
origin, or updating a title/summary. Content edits redeploy to Cloudflare on
their own; the records don't need re-publishing for those.

### The secret convention

If you want the app-password available to a workflow, the convention is a
repo secret — never a file in the repo, never a hardcoded string:

```bash
gh secret set BORIS_APP_PASSWORD
```

Boris accepts the password **only via stdin** (never argv, profile, or env),
so any workflow that publishes must pipe the secret:

```yaml
- run: printf '%s' "$BORIS_APP_PASSWORD" | boris standard-site login --app-password --handle boris.filed.fyi
- run: boris standard-site publish --profile docs/standard-site.json --plan docs/evidence/plan.json
```

For tighter scoping, put the secret in a GitHub *environment* (e.g.
`production`) so only that deploy workflow can read it.

### Why CI publish is manual-dispatch only

There is a publish workflow (`.github/workflows/publish-atproto.yml`), but it
is **manual-only** — it runs from the Actions tab, never automatically. This
matches Boris's design stance: *explicit, never implicit* — live publication
is a gated act, and `smoke` is documented as "never in CI". Content edits
deploy to Cloudflare automatically; the PDS records only move when you
trigger the publish.

The workflow builds Boris from source (`drawmeanelephant/boris`, Zig 0.16.0)
on a macOS-arm64 runner (the kit binary is Darwin-arm64 and gitignored, so CI
can't use it directly), logs in with the `BORIS_APP_PASSWORD` secret via
stdin, publishes, and uploads `docs/evidence/publish-ci.json` as an artifact.

Local alternative (same result, password never leaves your machine):

```text
# login (once per session; password via stdin)
printf '%s' "$BORIS_APP_PASSWORD" | boris standard-site login --app-password --handle boris.filed.fyi

# publish (only when site structure changed)
boris standard-site publish --profile docs/standard-site.json --plan docs/evidence/plan.json
```

The result lands in `docs/evidence/publish*.json` and should be committed so
the record of what was published stays in the repo.
