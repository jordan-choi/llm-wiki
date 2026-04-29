# llm-wiki

A Claude Code / Cowork plugin that turns an Obsidian-based markdown vault into a disciplined, queryable LLM wiki — with idempotent ingestion, cheap retrieval, and auto-fix maintenance.

## What this is

If you maintain a personal knowledge base in Obsidian (or any markdown vault) with a structured schema — sources, entities, concepts, synthesis pages, sub-indexes per domain, append-only log, rolling hot-cache — this plugin operationalizes the schema's workflow into three slash commands.

- **`/llm-wiki:ingest`** — capture a source (file, URL, YouTube, pasted text) into your wiki with citation discipline and idempotency guards built in.
- **`/llm-wiki:query`** — answer questions about your wiki cheaply via hot cache → index → sub-index → pages. Auto-fires on wiki-related questions.
- **`/llm-wiki:lint`** — health-check the wiki and auto-fix safe categories (citations, frontmatter, hot-cache hygiene).

The plugin is a workflow runtime over your existing schema. The schema is the IP; the plugin reduces friction and prevents drift across many sessions.

## Why this plugin (vs. just using the schema)

If you already have a `CLAUDE.md` defining a wiki schema, you can ask Claude to follow it directly without any plugin. The plugin earns its keep by adding **operational discipline** and **reduced friction** — not new capabilities.

| Capability                         |       Bare CLAUDE.md schema       |               Plugin                |
| ---------------------------------- | :-------------------------------: | :---------------------------------: |
| Schema definition                  |                 ✓                 |                  ✓                  |
| Ingest execution consistency       |      drifts across sessions       |            deterministic            |
| Slash-command invocation           | absent (paragraph per invocation) |         `/llm-wiki:ingest`          |
| Idempotency guard                  |    weak (agent often forgets)     |       grep-based, every time        |
| Bidirectional citation enforcement |          ~70% adherence           |         ~95% on fresh pages         |
| `--ingest` lint scope              |              absent               | one command for post-ingest cleanup |
| `--fix` auto-fixes                 |              absent               |    6 mechanical safe categories     |
| Hot-cache hygiene                  |              manual               |        compress-before-trim         |
| Token cost per invocation          |    higher (re-reads CLAUDE.md)    |                lower                |

**The plugin is justified when** you do ≥3 ingests/week, value citation discipline, and use the wiki across many sessions.

**The plugin is overkill when** you have <50 wiki pages, do <1 ingest/week, or are still experimenting with your schema (which the plugin codifies).

## Requirements

