# Suggest-o-matic 2026

`suggesto.py` is a single-file Python 3.11+ terminal app that follows a saved Markdown draft and finds related material in your existing Hister index. The design is recorded in [MARKDOWN_COMPANION_PLAN.md](MARKDOWN_COMPANION_PLAN.md).

![](screenshot.png)

## Todos

- [ ] create a summary of results, e.g. "The post [post] discusses [X,Y,Z] and is relevant because..." (this will likely require another LLM call)
- [ ] ability to use other LLMs besides OpenAI's
- [ ] ability to determine search queries without using an LLM

## Hister setup

Install Hister using its [installation guide](https://hister.org/docs/installing), then follow the [quickstart](https://hister.org/docs/quickstart) to run the server and index content. Enable semantic search using Hister's [semantic search configuration](https://hister.org/docs/configuration#semantic-search) before running Suggest-o-matic; Hister requires an embeddings endpoint for this feature.

## Starting with `uv`

With [uv](https://docs.astral.sh/uv/pip/environments/) installed, choose where to create the Python environment. To use the original location inside this project:

```sh
COMPANION_VENV=.venv-suggesto
```

Or use the parent directory (recommended when running from inside a content directory):

```sh
COMPANION_VENV=../.venv-<foo>
```

```sh
uv venv --python 3.12 "$COMPANION_VENV"
uv pip install --python "$COMPANION_VENV/bin/python" textual openai httpx pyperclip python-dotenv
```

uv downloads Python 3.12 if it is not already available. The explicit `--python` path targets the chosen environment without needing shell activation. The commands below use `COMPANION_VENV`; set it to your chosen path again when opening a new terminal.

Put your OpenAI API key in a `.env` file beside `suggesto.py`:

```dotenv
OPENAI_API_KEY='your-api-key'
```

The existing `export OPENAI_API_KEY='your-api-key'` method also works. Exported values take precedence over `.env` values. The `.env` file is ignored by Git. Start your existing Hister server with semantic search enabled, then launch the companion:

```sh
uv run --no-project --python "$COMPANION_VENV/bin/python" suggesto.py --help
uv run --no-project --python "$COMPANION_VENV/bin/python" suggesto.py test.md --word-step 250
```

These [uv run](https://docs.astral.sh/uv/reference/cli/#uv-run) commands explicitly use the named environment's Python. `--no-project` skips Python project discovery for this standalone script. No shell activation is required. Expect `--help` to display CLI options and the launch command to open the word counter and search-results TUI.

Replace `test.md` with your draft path. If Hister requires a token, set `HISTER_ACCESS_TOKEN` in `.env` or export it. The app uses `python-dotenv` to read `.env` beside the script. Override the full endpoint with `--hister-url`, the OpenAI model with `--model` (default `gpt-6-luna`), and results per query/displayed with `--limit` (default 10, maximum 50). The endpoint must end in `/search`; an optional deployment prefix is supported (for example `http://localhost:4433/hister/search`).

Omit the file argument to open a searchable picker:

```sh
uv run --no-project --python "$COMPANION_VENV/bin/python" suggesto.py
```

The picker lists `.md` and `.markdown` files recursively from `_drafts` and `_posts` beside the script. Drafts appear first, with filenames sorted in reverse order within each directory so dated files show newest first. Type words from the filename or directory to filter; hyphens and underscores count as spaces, and all search words must match. Use Up/Down and Enter to choose, or click a result. It shows up to 200 matches at once; narrow the search to find older files. Escape cancels (and exits if no file has been selected yet). Press `b` to browse again while watching; `f` still accepts an explicit path outside those directories. No watching or API calls begin until the initial file is selected.

The default model uses `reasoning.effort=none` for this focused query-generation task and a 500-token output cap. Other model overrides use their model defaults; reasoning models may need a larger output budget than this utility provides. See [OpenAI's model documentation](https://developers.openai.com/api/docs/models/gpt-6-luna) for current capabilities and pricing.

The TUI shows the file, prose word count, next automatic search count, queries, ranked results, and a detail pane. After two seconds of stable saved content, it searches initially at 250 words and thereafter at each net increase of 250 words from the last successful search. A large initial draft gets one search, not one for each historical interval. Revisions and deletions alone do not trigger calls. Counts exclude front matter, fenced code, and link destinations using lightweight Markdown cleanup rather than a complete Markdown parser.

The watched post is excluded using its resolved local path and inferred Jekyll URL. URL matching ignores HTTP/HTTPS differences, trailing slashes, query strings, and fragments. Jekyll inference reads the nearest `_config.yml`, the dated post filename, and simple front-matter `date`, `slug`, and `permalink` values; complex YAML, custom permalink placeholders, and alternate domains may need a repeatable `--exclude-url 'https://example.com/post/'` argument. These explicit exclusions remain active when switching files.

Select a result to see the passages that matched each query, with each passage's similarity. Hister supplies these as `semantic_hits[].matched_chunk`: the best matching chunk per document per query. Current [Hister source](https://github.com/asciimoo/hister/blob/master/server/indexer/indexer.go) limits each returned chunk to 500 characters; it does not include a location offset or surrounding-context window in that hit. The app retains passages from all three queries when merging results. It does not fetch or render the entire document automatically, including large external Logseq notes. If no chunk is supplied, it labels the fallback as a search text excerpt rather than a semantic match. Passage previews have a defensive limit of 2,000 characters each for older server versions.

Press `v` to request a longer preview, and again to return to matches. This reads local Markdown (with front matter removed) when available, otherwise Hister's indexed text through `/api/document`. Longer previews are capped at 20,000 characters to keep rendering responsive; the status indicates whether the document is complete or truncated. Enter on the result list opens the original document. A failed load leaves the matched passages visible.

The preview is read-only, fills the right side, and supports mouse selection, Shift+arrow selection, and Ctrl+A. Ctrl+C copies selected text from the preview. Press `c` to copy the selection, or the current preview when nothing is selected. Changing previews clears the previous selection. Clipboard integration is delegated to [Pyperclip](https://github.com/asweigart/pyperclip), which selects the platform backend. Copies run in a background thread and report success only after reading back matching clipboard text. Pyperclip still requires a working system clipboard provider on Linux; installing the Python package alone does not provide one. It uses native Windows and macOS clipboard support as well.

## Linux copy functionality

For Linux copy support (tested on Pop!_OS with GNOME Terminal), install the clipboard helpers from any directory:

```sh
sudo apt install xclip wl-clipboard
```

Install the Python integration into the existing environment from the project root, then restart the companion:

```sh
uv pip install --python "$COMPANION_VENV/bin/python" pyperclip
```

No Zsh or oh-my-zsh changes are required. If copying still fails, run these diagnostics in the **same terminal** used to launch the app (from any directory):

```sh
command -v xclip wl-copy wl-paste
printf 'Session=%s\nDISPLAY=%s\nWAYLAND_DISPLAY=%s\n' "$XDG_SESSION_TYPE" "$DISPLAY" "$WAYLAND_DISPLAY"
```

Expect installed helper paths and at least one nonempty display variable. Missing helper paths indicate a missing package or PATH issue. If both display variables are empty, the app has no desktop display connection; launch it directly from GNOME Terminal in your desktop session. Do not guess a display value. For a manual clipboard check, use either command below according to the available display connection:

The app displays the provider's error type and first message line in both a notification and the status area. `PyperclipException` can indicate a missing clipboard mechanism; `Clipboard contents did not match the requested text` means the copy call returned but reading back the clipboard did not verify the text. Share that exact error alongside the diagnostics above when troubleshooting.

```sh
# X11, or Wayland with XWayland / DISPLAY available:
printf '%s' 'suggestomatic clipboard check' | xclip -selection clipboard -in
# Wayland:
printf '%s' 'suggestomatic clipboard check' | wl-copy --type 'text/plain;charset=utf-8'
```

Paste into a GNOME text editor; expect `suggestomatic clipboard check`. If `wl-copy` fails under GNOME Wayland, try `xclip` through XWayland. To test app copying, select a result and press `c`, then paste into the editor. For selected text, focus the preview, select a passage, and press Ctrl+C. GNOME Terminal's Ctrl+Shift+C shortcut copies the terminal's own selection rather than invoking the app's copy action.

## Keyboard shortcuts

| Key | Action |
| --- | --- |
| `r` | Search the saved draft now, even below the threshold or while paused |
| `p` | Pause/resume automatic searches; an in-flight search may finish |
| `f` | Enter a different Markdown file path; Escape cancels |
| `b` | Browse/search files in `_drafts` and `_posts`; Escape cancels |
| Up/Down | Select a result and show its details |
| Enter | Open the selected web result or local file |
| Tab | Move focus between panes/controls |
| `c` | Copy selected text; with no selection, copy the current preview |
| `v` | Toggle matched passages / request longer document preview |
| Ctrl+A / Ctrl+C in the text preview | Select all / copy selection |
| `q` | Quit (Ctrl+C outside the text preview also exits) |

OpenAI receives the cleaned whole draft and title to generate three conceptual queries; above 12,000 prose words it receives the first and last 6,000 words. The app uses `store=False` for Responses requests. Hister searches all indexed content through its direct HTTP `/search` API with `semantic=true` and `format=json`, using its existing embedding configuration. The app checks `/api/config` for availability, authentication, and enabled semantic search before calling OpenAI. Results include ranked documents and separate semantic hits with similarity and matching passages. Results are merged by reciprocal rank fusion and deduplicated by URL. Nothing is written to the draft or added to Hister's index by this utility.

Previous results remain visible during requests and errors. A failed cycle waits for `r` or a changed draft that meets the word threshold, rather than continuously retrying. Searches require a successful connection to Hister before making an OpenAI request. Switching files discards responses for the previous file. Session state is in memory, so restarting performs a fresh initial search.

---

## Technical walkthrough

### High-level structure

The file is organized into layers:

1. **CLI argument parsing** (lines 20–65)
2. **Import guard** (lines 68–75) — checks for required dependencies before importing them
3. **Core data structures & utilities** (lines 78–300)
4. **Hister client** (lines 302–365)
5. **OpenAI integration** (lines 368–413)
6. **Result merging** (lines 416–455)
7. **Textual UI framework** (lines 458–803)

### 1. CLI Argument Parsing (lines 20–65)

```python
def arguments() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("file", type=Path, help="Markdown file to watch (.md or .markdown)")
    parser.add_argument("--word-step", type=positive_integer, default=250)
    parser.add_argument("--model", default="gpt-6-luna", ...)
    parser.add_argument("--hister-url", default="http://127.0.0.1:4433/search", ...)
    parser.add_argument("--limit", type=positive_integer, default=10, ...)
    parser.add_argument("--exclude-url", action="append", default=[], ...)
    args = parser.parse_args()
```

The app requires:
- A file path (the Markdown draft to watch)
- Optional overrides for word threshold, OpenAI model, Hister endpoint, result limit, and custom URLs to exclude

**Validation happens early:**
- Python 3.11+ check
- File exists and is .md/.markdown
- `--limit` is 1–50
- URL is valid HTTP/HTTPS and ends in `/search`
- `OPENAI_API_KEY` environment variable is set

Parsing happens at the very top (line 68) **before** importing Textual or OpenAI, so `--help` works even if dependencies aren't installed.

### 2. Import Guard (lines 68–75)

```python
if __name__ == "__main__":
    CLI_ARGS = arguments()

try:
    import httpx
    from openai import AsyncOpenAI, APIConnectionError, APIStatusError
    from textual import on
    from textual.app import App, ComposeResult
    # ... more textual imports
except ImportError as exc:
    raise SystemExit(
        "Missing dependency. Run: uv pip install --python "
        ".venv-markdown-companion/bin/python textual openai httpx"
    ) from exc
```

If dependencies are missing, it tells you exactly how to install them.

### 3. Core Utilities & Data (lines 78–300)

#### Markdown parsing helpers

```python
WORD_PATTERN = re.compile(r"[^\W_]+(?:[''‐][^\W_]+)*", re.UNICODE)
CONTEXT_WORD_LIMIT = 12_000
SETTLE_SECONDS = 2.0

def literal(value: object) -> str:
    """Remove terminal controls; never interpret source text as Rich markup."""
```

**`literal()`** — strips Rich/ANSI terminal codes from text so user-supplied content can't break the UI or inject commands.

#### `markdown_body(text: str) -> str`

Removes the YAML front matter (the `---` block at the top). Used when extracting the body for word counting and searching.

#### `scalar(text: str, key: str, default: str = "") -> str`

Parses simple Jekyll YAML metadata without a full YAML parser:
```yaml
date: 2026-10-04
title: My Post
slug: my-slug
```

Extracts `date`, `title`, `slug`, etc. This is used later to reconstruct Jekyll URLs.

#### `url_key(url: str) -> str` and `local_path(url: str) -> Path | None`

Helpers to normalize URLs for deduplication:
- Strip scheme (http/https are the same)
- Strip trailing slashes
- Strip query strings and fragments
- Casefold the domain

Local file URLs are converted to canonical paths.

#### `source_index(watched: Path) -> tuple[dict[str, Path], set[str]]`

**This is important.** It:
1. Reads the repository's `_config.yml` (Jekyll config)
2. Finds all posts in `_posts/` and reconstructs their published URLs
3. Builds a map: `url_key` → local Path
4. Identifies which URL keys should be **excluded** from results (the current watched file + Jekyll-inferred URL)

This allows the app to show results but **exclude the file you're currently editing** from appearing in its own search results.

#### `prose_and_title(markdown: str, fallback_title: str) -> tuple[str, str]`

**Core text extraction.** Processes the Markdown to extract prose words:

1. Removes front matter
2. Skips fenced code blocks (` ``` ` or ` ~~~ `)
3. Removes reference definitions (`[ref]: url`)
4. Strips Markdown syntax: links, images, list markers, HTML comments, bare URLs
5. Unescapes HTML entities
6. Counts words using `WORD_PATTERN` (handles Unicode and contractions)

The result is clean prose (e.g., "The quick brown fox" instead of "The [quick](url) **brown** fox").

#### `Snapshot` dataclass

```python
@dataclass(frozen=True)
class Snapshot:
    digest: str          # SHA256 of the file's raw bytes
    count: int           # Prose word count
    title: str           # Extracted from front matter or filename
    context: str         # Cleaned prose (for OpenAI)
    trimmed: bool        # Whether context was trimmed to 12k words
```

Immutable snapshot of the file state at a given moment.

#### `read_snapshot(path: Path) -> Snapshot`

Reads the file, extracts prose and title, counts words, and trims if over 12,000 words. The trimming keeps the **first and last 6,000 words** to preserve both the start and end context:

```python
if trimmed:
    half = CONTEXT_WORD_LIMIT // 2
    prose = (prose[:words[half - 1].end()] + "\n\n[Middle of draft omitted]\n\n"
             + prose[words[-half].start():])
```

#### `CompanionError` and `error_message()`

A safe exception class for user-facing errors (no stack traces, no credentials leaked).

`error_message(exc)` converts various exceptions into short, friendly messages:
- OpenAI errors → "check key, model, quota"
- Hister errors → "check endpoint, token"
- Timeouts → "check services and retry"

### 4. Hister Client (lines 302–365)

```python
class HisterClient:
    """Read-only client for Hister's direct HTTP search API."""
```

#### `__init__`
Stores the endpoint URL and derives the config endpoint (used for auth/capability checks).

#### `get_json(endpoint: str, params: dict | None = None) -> dict`

Generic HTTP GET with error handling. Checks for JSON, validates it's a dict, looks for Hister error fields.

#### `check_available() -> None`

Before spending API quota, check:
- Is Hister running?
- Is authentication configured? Do we have a valid token?
- Is semantic search **enabled**? (crucial — keyword-only won't work)

#### `search(query: str, limit: int) -> list[dict]`

The core query. Sends:
```json
{
  "q": "user query here",
  "semantic": "true",
  "format": "json",
  "query": {"limit": 10, "include_html": false}  // JSON-encoded extra params
}
```

Then parses the response, extracting:
- `result["history"]` — recent docs
- `result["documents"]` — keyword matches
- `result["semantic_hits"]` — semantic embeddings with similarity scores and matched passages

All are dedup'd by URL and merged into a flat list of dicts.

#### `document_text(url: str) -> str`

Fetches the full indexed text for a given URL via `/api/document`. Used to load previews.

### 5. OpenAI Integration (lines 368–413)

```python
async def generate_queries(client: AsyncOpenAI, model: str, snapshot: Snapshot) -> list[str]:
```

Sends the draft to OpenAI and asks for **exactly three** conceptual search queries.

**Key details:**
- Uses **Structured Output API** — guarantees a JSON response with three strings
- Sets `store=False` — doesn't save the request for training
- Uses `reasoning.effort=none` **only for gpt-6-luna** (other models don't support it)
- Budget: 500 output tokens max
- The prompt is carefully written to treat the draft as **data**, not commands (injection defense)
- Drops punctuation from queries to avoid Hister query operators
- Checks for empty/duplicate queries and raises `CompanionError` if invalid

### 6. Result Merging (lines 416–455)

```python
@dataclass
class Hit:
    url: str
    title: str
    snippet: str          # from matching passage or Hister snippet
    kind: str
    similarity: object    # similarity score or None
    fused_rank: float = 0.0
    queries: list[str] = field(default_factory=list)
```

#### `merge_results(queries, groups, limit, excluded) -> list[Hit]`

Takes three lists of Hister results (one per query) and merges them:

1. **Reciprocal Rank Fusion (RRF)** — weights higher-ranked results more heavily:
   ```
   score += 1 / (60 + rank)
   ```
   This balances results that appear in multiple queries.

2. **Deduplication** — same URL only counted once (but tracks all queries that matched it)

3. **Exclusion** — removes the watched file and custom-excluded URLs

4. **Ranking** — sorts by fused score, returns top `limit`

### 7. Textual UI (lines 458–803)

#### `FileScreen` (lines 458–490)

A modal dialog to switch files. Let user type a new file path, validates it.

#### `PreviewText` (lines 493–500)

A read-only text area with a custom `action_copy()` that sends selected text to the terminal clipboard using OSC 52 (for remote SSH, tmux, etc.).

#### `Companion` App (lines 503–803)

The main TUI application.

##### **Initialization**

```python
def __init__(self, args: argparse.Namespace):
    self.path = args.file
    self.snapshot: Snapshot | None = None
    self.baseline: int | None = None
    self.attempted_digest: str | None = None
    self.paused = False
    self.busy = False
    self.hits: list[Hit] = []
    # ... many UI widget refs
```

Stores the watched file path, current snapshot, word count baseline (for threshold triggering), and search state.

##### **Layout**

```
┌─────────────────────────────────────────────┐
│ Markdown companion (Header)                 │
├─────────────────────────────────────────────┤
│ /path/to/draft.md (file-path)               │
│ 1,250 prose words · next search at 1,500    │
│ Searching Hister… (status)                  │
├──────────────────┬──────────────────────────┤
│  Results (45%)   │  Preview (55%)           │
│  1. Title        │  Title                   │
│  2. Title        │  URL                     │
│  ...             │  Similarity: 0.8234      │
│                  │                          │
│                  │  [Formatted|Select/copy] │
│                  │  ────────────────────    │
│                  │  Preview text here...    │
├──────────────────┴──────────────────────────┤
│ r=Search p=Pause f=File c=Copy q=Quit       │
└─────────────────────────────────────────────┘
```

##### **Polling (`poll()`)**

Called every 1 second. Reads the watched file:
- Computes a new `Snapshot`
- If the file digest changed, update snapshot and reset the "settle timer"
- If the file is gone/unreadable, show an error and wait for it to return

##### **Threshold Logic (`maybe_search()`)**

Decides whether to trigger a search:
- Is the file stable (settled for 2+ seconds)?
- Is it not already busy?
- Either manual search requested (`r` key) OR automatic (word count ≥ baseline + word-step)?
- Has the file actually changed since the last search?

If all true, queue a `search_cycle()` worker.

##### **Search Cycle (`search_cycle()`)**

The big async function:

1. Check Hister is available (auth, semantic search enabled)
2. Call `generate_queries()` → get three queries
3. Run three `hister.search()` calls in parallel (one per query)
4. Call `source_index()` to map Jekyll posts
5. Merge results with `merge_results()`
6. **Clear the preview cache** (old results for old file don't pollute)
7. Update UI: show queries, populate results list, show first result detail
8. Update status message with word count and timestamp
9. On any error, show the error in status and let the user retry with `r`

**Important:** Errors **preserve** previous results. If Hister goes down mid-search, old results stay visible.

##### **Preview Loading (`load_preview()`)**

When a result is selected:
1. Check if we've cached it already
2. If not cached:
   - Try to load from `sources` (Jekyll post paths)
   - Fall back to `hister.document_text()`
   - Cache the result
3. If still valid (user hasn't scrolled away), show it

This is **lazy** — previews load on demand, not upfront.

##### **Key bindings**

- `r` — force an immediate search (even if below threshold)
- `p` — pause/resume automatic searches
- `f` — switch to a different file (clears all state)
- `c` — copy the full current preview to clipboard
- Up/Down — navigate results
- Enter — open result in browser/file manager
- Tab — move focus between panes
- `q` — quit

##### **Opening results**

- Web URLs → `webbrowser.open()`
- Local file:// URLs → `xdg-open` (Linux), `open` (macOS), `os.startfile` (Windows)
- Validates to prevent injection/path traversal

### Key Design Decisions

1. **Async I/O** — Uses `asyncio` for OpenAI + Hister requests so the UI never blocks
2. **Worker threads** — Expensive file I/O (`read_snapshot`, `source_index`) runs in a thread pool, not blocking the event loop
3. **Snapshot + digest** — SHA256 hash prevents redundant work if file hasn't actually changed
4. **Settle timer** — Waits 2 seconds for editor saves to finish before triggering a search
5. **Exclusion by Jekyll URL** — Automatically excludes the watched post so results aren't self-referential
6. **Trimming for large drafts** — Over 12,000 words, keeps first+last 6,000 to stay under OpenAI context budgets
7. **Error recovery** — Preserves results on failure; no continuous retries
8. **Terminal clipboard** — Uses OSC 52 for remote environments (SSH, tmux)

The core loop is: poll → threshold check → search cycle → merge results → update UI → select result → load preview.
