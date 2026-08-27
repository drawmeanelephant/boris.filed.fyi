---
title: Operating this site
parent: index
tags: [guide, ops, github, cloudflare]
relations: [relates_to=standard-site, relates_to=boris]
published_at: 2026-08-27T18:45:00Z
summary: The ops playbook — how this site is built, deployed, and published to ATProto, including the GitHub branch protection, the Cloudflare deploy pipeline, and the BORIS_APP_PASSWORD secret convention.
---

# Operating this site

This page is the ops playbook: how `boris.filed.fyi` is built, deployed,
and published to the AT Protocol. It is the *this-repo-specific* guide —
the general Boris concepts live in [[boris]], [[atproto]], and
[[standard-site]]. The upstream project documentation at
`github.com/drawmeanelephant/boris` (GitHub Pages) remains the canonical
source for the tool itself; this site is a complementary help area, not a
replacement.

## The two-layer model

This site lives in **two places** that are easy to conflate, and they do
different jobs:

1. **The hosting layer — Cloudflare Pages.** Serves the static HTML at
   `https://boris.filed.fyi/`. The project is `boris-product`, and its
   default pages.dev URL (`boris-product.pages.dev`) points at the same
   deployment as the custom domain.
2. **The ATProto layer — your PDS.** Holds the records that *point at*
   the hosted site: `site.standard.publication/self` says "this DID owns
   the site at this URL", and the `site.standard.document` records point
   at individual pages.

`standard.site` is **not a web host.** It's a lexicon — a record type on
your PDS. The HTML still needs a host; the records make the hosted site
verifiable and portable on the network. Cloudflare serves the files; the
PDS serves the pointers. Both are required; neither replaces the other.

The bridge between them is the `.well-known/site.standard.publication`
file: a reader resolves the DID → gets the URL → fetches the well-known
file → confirms the site claims that DID. The loop is closed when the
deployed file matches the PDS record (see [[standard-site]]).

## Deploy pipeline

The site deploys automatically from GitHub. The flow:

```text
push/merge to main
  → GitHub Action "Deploy to Cloudflare Pages"
  → wrangler pages deploy docs/dist --project-name=boris-product
  → live at boris.filed.fyi
```

Key facts:

- **`dist/` is committed.** Boris is a native binary that can't run on a
  Cloudflare or GitHub Actions runner (it's Darwin-arm64 and lives in the
  gitignored `boris-agent-kit/`), so the built output is committed and
  deployed as-is. Rebuild locally, commit `dist/`, push.
- The workflow runs on **both** push to main (production deploy) and
  pull requests (preview deploy) — the PR run is what satisfies the
  required status check.
- The deploy workflow is the required check on `main` — PRs can't merge
  without it passing.

The two-command build:

```text
boris build --input docs/content --html-dir docs/dist --theme docs/themes/boris \
  --profile docs/standard-site.json --sitemap --site-url https://boris.filed.fyi/
boris --input docs/content --llms --llms-path docs/dist/llms.txt
```

## Branch protection on main

`main` is protected so agents can work safely:

- **Pull requests required** — no direct pushes to main.
- **0 required approvals** — PRs merge when the deploy check passes
  (auto-merge friendly; no human gate for agents).
- **Required status check** — the `deploy` job must pass.
- **Strict up-to-date** — PRs must be rebased on latest main.
- **No force pushes, no deletions** — enforced on admins too.
- **Dismiss stale reviews** when new commits land.

Repo-level conveniences: **auto-merge** is on (PRs merge themselves once
green) and **auto-delete head branches** is on (merged PR branches vanish
automatically).

## The secret convention

The repo has three secrets:

| Secret | Used for |
|--------|----------|
| `CLOUDFLARE_API_TOKEN` | Deploying to Cloudflare Pages |
| `CLOUDFLARE_ACCOUNT_ID` | Deploying to Cloudflare Pages |
| `BORIS_APP_PASSWORD` | Manual ATProto publish (stdin only) |

The ATProto app-password is a **broad-write** credential. It is stored
only as a GitHub secret (or the local session store) — never in a file in
the repo, never in shell history. Boris accepts it **only via stdin**:

```text
printf '%s' "$BORIS_APP_PASSWORD" | boris standard-site login --app-password --handle boris.filed.fyi
```

If it's ever exposed (e.g. pasted in a chat), revoke it in the provider's
app-password settings and mint a new one.

## The publish routine

The ATProto records are **not** deployed by CI. They are published
manually, and only when the site *structure* changes — adding/renaming
pages, changing the origin, or updating a title/summary. Content edits
redeploy to Cloudflare automatically; the records don't move for those.

```text
# login (once per session; password via stdin)
printf '%s' "$BORIS_APP_PASSWORD" | boris standard-site login --app-password --handle boris.filed.fyi

# publish (only when site structure changed)
boris standard-site publish --profile docs/standard-site.json --plan docs/evidence/plan.json
```

The result lands in `docs/evidence/publish*.json` and should be
committed so the record of what was published stays in the repo. This
manual-explicit design is deliberate — see the "explicit, never
implicit" stance in [[boris]].

## The agent workflow

The whole pipeline is built for agent-driven changes:

```text
agent: branch → commit → push → open PR (auto-merge enabled)
  → deploy check runs (preview deploy)
  → check passes → PR auto-merges
  → production deploy fires → head branch auto-deletes
```

No force-pushes, no deleted branches, no direct-to-main commits — but
agents can ship without waiting on a human review. The evidence trail
(publish artifacts, proof pack, verify results) stays in the repo so
every change is auditable.