- **A vault with a `CLAUDE.md` schema** defining your wiki conventions — page types, sub-indexes, log/hot.md format. See [Schema reference](#schema-reference) below for the assumed structure.
- **Claude Code or Cowork** with plugin support.
- **YouTube transcript MCP** (optional): `@jordanchoi/mcp-server-youtube-transcript` is declared in this plugin's `.mcp.json` — installs via `npx` on first use. Required only if you want to ingest YouTube URLs.

## Install

```
/plugin marketplace add jordanchoi/jordan-marketplace
/plugin install llm-wiki@jordanchoi
```

## Configure

The plugin needs three parameters. Resolution order:

1. `config.local.json` at plugin root (if it exists)
2. `## LLM Wiki Config` block in your vault's `CLAUDE.md` (fallback)
3. Interactive prompt on first invocation (saves to `config.local.json`)

### Primary: `config.local.json`

```bash
cp config.example.json config.local.json
$EDITOR config.local.json
```

| Parameter    | Meaning                                                                   | Default    |
| ------------ | ------------------------------------------------------------------------- | ---------- |
| `vault_path` | Absolute path to your Obsidian vault root                                 | (required) |
| `wiki_dir`   | Subdirectory for maintained wiki pages                                    | `wiki`     |
| `raw_dir`    | Subdirectory for raw sources (organize however you like; fully immutable) | `raw`      |

### Fallback: vault `CLAUDE.md` block

Add to your vault's CLAUDE.md:

```markdown
## LLM Wiki Config

{
"vault_path": "/abs/path/to/vault",
"wiki_dir": "wiki",
"raw_dir": "raw"
}
```

(Wrap the JSON in a fenced code block in the actual file.)

## Quick start

```
# 1. Ingest a source (YouTube URL, blog URL, local file, or pasted text)
/llm-wiki:ingest <your-source>

# 2. Confirm at the pause — agent shows TL;DR and proposed wiki touches
yes

# 3. Backfill any citation / metadata gaps
/llm-wiki:lint --ingest --fix

# 4. Ask a wiki question in chat (query auto-fires; no slash needed)
What did <source> say about <topic>?

# 5. Periodic maintenance (weekly is plenty)
/llm-wiki:lint --fix
```

## Examples

### Ingest a blog post

```
/llm-wiki:ingest https://blog.example.com/article
```

WebFetch path. Source page uses `source_url:`; no verbatim section (the URL is the durable record).

### Ingest a PARA-filed local file

```
/llm-wiki:ingest raw/Projects/foo/notes.md
```

The raw file is preserved in place — PARA content is immutable per the schema. Source page uses `raw_path:`.

### Ingest pasted text

```
/llm-wiki:ingest <large blob of pasted text>
```

Verbatim text preserved on the source page under `## Full content` (the only durable record — no URL to re-fetch). Idempotency uses title-equality.

### Lint scoped to a specific ingest

```
/llm-wiki:lint --ingest <source-slug> --fix
```

Reads the matching log entry, lints exactly the pages that ingest touched, applies safe fixes.

### Preview what `--fix` would change

```
/llm-wiki:lint --fix --dry-run
```

Same report, no writes. Useful for audit-conscious passes.

## Skill command reference

### `/llm-wiki:ingest`

| Command                            | What it does                                                           |
| ---------------------------------- | ---------------------------------------------------------------------- |
| `/llm-wiki:ingest <input>`         | Standard ingest — auto-detects file path, URL, YouTube, or pasted text |
| `/llm-wiki:ingest <input> --force` | Bypass idempotency guard (re-ingest deliberately)                      |

### `/llm-wiki:query`

Auto-fires on wiki questions. Just ask in chat — no slash command needed.

### `/llm-wiki:lint`

| Command                                  | Scope              | Action              |
| ---------------------------------------- | ------------------ | ------------------- |
| `/llm-wiki:lint`                         | full wiki          | report only         |
| `/llm-wiki:lint --fix`                   | full wiki          | report + safe fixes |
| `/llm-wiki:lint --fix --dry-run`         | full wiki          | preview fixes       |
| `/llm-wiki:lint --ingest`                | most recent ingest | report only         |
| `/llm-wiki:lint --ingest <slug>`         | named ingest       | report only         |
| `/llm-wiki:lint --ingest [<slug>] --fix` | ingest scope       | report + fixes      |
| `/llm-wiki:lint <files-or-glob>`         | explicit           | report only         |
| `/llm-wiki:lint <files-or-glob> --fix`   | explicit           | report + fixes      |

`--ingest` and explicit files are mutually exclusive.

## Schema reference

The plugin assumes this vault layout:

```
your-vault/
├── CLAUDE.md            # schema (you maintain this)
├── raw/                 # source material (organize however; fully immutable)
└── wiki/                # plugin-maintained — REQUIRED structure
    ├── index.md         # master content catalog
    ├── log.md           # append-only activity log
    ├── hot.md           # ~500-word rolling cache
    ├── index-{domain}.md  # one per domain
    ├── sources/         # one page per ingested source
    ├── entities/        # people, orgs, products, places, tools
    ├── concepts/        # topics, ideas, methods
    └── synthesis/       # cross-cutting analyses
```

### What's required vs. user's choice

- **The `wiki/` structure is required.** All four content folders (`sources/`, `entities/`, `concepts/`, `synthesis/`) plus `index.md`, `log.md`, `hot.md`, and per-domain sub-indexes (`index-<domain>.md`) — the plugin writes to all of these.
- **`raw/` is fully immutable, organized your way.** PARA, flat, your own convention — the plugin doesn't care. It reads from `raw/` during ingest but never modifies, renames, or deletes anything inside. Source organization is your responsibility.

If you don't have a `CLAUDE.md` schema yet, build your own using the structure described above as a starting point — the plugin's three skills are self-contained and read schema details from `CLAUDE.md` at runtime. (A `/llm-wiki:write-schema` skill that bootstraps the schema for you is planned for v0.2.)

## Customizing the YouTube transcript MCP

The plugin ships with `@jordanchoi/mcp-server-youtube-transcript` declared in `.mcp.json` — installs via `npx` on first use. If you want to use a different MCP server (your own fork, a different transcript service, or a self-hosted version):

**Option 1 — Override at user scope (cleanest).** Add your MCP at user scope; if it's named `youtube-transcript` and exposes a `get_transcript` tool, the plugin uses it transparently:

```bash
claude mcp add youtube-transcript --scope user -- <your-command-and-args>
```

**Option 2 — Edit `.mcp.json` after install.** Direct override. Replace the bundled package with yours. As long as the server name (`youtube-transcript`) and tool name (`get_transcript`) match, the ingest skill keeps working. Note: plugin updates may overwrite your edit.

**Option 3 — Fork the plugin.** For deep customization (different tool names, different fetch logic). Fork, edit `.mcp.json` + the YouTube fetch reference in `skills/ingest/SKILL.md`, install via `--plugin-dir` against your fork.

If you don't ingest YouTube at all, you can delete `.mcp.json` — the plugin works fine without it; the YouTube input branch will just fail with a clear error if invoked.

## Gotchas

- **Pause at step 2 of ingest** — agent waits for `yes` / `edit the summary` / `skip` before writing anything. If you skip the pause, files don't land.
- **Idempotency guard** — re-ingesting the same URL/path prompts before creating duplicates. Pass `--force` to override.
- **Pasted-text idempotency** uses title-equality — weaker than URL/path-based. Re-pasting same content with a different title misses the guard.
- **Cross-domain sources** — if a source spans multiple domains, ingest picks one as "lead" sub-index and stubs pointers in the rest.
- **YouTube availability** — some videos have no transcript; the MCP fails. Fall back to pasting the transcript manually.
- **No `--replace` in v0.1** — to upgrade a pasted-text source to URL-backed, edit the frontmatter manually. `--replace` ships in v0.2.

## Version

**0.1.0** — initial release. Three skills, four input types, twelve lint checks (six with auto-fix).
