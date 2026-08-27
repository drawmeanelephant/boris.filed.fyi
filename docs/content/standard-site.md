---
title: Standard.site — the Boris/ATProto loop
parent: index
tags: [guide, atproto, meta]
relations: [relates_to=boris, relates_to=atproto]
published_at: 2026-08-27T17:30:00Z
summary: The operator's manual for the build, plan, verify, login, publish, and smoke loop that turns a Boris-built site into an on-network ATProto publication.
---

# Standard.site — the Boris/ATProto loop

This is the meta guide. It documents how to take a Boris-built site and
project, verify, authorize, and publish it to the AT Protocol via the
`boris standard-site` family — and the site you are reading is itself
produced by that loop. Every command below is run against this site.

The two references this page ties together: [[boris]] (the compiler and
proof pack) and [[atproto]] (the network pieces). The procedures are in
[[guides/standard-site-tests]]; the run log in [[guides/test-runs]].

## The loop

```text
            ┌─────────────────────────────────────────────────────┐
            │  content/  ──boris build──▶  dist/  +  _boris/proof │
            │                          (offline, deterministic)   │
            └─────────────────────────────────────────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
       standard-site plan    standard-site verify    standard-site records
        (offline project)    (offline cross-check)   (offline payloads)
              │                     │
              └──────────┬──────────┘
                         ▼
                 standard-site login      (live, session stored 0600)
                         │
                         ▼
                 standard-site publish   (live, explicit, plan-guarded)
                         │
                         ▼
                 standard-site smoke     (live, manual, bounded, no residue)
```

The first row (build, plan, verify, records) is **offline, deterministic,
and CI-safe** — no network, no credentials. The second row (login,
publish, smoke) is **live, explicit, and session-gated** — nothing is
written to the network without an explicit command.

## The profile

`standard-site.json` is the anchor. It records the identity and origin
that tie a built tree to the network:

```json
{
  "format": "boris-publication-profile",
  "publication": {
    "target": "standard-site",
    "base_url": "https://boris.filed.fyi/",
    "origin": "https://boris.filed.fyi/",
    "did": "did:plc:jiokpoojzqntdpyw5xvfr7rv",
    "name": "Boris on ATProto",
    "show_in_discover": false,
    "prune": false
  },
  "targets": [
    { "name": "public", "output": "dist", "public": true, "theme": "themes/boris" }
  ]
}
```

- `did` — the owning identity. This site is wired to the real
  identity `did:plc:jiokpoojzqntdpyw5xvfr7rv` (handle `boris.filed.fyi`,
  PDS `https://auriporia.us-west.host.bsky.network`).
- `base_url` / `origin` — the production URL. **This is publication
  truth.** `https://boris.filed.fyi/` — the offline steps run against
  it, and `publish` writes the real record to the PDS.
- `show_in_discover` — whether the publication advertises itself to
  discovery surfaces.
- `prune` — whether publish may delete stale documents. ANDs with the
  explicit `--prune` flag; never implicit.

The old placeholder DID (`did:plc:aaaa…`) and the throwaway test
identity (`did:plc:xrqjadveiamk7vq7sfnvfddz`) are both retired from
this site's config. The live session for the new identity is stored
and `publish`-ready.

## Step 1 — build (offline)

The build is two commands: the HTML build (with profile, sitemap) and
the standalone `llms.txt` export:

```text
boris build --input content --html-dir dist --theme themes/boris \
  --profile standard-site.json --sitemap --site-url https://boris.filed.fyi/
boris --input content --llms --llms-path dist/llms.txt
```

The first command produces the HTML target, the proof pack under
`dist/_boris/proof/`, the `.well-known/site.standard.publication`
verification file, and `dist/sitemap.xml`. The `.well-known` file
contains the publication record's AT-URI
(`at://did:plc:jiokpoojzqntdpyw5xvfr7rv/site.standard.publication/self`)
so a reader who fetches
`https://boris.filed.fyi/.well-known/site.standard.publication` can
confirm the DID points at this origin. The sitemap lists all 9 pages
under `https://boris.filed.fyi/`. The second command exports `llms.txt`,
a deterministic LLM-consumable index of every page with its title and
summary. See [[boris]] for what the pack guarantees. This is the step
you repeat on every content change.

### Dated documents project as records

