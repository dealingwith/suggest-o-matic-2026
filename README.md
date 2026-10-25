## Suggest-o-matic 2026

`suggesto.py` is a single-file Python 3.11+ terminal app that follows a saved Markdown draft and finds related material in your existing Hister index. The design is recorded in [MARKDOWN_COMPANION_PLAN.md](MARKDOWN_COMPANION_PLAN.md).

### Hister setup

Install Hister using its [installation guide](https://hister.org/docs/installing), then follow the [quickstart](https://hister.org/docs/quickstart) to run the server and index content. Enable semantic search using Hister's [semantic search configuration](https://hister.org/docs/configuration#semantic-search) before running Suggest-o-matic; Hister requires an embeddings endpoint for this feature.

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

The **Formatted** tab renders Markdown headings, lists, emphasis, and code. Hister's indexed web text may lack the original Markdown formatting. The **Select / copy text** tab is read-only and supports mouse selection, Shift+arrow selection, and Ctrl+A. Ctrl+C in that tab copies the selected text. In either preview tab, `c` copies the selected text when there is a selection; otherwise it copies the current preview (passages or the displayed longer excerpt). Changing previews clears the previous selection. Clipboard integration is delegated to [Pyperclip](https://github.com/asweigart/pyperclip), which selects the platform backend. Copies run in a background thread and report success only after reading back matching clipboard text. Pyperclip still requires a working system clipboard provider on Linux; installing the Python package alone does not provide one. It uses native Windows and macOS clipboard support as well.

For Pop!_OS with GNOME Terminal, install the clipboard helpers from any directory:

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

Paste into a GNOME text editor; expect `suggestomatic clipboard check`. If `wl-copy` fails under GNOME Wayland, try `xclip` through XWayland. To test app copying, select a result and press `c`, then paste into the editor. For selected text, focus the **Select / copy text** tab, select a passage, and press Ctrl+C. GNOME Terminal's Ctrl+Shift+C shortcut copies the terminal's own selection rather than invoking the app's copy action.

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

### Manual verification

Run the launch command above and confirm the count matches the saved prose. If the file has fewer than 250 words, expect it to wait; press `r` to search early. At or above 250 words, expect three queries and related Hister results (or an explicit empty-results message) after the file settles.

- Launch without a file: expect a focused filename search box and a file list. Type part of a filename, choose with Up/Down and Enter, and confirm the selected path and prose count. Try a query with no matches, then clear it. Escape before selecting should exit.
- While watching, press `b` and choose another file: expect the watcher and results to reset. Escape from this picker should keep the existing watched file. An explicit file argument should bypass the picker.

- Add 250 net words and save: one new search should run. Revise without increasing the count: no automatic search should run.
- Pause, add words, and save: counting continues without an automatic request.
  Resume: a qualifying saved draft is searched.
- Switch files during a request: old responses should not populate the new file.
- Save using an editor that replaces files atomically, or temporarily move the draft away and back: the watcher should recover.
- Stop Hister or use invalid credentials: expect an error with previous results preserved and no continuous retries. Restore the service/key and press `r`.
- Disable semantic search in Hister: expect an explicit error before the OpenAI call. Re-enable it and press `r`. The direct API must return JSON, not HTML.
- Check repeated URLs appear once and the watched post is absent, including its published URL. Only similarity is displayed; there is no Score field.
- Select a large external Logseq result: expect short matched passages labeled with their queries and similarities, without an automatic full-document load. A result without a matched chunk should explicitly identify its fallback excerpt.
- Press `v`: expect a longer preview, capped at 20,000 characters with a truncation notice for large documents. Press `v` again to return to matches. Switch results during loading and confirm an older preview never replaces the selection.
- Check Markdown formatting in the Formatted tab. Select a passage in each tab, press `c`, and paste into your editor: expect only the selection. Clear the selection and press `c`: expect the current preview. Ctrl+C in the text tab should also copy its selection. Change results and confirm previous selections are cleared. Enter on the result list should still open the selected result. For a draft over 12,000 words, expect a query-context trimming notice.
