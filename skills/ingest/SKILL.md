---
description: Ingest a source (file path, URL, YouTube video, or pasted text) into an LLM wiki vault per the vault's CLAUDE.md schema. Produces a schema-conformant source page and updates entities, concepts, indexes, log, and hot cache. Use when the user asks to "ingest <source>", "add this to my wiki", "save this source", or drops a link/transcript/file for wiki processing.
disable-model-invocation: true
---

# Ingest

## Purpose

Take a source in any supported form and integrate it into an LLM wiki per the vault's schema. The schema lives in the vault's `CLAUDE.md`; this skill implements the ingest workflow operationally.

## Prerequisites

- Config resolved (see [config resolution](#config-resolution) below). Required: `vault_path`, `wiki_dir`, `raw_dir`.
- For YouTube inputs: `@jordanchoi/mcp-server-youtube-transcript` MCP connected (declared in this plugin's `.mcp.json`; installs via `npx` on first use).
- Vault's `CLAUDE.md` loaded as the schema source of truth.

## Supported inputs

| Input | Detection | Fetch |
|---|---|---|
| File path under `{vault_path}/{raw_dir}/` | starts with `raw/` or is an absolute path inside the vault | Read tool |
| YouTube URL | matches `youtube.com/watch` or `youtu.be/` | YouTube transcript MCP |
| Other URL | `http(s)://` not matching YouTube | WebFetch |
| Pasted text | no URL, no path — text in the invocation | use as-is |

**Input validation:** if an input doesn't cleanly match any pattern, STOP and ask. Do not guess.

## Config resolution

1. Check `{plugin_root}/config.local.json` — use it if present.
2. Otherwise parse `## LLM Wiki Config` block from vault's `CLAUDE.md` (fenced JSON).
3. Otherwise prompt user for `vault_path`, `wiki_dir`, `raw_dir`, `build_backlog_path`. Save to `config.local.json`.

Validate: `vault_path` must exist. `{vault_path}/{wiki_dir}/` must exist. Abort with a clear message if not.

## Workflow

### Step 0 — Resolve config

Load config per above. Change into the vault mentally; all subsequent paths are relative to `{vault_path}` unless absolute.

### Step 1 — Detect input type and fetch

Per the input table. Specifics:

- **YouTube:** fetch full transcript via MCP. Preserve verbatim — you'll write it under `## Full transcript` on the source page.
- **URL:** use WebFetch. If it fails or returns 4xx/5xx, stop and ask the user for a paste fallback.
- **Pasted text:** treat the text in the invocation as the raw content.
- **File path:** Read the file.

Capture: raw content, source title (from `<title>` / `ytdl` metadata / filename / first line), source date (if detectable), source URL (for transient sources), canonical path (for PARA-filed).

### Step 2 — Idempotency check

Before any writes:

```bash
# Transient sources with URL (YouTube, fetched URLs)
grep -l "source_url: <url>" {vault_path}/{wiki_dir}/sources/*.md

# PARA-filed sources
grep -l "raw_path: <relative-path>" {vault_path}/{wiki_dir}/sources/*.md

# Pasted text (neither source_url nor raw_path)
grep -l 'title: "<title>"' {vault_path}/{wiki_dir}/sources/*.md
```

For pasted text, title-equality is the dedup check. Weaker guarantee than URL/path-based — re-pasting the same content with a slightly different title misses the guard. Use `--force` to deliberately re-ingest.

If match found, prompt:
> "Source already ingested as `<existing-slug>` on `<ingested-date>`. Update existing / create new version / abort?"

- Default: **update existing** (proceed to step 4 against the existing source page).
- `--force` flag in invocation skips this check.

### Step 3 — Read and summarize

Read the full source content; do not skim. Write a summary:

- Typically 3–5 bullets.
- Use fewer for short sources (a tweet), more for dense ones (a 2-hour podcast).
- Goal: the user can confirm you captured the actual contribution.

### Step 4 — Pause for user confirmation

Emit the summary, ending with:

```
Proceed? (yes / edit the summary / skip)
```

DO NOT write any files until the user replies. For multi-source ingests, pause per source unless user says "proceed with all remaining."

On `edit`: apply correction to internal summary, re-emit, pause again.
On `skip`: exit cleanly, no files written.
On `yes`: proceed.

### Step 5 — Create (or update) source page

Path: `{vault_path}/{wiki_dir}/sources/<slug>.md`

`<slug>` = kebab-cased title.

**Slug collision:** if a file with this slug already exists for a DIFFERENT source, append `-2`, `-3`, etc. Update the first file's frontmatter to include `sequels: [<new-slug>]`.

**Frontmatter** (per vault CLAUDE.md source template):

```yaml
---
title: <human title>
type: source
# for PARA-filed sources only:
raw_path: raw/<relative path>
# for transient sources (YouTube, fetched URL, pasted text):
source_url: <URL if any>
source_kind: article | paper | book-chapter | podcast | image | note | transcript | video | fetched-article | pasted-text
author: ...
date: ... # source's own date if known
ingested: YYYY-MM-DD               # creation timestamp for source pages
updated: YYYY-MM-DD                # initialized to `ingested`; bumped on actual modification
tags: [...]
sources: []
---
```

**Body sections** (per vault CLAUDE.md source template):

- `## TL;DR` — the bullets from step 3.
- `## Key points` — structured notes. Enough detail that future Claude can answer questions without re-reading the raw source.
- `## Entities & concepts touched` — bulleted wikilinks.
- `## Quotes / figures worth keeping` — verbatim excerpts with surrounding context.
- `## Connections` — how this source agrees with, extends, or contradicts existing wiki pages.
- `## Full transcript` — YouTube sources ONLY. Paste the verbatim transcript. The URL may rot and transcript services may return different results on future fetches — this is the durable record.
- `## Full content` — Pasted-text sources ONLY (neither `source_url` nor `raw_path`). Preserve the verbatim text. This is the only durable record — no URL to refetch from, no PARA file in `raw/`.

### Step 6 — Walk entities and concepts

For each entity/concept mentioned:

- **Page exists:** update `## Summary`, add a bullet to `## Key facts` citing `[[sources/<slug>]]`, append to `## Source log`, add the source slug to `sources:` frontmatter list, update `updated: YYYY-MM-DD`.
- **Doesn't exist AND substantive:** create new page per the vault CLAUDE.md entity/concept template.
- **Only mentioned in passing:** leave a wikilink in the source page; do NOT spawn a stub page.

**Bidirectional links:** when two or more entities/concepts appear together in a source for the FIRST time and the relationship is substantive (not mere co-occurrence), add cross-links in each page's `## Related` section. **Each Related entry MUST end with the source citation `[[sources/<source-slug>]]`** — the citation is not optional. Format:

`- [[<target-page>]] — one-line justification of the relationship. [[sources/<source-slug>]]`

**Contradictions:** if new info contradicts an existing page, add `## Open questions` or `## Conflicting accounts` with both citations. Do NOT silently overwrite.

### Step 7 — Update sub-indexes

Identify the **lead** domain by consulting the "Domain sub-indexes" paragraph in the vault's CLAUDE.md.

- **Lead sub-index** (`{wiki_dir}/index-{lead}.md`): add a rich summary (2–3 sentences with key facts). If a new entity/concept page was created, add it to the lead sub-index too.
- **Secondary sub-indexes:** for each additional domain this source spans, add a one-line pointer:
  `- [[sources/<slug>]] — see full summary in [[index-{lead}]]`
- **Master index** (`{wiki_dir}/index.md`): only update if the domain directory or recently-active section needs refreshing.

### Step 8 — Append to log

Append to `{wiki_dir}/log.md`:

```
## [YYYY-MM-DD] ingest | <source title>
- created: [[sources/...]], [[entities/...]]
- updated: [[entities/...]], [[concepts/...]]
- notes: <optional one-liner>
```

### Step 9 — Append to hot.md

Append a brief dated note to `{wiki_dir}/hot.md` — **≤75 words** (1–3 sentences typical), with wikilinks to the pages where the full content lives. Detailed reasoning belongs on the source / entity / concept pages, NOT in `hot.md`.

```
### [YYYY-MM-DD] <source title>
<1–3 sentences pointing at [[sources/<slug>]] and 1–2 key entity/concept pages. ≤75 words total.>
```

If the total file exceeds ~500 words after append:

1. **First, compress** entries older than 24 hours to one-line headlines: `### [YYYY-MM-DD] <title> — see [[wikilink]]`.
2. **Only if still over budget**, trim the oldest compressed entries.

Aim to keep ≥4–7 days of activity visible.

### Step 10 — Report

Report:
- **Pages created:** [list]
- **Pages updated:** [list]
- **Skipped stub candidates:** [list — entities/concepts mentioned in passing that didn't warrant a page yet]
- **Notes:** any contradictions surfaced, multi-domain routing, or other edge cases.

A typical ingest touches 5–15 pages. Prefer updates over new pages.

## Tone and style

- Write summaries in the vault's prevailing voice (check a few existing source pages before writing).
- Cite everything. Every claim in entity/concept pages that came from a source gets `[[sources/<slug>]]`.
- Don't silently overwrite contradictions. Keep both views; flag the disagreement.
- Link aggressively. A wiki without cross-references is just a folder of documents.

## Error handling

- **Fetch fails:** stop, report error, ask user for alternate source (paste fallback).
- **Frontmatter parse error in existing page:** stop, report path, ask user to fix manually.
- **Write failure (permissions, disk full):** stop, report, do not partial-write.
- **Config missing or invalid:** prompt user; save corrected config to `config.local.json`.
