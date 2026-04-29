---
description: Answer a question about content in the LLM wiki using a cheap retrieval protocol (hot cache → index → sub-index → pages → grep). Use ONLY for questions about wiki content — entities, concepts, sources, prior decisions, or synthesis in the vault. Do not use for general questions unrelated to the wiki.
---

# Query

## Purpose

Answer user questions about the LLM wiki by reading the cheapest relevant pages first, not by crawling the full vault. Follows the vault's CLAUDE.md retrieval protocol.

## Prerequisites

- Config resolved (need at least `vault_path` and `wiki_dir`).
- Vault `CLAUDE.md` loaded as schema reference.

## Retrieval protocol

Stop as soon as you can answer confidently.

```
1. hot.md           → resolves most recent-context queries         (~500 tokens)
2. index.md         → master routing: identify the domain          (~30 lines)
3. index-{domain}.md → rich summaries; ANSWER HERE if possible     (1 read)
4. Specific pages   → deep dives only                              (cap: 5 reads)
5. Grep fallback    → if nothing is indexed
```

### Steps

1. **Read `{wiki_dir}/hot.md`.** If the answer is in recent context, answer immediately.
2. **Read `{wiki_dir}/index.md`** to identify which domain(s) the question touches.
3. **Read the relevant `{wiki_dir}/index-{domain}.md` sub-index.** Rich summaries often contain enough detail (specific numbers, dates, metrics) to answer factual queries **without opening individual pages**. If the sub-index summary answers, stop and go to step 7.
4. **Follow links to specific pages.** Cap: 5 page reads beyond the sub-index. If you need more than 5, the sub-index summaries need enriching — flag this for the next maintenance pass.
5. **If nothing is indexed, Grep as fallback.**
6. **If the answer requires a raw source directly** (e.g. a specific quote), follow `raw_path` or `source_url` from the source page.
7. **Answer with citations** — wikilinks to source / entity / concept pages.
8. **Filing rule:** if the answer required reading ≥3 wiki pages to synthesize, ask the user whether to file it as a new `{wiki_dir}/synthesis/<slug>.md`. Answers drawn from a single page are retrieval, not synthesis — don't offer to file those. When offering, mention how many pages were combined.
9. **Append to `{wiki_dir}/log.md`:**

```
## [YYYY-MM-DD] query | <short question>
- pages: [[...]], [[...]]
- filed: [[synthesis/...]]     # if the answer was saved
```

10. **Append to `{wiki_dir}/hot.md`** — ≤75-word entry pointing at the answered question and the pages consulted. Apply compress-before-trim if window exceeds ~500 words (compress entries older than 24 hours to one-line headlines: `### [YYYY-MM-DD] <title> — see [[wikilink]]`; trim only if still over budget).

## When the wiki doesn't have the answer

- If hot → index → sub-index → 5 pages → grep all turn up empty: say so. Do not fabricate.
- Offer to ingest relevant external material (web search, new source) if the user wants — but do NOT fetch by default.

## Memory discipline

- A memory that summarizes repo state is frozen in time. If the user asks about recent or current state, prefer reading `wiki/log.md` over recalling older context.
- Before answering definitively, verify the current state of the wiki — don't rely solely on what used to be true.
