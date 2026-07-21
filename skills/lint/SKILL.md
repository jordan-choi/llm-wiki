---
description: Run a health check on the LLM wiki — find contradictions, stale claims, orphans, missing entity/concept pages, out-of-sync metadata, raw/wiki drift, schema-conformance issues, hot-cache hygiene. Optionally scope to specific files, globs, or a recent ingest's touch surface. Reports findings by default; with `--fix`, applies safe automatic corrections. Use when the user asks to "lint the wiki", "health check", "audit the wiki", "lint these files", "lint what I just ingested", or similar.
disable-model-invocation: true
---

# Lint

## Purpose

Scan the wiki for schema violations and quality issues. Produce a report; leave fixes to the user (or explicit follow-up runs). Follows the vault's CLAUDE.md lint workflow.

## Invocation

```
/llm-wiki:lint                                       # full-wiki scan, report only
/llm-wiki:lint <glob-or-paths>                       # explicit-file scope, report only
/llm-wiki:lint --ingest [<slug>]                     # ingest scope, report only
/llm-wiki:lint [<scope>] --fix                       # report + apply safe fixes
/llm-wiki:lint [<scope>] --fix --dry-run             # preview fixes; do not write
```

Where `<scope>` is one of: `<glob-or-paths>`, `--ingest [<slug>]`, or omitted (full-wiki).

**Three scope modes** × **three action modes** (report / fix / preview) = 9 useful combinations.

Examples:

```
/llm-wiki:lint --ingest --fix                            # post-ingest follow-up — lints + fixes the touch surface
/llm-wiki:lint --ingest build-agent-teams-in-claude-cowork --fix --dry-run
                                                          # preview fixes for a specific ingest
/llm-wiki:lint wiki/entities/*.md --fix                  # apply safe fixes across entity pages
/llm-wiki:lint wiki/entities/claude-cowork.md            # single hand-edited file
/llm-wiki:lint wiki/sources/                             # directory glob
/llm-wiki:lint --fix                                     # full-wiki cleanup pass
/llm-wiki:lint --fix --dry-run                           # preview a full-wiki fix run
```

**Argument precedence:** `--ingest` and explicit `<files>` are mutually exclusive. If both are given, error with: `--ingest cannot be combined with explicit file scope; pick one.`

When scoped (either explicit-file or `--ingest`), only checks that make sense per-file are run; full-wiki checks (orphans, stale claims, missing-entity counts, data gaps) are skipped with a notice in the report.

## Ingest scope resolution (`--ingest [<slug>]`)

When invoked with `--ingest`, lint resolves scope by reading `{wiki_dir}/log.md`:

- **No slug** (`--ingest`): find the most recent `## [YYYY-MM-DD] ingest |` entry. Use it.
- **Slug given** (`--ingest <slug>`): find the most recent `ingest` entry whose source slug appears in its `created:` list (typically `[[sources/<slug>]]`). Use it.

From the matched entry, parse the `created:` and `updated:` wikilink lists. Resolve each `[[<page>]]` to its file path (e.g., `[[entities/foo]]` → `{vault_path}/{wiki_dir}/entities/foo.md`). Lint that set of files.

**Errors:**
- **No matching entry:** stop with `No ingest log entry found for slug '<slug>'. Run /llm-wiki:lint --ingest (no slug) for the most recent ingest, or check {wiki_dir}/log.md`.
- **Listed page no longer exists on disk:** include in the report as a finding (`Ingest log references missing page: <path>`) but continue linting the remaining files.

**What gets included in the scope:**
- Source page from `created:`
- New entity/concept pages from `created:`
- Updated entity/concept pages from `updated:`
- Sub-index pages from `updated:` (e.g., `index-ai.md`, `index.md`)
- `log.md` and `hot.md` are NOT included — they're append-only and not subject to schema-conformance checks.

## Prerequisites

- Config resolved (need `vault_path`, `wiki_dir`, `raw_dir`).
- Vault `CLAUDE.md` loaded as schema reference.

## Checks

Each check is marked **[F]** (full-wiki only — needs cross-page state) or **[P]** (per-file — runs in scoped mode too).