A page projects as a `site.standard.document` record only when its
frontmatter carries a `published_at` date (full UTC:
`YYYY-MM-DDTHH:MM:SSZ`) and a non-empty `summary`. Without both, the
page appears in the plan's `exclusions` with reason `missing-date`.
Every page on this site now carries both, so all 9 project as document
records — 10 records total (1 publication + 9 documents).

## Step 2 — plan (offline)

```text
boris standard-site plan --profile standard-site.json --out evidence/plan.json
```

Emits a deterministic projection of the records that *would* be
published: the `site.standard.publication` record (rkey `self`), the
9 `site.standard.document` records (one per dated page), the
`.well-known/site.standard.publication` verification file path, the
document exclusions (now empty — every page is dated), and the
verification surface projection.

No network, no credentials. The output is byte-stable across runs with
identical input + profile. This is what you commit and diff against.

## Step 3 — records (offline)

```text
boris standard-site records --profile standard-site.json --out evidence/records.json
```

Dumps the full canonical record payloads — the exact JSON that would be
written to the PDS. For a fresh site this is just the `self` publication
record (`{"url":..., "name":...}`); once pages carry `published_at`
dates, it includes the `site.standard.document` records too.

## Step 4 — verify (offline)

```text
boris standard-site verify --profile standard-site.json --dist dist --out evidence/verify.json
```

Cross-checks the **built** tree against the **profile projection**:
the head links in the HTML and the `.well-known/site.standard.publication`
project file. Fails on drift.

Before a real origin and dated documents exist, `verify` correctly
reports `overall_passed: false` (well-known `missing`, `documents`
empty). This is the fail-closed gate doing its job, not a tool failure —
see the insight in [[guides/test-runs]]. T-04 passes once the production
origin is configured and documents carry dates.

## Step 5 — login (live)

```text
# OAuth + DPoP path (granular scopes; needs a compliant PDS)
boris standard-site login --did did:plc:jiokpoojzqntdpyw5xvfr7rv

# app-password path (broad write; works on bsky.social; secret via stdin)
boris standard-site login --app-password --handle boris.filed.fyi
```

Persists a session to a `0600` store. `sessions` lists DID/flavor/PDS
with no secrets. The app-password is read **once from stdin only** —
never argv, profile, or environment.

**Finding:** on `bsky.social`, OAuth + DPoP writes fail with
`InvalidNonce`; the app-password path is the supported live target.
See [[atproto]] and [[guides/test-runs]].

```text
boris standard-site sessions
boris standard-site logout --handle <test-handle>
```

## Step 6 — publish (live, explicit)

```text
boris standard-site publish --profile standard-site.json --out evidence.json
```

One-shot publish from the stored session. Reconciles the live repository
state, rotates tokens (rotate-or-die on refresh failure), and writes the
projected records. With a committed `--plan PATH`, it is **fail-closed**
on drift between the profile and the committed plan.

Pruning only occurs with explicit `--prune` authority (ANDs with the
profile `prune` flag). Nothing is deleted implicitly.

## Step 7 — smoke (live, manual, never in CI)

```text
boris standard-site smoke --did did:plc:jiokpoojzqntdpyw5xvfr7rv \
  --namespace boris-smoke-test --out evidence/live-smoke.json
```

A bounded, manual interop check — **never run in CI**. It creates two
records under unique `boris-smoke-<…>` rkeys, readback-verifies them
against the PDS (AT-URI + value + CID), then deletes them, leaving no
residue. Optional `--surface-url` checks the served origin;
`--indexer` observes a non-normative AppView.

This site's last smoke (2026-08-27, app-password path, against the
earlier throwaway identity) **passed** across discovery, authorization,
write, readback, and cleanup. See `evidence/live-smoke.json` and
[[guides/test-runs]]. A fresh smoke against the live `boris.filed.fyi`
identity is the next planned run.

## The meta point

This page is part of the site it documents. The Markdown source lives at
`content/standard-site.md`; `boris build` compiles it into the HTML you
are reading and includes it in the proof pack's artifact inventory;
`standard-site plan` would project it as a `site.standard.document`
record (once it carries a `published_at` date); `standard-site verify`
would cross-check its head links offline; and `standard-site publish`
would write it to the PDS, gated by the stored session and the
committed-plan guard.

Every command in this loop is reproducible from the kit. Start with
[[guides/getting-started]] to add a page, [[guides/publishing]] for the
profile details, and [[guides/standard-site-tests]] for the per-surface
procedures.
