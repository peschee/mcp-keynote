# Prompt: batch the describe_deck full read

Paste everything below the line into a new Claude Code session in this repo.

---

Rewrite the full-detail path of `describe_deck` (`DeckTools._describe_slides`
in `src/keynote_mcp/tools/deck.py`) so it reads Keynote a class at a time
instead of one element at a time. Work on branch `chore/mcp-1.30`. Read
`CLAUDE.md` first, then do this in order.

## Why

On this host (macOS 27.0, Keynote 15.3.1 "Keynote Creator Studio") every
Apple event costs about 17 ms. Finder and System Events take the same, so
the cost comes from the OS or the host, not Keynote. On macOS 26 it was
about 0.35 ms. `_describe_slides` sends roughly 200 events per slide on the
profiling deck (about 15 per text item, 10 per image, 4 per line, 2 to 3
per table cell, plus the `TEXT_ITEM_FILTER` identity loop and slide-level
reads). Measured with `scripts/verify_tools.py` on the 35-slide/560-element
scale deck: `slide_range='1-5'` takes 17.3 s (budget: "small and quick"),
and the full read takes 117.5 to 128.9 s, at or past the 120 s budget.

The summary path had the same problem and is already fixed in commit
`cf87612` (`_summarize_slides_batched`): 12.0 s down to 0.53 s, with output
identical to the old path on all 16 decks in `.scratch/`. Use it as the
pattern.

## What is known to work on 15.3.1 (probed)

- `<property> of every <class> of every slide of targetDoc` returns a list
  of lists in one event. Verified for `skipped`, `title showing`,
  `every image`, `object text of every text item`, and
  `object text of default title item`.
- Phantom placeholders can be subtracted by count: each existing default
  title or body item adds exactly one phantom, and phantoms trail the real
  items. This matched the identity filter on all 16 harness decks. Real text
  items are the first `count - hasDefaultTitle - hasDefaultBody` entries. A
  showing placeholder also appears among the real items in z-order, and its
  role (title/body) still has to be worked out. Check how `realRoles` is
  used before replacing the filter.
- `color/font/size of every character of object text of ti` already works
  per item (see `TEXT_RUNS_FRAGMENT` in `tools/fragments.py`). Check whether
  `... of every character of object text of every text item` works in one
  event. If it does not, keep the runs per item.

## Traps already hit

- `st`, `texts`, `before`, `nd`, `rd`, `th` are reserved in AppleScript.
  `set texts to ...` compiles and silently refers to the `text` class. Use
  names like `slideTexts`.
- The runner returns `stdout.strip()`, and Python's `str.strip()` treats
  `\x1c` to `\x1f` as whitespace. A record that ends in empty fields loses
  them on the last slide. Parse by count and pad, as
  `_summarize_slides_batched` does.
- A list read over `every slide` fails as a whole if one slide lacks the
  item (for example `default title item`). Wrap each list read in `try` and
  fall back to a per-slide loop for that one property.
- Property reads that can fail per element (`font`, `rotation`, `file of
  image`, `description`, `formula of cell`) are currently wrapped in `try`
  one element at a time. A list read over `every <class>` may fail if any
  element errors. Probe each one live before relying on it.

## Constraints

- Output must stay byte-identical to the current `_describe_slides`
  output, including `index`, `role`, placeholder `PT`/`PB` records, runs,
  table cell formulas, image paths, group counts and the `not_reported`
  block. `docs/INDEX_CONTRACT.md` defines what the indices mean.
- `element_types` filtering must still skip whole classes (the harness
  checks that filtering is a real speedup).
- Keep the current per-element path as a fallback when a batched read
  fails, as the summary does.
- Injection invariant: user strings go in as argv only. Only validated ints
  may be interpolated into the script.

## How to verify

1. Write a comparison script like the one used for the summary: for every
   `.scratch/*.key` deck, open it with `PresentationTools.open_presentation`,
   run the old and new `_describe_slides` on all slides with and without
   `include_text_runs`, and diff the parsed output. Close each deck with
   `saving no`. `.scratch/phase9-scale.key` is the large one; if it is
   missing, run `scripts/verify_tools.py` once to regenerate it.
2. Add unit tests in `tests/unit/test_describe_scale.py` that feed canned
   batched output through the parser (see `TestBatchedSummary`), including
   the stripped-trailing-fields case and the fallback.
3. `make check` must pass, with coverage at or above 85%.
4. Ask the user before live runs: they take window focus, and the user must
   not quit Keynote during them (that broke two earlier runs). Then run
   `uv run pytest -m keynote` and `uv run python scripts/verify_tools.py`.
   Targets: the `phase9/scale` page check under its budget and the full
   read well under 120 s. The known flaky check is the `add_table` header
   colour (failed 2 of 3 runs, unrelated).
5. Update the `describe_deck` row in `docs/TOOL_MATRIX.md` and the results
   line in `docs/ENVIRONMENT.md` with the measured numbers, then commit on
   the current branch.

## Also worth checking afterwards

Other tools also loop per element and will be slow on this host in the
same way, for example `get_slide_content` and the `TEXT_ITEM_FILTER`
identity loop used across `content.py`, `objects.py` and `deck.py`.
Measure first, and only batch where a real call is slow.