1. **[F] Contradictions** — pages saying conflicting things about the same entity/concept. Surface both claims and their source citations.
2. **[F] Stale claims** — superseded by newer sources per `log.md` chronology.
3. **[F] Orphan pages** — entities, concepts, or synthesis pages with no inbound wikilinks.
4. **[F] Missing entity/concept pages** — mentioned in ≥2 sources but no dedicated page exists.
5. **[P] Out-of-sync metadata** — `updated:` dates or `sources:` lists in frontmatter don't match the page body.
6. **[P] Unresolved markers** — `TODO` / `??` / `FIXME` left behind.
7. **[F] Data gaps** — targeted web search could fill a known unknown. Flag only; do not fetch unless user explicitly asks.
8. **[P] Dead `raw_path:` targets** — source pages referencing a raw file that no longer exists on disk.
   - Default proposal: append a dated `## Update YYYY-MM-DD` section to the wiki source page documenting the deletion. Rewrite current-truth pages (entities/concepts/synthesis) referencing the raw state to past tense.
   - Flag as "possible mistaken ingest — user to confirm" ONLY if the source page is additionally thin-bodied AND has no inbound wikilinks.
   - **Never delete** a source page without explicit user confirmation.
9. **[P] Schema conformance — Related-link citations** — every entry in `## Related` on entity/concept/synthesis pages must end with `[[sources/<source-slug>]]`. Surface entries missing the citation. (Enforces the bidirectional-links amendment in CLAUDE.md.)
10. **[P] Schema conformance — required body sections** — entity/concept/synthesis pages should include the sections defined by their template in CLAUDE.md (`## Summary`, `## Key facts`, `## Related`, `## Open questions`, `## Source log` for entities/concepts; equivalents for synthesis). Flag missing sections — **but** a section a page *genuinely has nothing for* (e.g. `## Open questions` with no open unknowns) may be omitted per CLAUDE.md's **Empty sections** rule; report such an omission as a **soft/informational** flag, not a violation, and never auto-insert an empty header (an empty section is a stub).
11. **[P] Schema conformance — required frontmatter** — requirements vary by page type:
    - **Sources** (`type: source`): `title`, `type`, `source_kind`, `ingested`, `updated`. Either `raw_path` OR `source_url` (or NEITHER for pasted-text). Use `ingested` as the creation timestamp; do NOT require `created` on source pages.
    - **Entities / concepts / synthesis**: `title`, `type`, `created`, `updated`, `sources` list (may be empty).
    - **Indexes** (`type: index` or filename starts with `index`): only `type: index` required. `created` / `updated` / `tags` are optional — index pages are navigation, not content.
    Flag missing fields per the page type.
12. **[F] Hot cache hygiene** — `{wiki_dir}/hot.md` must stay under ~500 words AND have entries older than 24 hours compressed to one-line headlines. Flag any of: (a) total word count > 550 (10% over budget); (b) any single entry exceeds 75 words; (c) any entry older than 24 hours has more than one line of content. Runs only when scope is full-wiki or includes `hot.md` directly.

## Output

Save the report as `{wiki_dir}/synthesis/lint-YYYY-MM-DD.md` using the vault's synthesis template:

```yaml
---
title: Wiki lint report — YYYY-MM-DD
type: synthesis
tags: [lint, maintenance]
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources: []
status: draft
---

## Question / thesis
Wiki health check — YYYY-MM-DD

## Answer
Summary: N findings across M categories. Priority: [high/med/low split].

## Evidence

### Contradictions
- [[page-a]] vs [[page-b]] — disagree on <point>. Cited from [[sources/x]] and [[sources/y]].

### Stale claims
- ...

### Orphan pages
- [[...]]

### Missing entity/concept pages
- `<name>` — mentioned in [[sources/a]], [[sources/b]]; no page.

### Out-of-sync metadata
- ...

### Unresolved markers
- ...

### Data gaps (flagged, not fetched)
- ...

### Dead raw_path targets
- [[sources/x]] → `raw/...` no longer on disk. Proposed: append-Update.

### Schema conformance — Related-link citations
- [[entities/foo]] line N — Related entry to [[entities/bar]] missing `[[sources/<slug>]]` citation.

### Schema conformance — required body sections
- [[entities/foo]] — missing `## Open questions` section.

### Schema conformance — required frontmatter
- [[sources/foo]] — missing `source_kind` field.

### Hot cache hygiene
- `hot.md` exceeds budget: 612 words (target ~500).
- Entry `2026-04-25 ...` (3 days old) is 4 paragraphs — should be compressed to one-line headline.
- Entry `2026-04-26 ...` is 145 words — exceeds 75-word per-entry cap.

## Counterpoints / tensions
<where findings are uncertain>

