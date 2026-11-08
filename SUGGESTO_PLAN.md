# Markdown writing companion CLI

AKA Suggest-o-matic

## Summary

Add `suggesto.py`: a single-file Python utility with a Textual interface. It watches one Markdown file, generates three conceptual queries through OpenAI, and displays related material from local Hister.

## Implementation

- Use Python 3.11+ with `textual`, `openai`, `httpx`, and `pyperclip`; embed UI styles in the script and document setup in README.md.
- Accept a file path and `--word-step` (250), `--model` (`gpt-6-luna`), `--hister-url` (`http://127.0.0.1:4433/search`), and `--limit` (10).
- Require a Markdown file or directory argument. A file starts watching directly; a directory opens a searchable picker for Markdown files recursively beneath it. Filter paths by all typed words, cap visible matches at 200, and support arrows, Enter, mouse selection, and Escape. Start watching only after selection; canceling the initial picker exits. `b` reopens the picker for the launch directory or file's parent; `f` retains manual path entry.
- Read `OPENAI_API_KEY` and optional `HISTER_ACCESS_TOKEN` from environment variables.
- Poll every second; wait for two seconds of stable content. Handle atomic saves and temporarily missing files.
- Count prose words excluding YAML front matter, fenced code, and link destinations; retain headings and visible link text.
- Search initially at 250 words, then after each net increase of 250 words relative to the last successful search snapshot. Revisions and deletions alone do not trigger searches.
- Send the whole cleaned draft and title to OpenAI. Above 12,000 words, use the first and last 6,000 words and show a trimming notice.
- Use Responses API structured output for three distinct short conceptual queries, with `store=False`; treat draft content as data.
- Use `reasoning.effort=none` for the default `gpt-6-luna` model with a 500-token output budget. Do not send this model-specific setting to other model overrides.
- Keep requests asynchronous, allow one search cycle at a time, coalesce pending triggers, and discard results for previously selected files.
- Use Hister's direct HTTP API: check `/api/config` for authentication and semantic availability before calling OpenAI, then GET `/search` with `q`, `semantic=true`, `format=json`, and the result limit in the JSON-encoded `query` parameter. Parse `history`, `documents`, and `semantic_hits`, preserving similarity and matching passages. Report disabled semantic search; do not silently fall back to keyword-only results.
- Require `--hister-url` to end in `/search` and preserve any deployment path prefix when deriving `/api/config`. Earlier explicit `/mcp` URLs must be updated.
- Fetch ten results per query by default, deduplicate URLs, merge with reciprocal rank fusion (constant 60), and display the top ten with matching queries.
- Exclude the watched local file and its inferred Jekyll URL before merging results. Support repeatable `--exclude-url` for aliases or custom permalinks.
- Show path, count, next trigger, status, queries, selectable results, and a detail pane containing title, URL/path, similarity, and matched queries; omit Score.
- Default to Hister's `matched_chunk` passages, preserving each query's passage and similarity when merging results. Join semantic hits that omit an embedded document to keyword results by document ID. Bound passages to 2,000 characters; label missing-chunk fallback excerpts explicitly. Do not automatically read or fetch full documents on selection.
- On `v`, load a longer preview asynchronously: local Markdown for this site's published posts and local Markdown files, otherwise Hister `/api/document` indexed text. Cap rendering at 20,000 characters and label truncation; Enter opens the original. Toggle back to matches with `v`. Keep matching passages on failure, cache longer previews per search, and cancel/discard stale responses when selection/file changes.
- Provide formatted Markdown and a read-only selectable text tab, removing local front matter. Copy the whole preview with `c`, or selected text with Ctrl+C in the text tab. Delegate system clipboard backend selection to Pyperclip, run copies serially in a background thread, and verify readback before reporting success. Keep Textual's internal clipboard synchronized. Document required Linux system helpers and display/PATH diagnostics; no custom subprocess backend or OSC 52 implementation.
- Keys: `r` manual search, `p` pause/resume, `f` switch file, Enter open result, `q` quit. Manual search bypasses the word threshold.
- Keep metadata literal, disable automatic preview link opening, and open browser/local results only after Enter.
- Preserve previous results during requests/errors. Failures wait for manual retry or another qualifying edit rather than retrying continuously.
- Keep state in memory. Observe saved content only; do not modify or index drafts. Search all material already indexed by Hister.

## Validation

Follow AGENTS.md: do not install dependencies, run code, or execute tests during implementation. Review changes statically and provide manual commands from the project root:

```sh
uv venv --python 3.12 .venv-suggesto
uv pip install --python .venv-suggesto/bin/python textual openai httpx pyperclip
uv run --no-project --python .venv-suggesto/bin/python suggesto.py --help
uv run --no-project --python .venv-suggesto/bin/python suggesto.py test.md --word-step 250
```

With the API key exported and Hister running, expect the TUI to count saved prose and show related results once the threshold is reached. Manually verify threshold crossing, revisions without growth, atomic saves, switching during requests, pause/resume, duplicates, empty results, unavailable Hister, invalid credentials, and context trimming.

## References

- [OpenAI structured outputs](https://developers.openai.com/api/docs/guides/structured-outputs)
- [OpenAI GPT-6 Luna model](https://developers.openai.com/api/docs/models/gpt-6-luna)
- [Hister HTTP API](https://github.com/asciimoo/hister/blob/master/server/api.go)
- [Hister search result types](https://github.com/asciimoo/hister/blob/master/server/indexer/indexer.go)

Draft context goes to OpenAI. Hister retains its existing embedding configuration.
