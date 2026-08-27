---
title: Getting Started
parent: index
tags: [guides]
relations: [relates_to=boris]
published_at: 2026-08-27T17:30:00Z
summary: How to author a Boris page — one Markdown file with YAML frontmatter that shapes the frozen graph.
---

# Getting Started

A page is one Markdown file with YAML frontmatter. The frontmatter
`id` is derived from the path unless you write one; `title`, `tags`,
and `parent` shape the graph. For the big picture of what Boris is,
see [[boris]]; for the ATProto publication loop, see
[[standard-site]].

## Add a page

Create `content/guides/example.md`:

```markdown
---
title: Example
parent: index
tags: [guides]
---

# Example

Hello from [[index]].
```

Rebuild with `boris --input content --html-dir dist --theme themes/boris`
and the page appears in the nav forest, the breadcrumb chain, and the
frozen graph.

## Frontmatter at a glance

- `title` — page title (`{{title}}`, search, and metadata).
- `parent` — entity id of the structural parent (this page lives under
  `index`).
- `tags` — free-form list rendered into page metadata.
- `relations` — semantic edges such as `[relates_to=target]`; see
  [semantic relations](https://github.com/drawmeanelephant/boris/blob/afterparty/docs/contracts/semantic-relations.md).

A wiki link `[[getting-started]]` is a real graph edge: a link to a
missing page fails the build instead of rendering as dead prose.