## Follow-ups
Suggested fixes, grouped by effort. Do NOT apply without user direction.
```

## Fix mode

By default, lint reports findings without modifying any files. With `--fix`, lint applies **safe** corrections — structural / citation / metadata gaps that are mechanically derivable from existing data. Semantic decisions (which contradiction is right, what content to write, whether to delete) always require explicit user direction and are never auto-fixed.

### What `--fix` will fix

| # | Check | What gets fixed |
|---|---|---|
| 5 | Out-of-sync metadata | Set `updated:` to today; sync `sources:` frontmatter list to body citations |
| 8 | Dead `raw_path:` targets | Append dated `## Update YYYY-MM-DD` section to source page documenting deletion (the existing append-Update flow) |
| 9 | Related-link citations missing | Append `[[sources/<slug>]]` to each Related entry, derived from the page's `sources:` frontmatter — only when source-of-citation is unambiguous (see Ambiguity guard) |
| 11 | Required frontmatter missing | Add defaultable fields only: `tags: []`, `created:` / `updated:` to today. Leave `title`, `type`, `source_kind`, `source_url`, `raw_path` for user — those need human input |
| 12 | Hot cache hygiene | Compress entries older than 24 hours to one-line headlines (`### [YYYY-MM-DD] <title> — see [[wikilink]]`). Trim only if still over budget after compression. Never compress entries < 24h old. |

### What `--fix` will NEVER fix

- **Contradictions** (#1) — needs human judgment on which view is right
- **Stale claims** (#2) — needs rewording or fresh fact-check
- **Orphan pages** (#3) — deletion is irreversible; user must confirm
- **Missing entity/concept pages** (#4) — requires substantive content
- **Unresolved markers** (#6) — TODO content needs resolution
- **Data gaps** (#7) — would require web fetch, separate decision
- **Required body sections missing** (#10) — never auto-insert an empty header; an empty section is a stub, and a genuinely-empty section may be legitimately omitted (CLAUDE.md **Empty sections** rule). Report-only soft flag.

These remain in the report as findings only.

### Dry run

`--fix --dry-run` runs the report AND lists each fix that *would* be applied under a "Would fix" section in the report — but does NOT write any changes. Useful for previewing the citation backfills (#9) and metadata syncs (#5) before committing.

### Ambiguity guard

For check 9 (Related-link citations): if a page has multiple entries in its `sources:` frontmatter and the missing citation can't be unambiguously attributed to one of them, **leave it in the report** — do NOT guess. The fix only applies when there's a single source candidate. Report wording: "Related entry to [[X]] missing citation; multiple sources in frontmatter — manual disambiguation required."

For check 5 (sources frontmatter sync): only add slugs that appear as `[[sources/<slug>]]` citations in the body. Never remove existing entries — removal could erase historical context.

### Hard rules

- **Never auto-delete** any page or content under any circumstance.
- **Never auto-rewrite** existing prose. Fixes only add structural elements (headers, citations, frontmatter fields, append-Update sections).
- **Always log** what was fixed in the report's "Fixed" section, with the page path and the fix applied.

## Post-action

Append to `{wiki_dir}/log.md`:

```
## [YYYY-MM-DD] lint | <scope>[ --fix][ --dry-run]
- findings: N
- fixed: M     # M = number of auto-fixes applied; 0 without --fix or with --dry-run
- report: [[synthesis/lint-YYYY-MM-DD]]
```

Where `<scope>` is one of:
- `full-wiki` — default runs (no scope argument)
- `scoped: <files-or-glob>` — explicit-file scope
- `ingest: <slug>` — `--ingest` runs

**Report filenames:**
- Full-wiki: `lint-YYYY-MM-DD.md`
- Scoped (explicit files): `lint-YYYY-MM-DD-scoped.md`
- Ingest scope: `lint-YYYY-MM-DD-ingest-<slug>.md`
- Dry-run (any scope): append `-dry` before `.md`

If multiple reports of the same shape exist on the same day, suffix with `-2`, `-3`, etc. before `.md`.

Append a brief note to `{wiki_dir}/hot.md` — ≤75 words pointing at the lint report. Apply compress-before-trim if window exceeds ~500 words.

## Archiving applied reports

Once every finding in a prior lint report has been applied or explicitly deferred, move that report to `{wiki_dir}/synthesis/lint-archive/`.

Exception: keep the report in place if it's still cited from a current-truth page (not `log.md` — log is append-only history, not navigation). When moving, update any non-log inbound wikilinks to the new `synthesis/lint-archive/...` path. Log wikilinks to the old path will go stale; that's acceptable.
