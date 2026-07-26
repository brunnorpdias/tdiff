# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project does

`tdiff` is a single-file Python CLI that diffs tasks between Obsidian daily notes (`<YYYY-MM-DD>`) and weekly planning files (`YYYY-W##`) and reports what was added, removed, changed, or kept between two dates — or between a day/weekly-file and an aggregated week.

File lookups are folder-agnostic: `obsidian tasks file=<name>` resolves a file by its bare filename anywhere in the vault (wikilink-style resolution), so `tdiff` never hardcodes which folder daily notes or weekly files live in. An optional `[vault]` section in the config file (see Configuration below) can pin explicit folder prefixes if name resolution ever becomes ambiguous.

External dependencies: `obsidian` CLI (task extraction) and `rg` (ripgrep, filters `#routine` tasks). Requires Python 3.11+ (uses stdlib `tomllib`).

## Running

```bash
# Run directly (no build step)
tdiff date_a [date_b] [flags]

# Common invocations during development
python3 tdiff 2026-04-14 2026-04-15 --no-color
python3 tdiff yesterday today --no-color -T s
python3 tdiff --dupes today --no-color
python3 tdiff today -w -1 --no-color
python3 tdiff 2026-05-17 w20 --no-color    # day vs weekly planning file
python3 tdiff w20 -w -1 --no-color         # weekly file as anchor for week offset
python3 tdiff w23 w24 -F --no-color        # full Sun-Sat week vs full Sun-Sat week
```

No test suite — testing is manual via CLI invocation.

## Architecture

The entire tool lives in a single executable file: `tdiff` (582 lines, Python 3).

The script processes tasks in this pipeline:

1. **Parse** — `_parse_lines()` shells out to `obsidian tasks | rg -Fv '#routine'` and returns `(base, status, day_index)` records. Project items (statuses `p`, `i`, `u`) are always excluded.

2. **Deduplicate** — `cluster_records()` uses union-find to merge task variants across days. `is_same_task()` merges if token sets are identical OR one is a strict subset sharing the same first word. `materialize()` picks the canonical form (most recent day, longest on tie) and winning status (priority: `x > - > !/*/#/space > / > &«»>=~`).

3. **Week aggregation** (`fetch_week()`, built on `week_records()`) — Groups 7 daily notes into one virtual note using US Sun-Sat weeks, and merges in the `YYYY-W##` weekly planning file for that week at `day_index=0` (lowest dedup priority among the 7 days — losing ties against any daily note). Missing weekly files are silently skipped. In `-w 0` mode the anchor is excluded from its own week aggregation: if the anchor is a day, that day is excluded from daily notes; if it's a weekly file, the weekly file is excluded from the agg side. `-F/--full-week` (`to_full_week_side()`) expands *both* sides of a plain two-date or `--dupes` comparison into their containing full week independently — no anchor-exclusion, since neither side is "inside" the other.

4. **Diff** — Pass 1 does exact name matching; Pass 2 uses `best_match()` with Jaccard and prefix similarity at 0.7 threshold to relabel close add/delete pairs as "changed".

5. **Output** — Sorted by lowercase task name, colored by type: deleted=RED, added=GREEN, changed=YELLOW, same=DIM.

## Key flags

| Flag | Effect |
|------|--------|
| `-o N` | Compare anchor vs anchor ± N days |
| `-w N` | Compare anchor vs Sun-Sat week N weeks away |
| `-F` | Expand day/weekly-file arguments to their full Sun-Sat week aggregation (works with day-vs-day, weekly-file-vs-weekly-file, or mixed; also with `-D`) |
| `-W` | With `-w` or `-F`, exclude the `YYYY-W##` weekly planning file from the aggregation |
| `-D` | Find duplicates within a single date |
| `-T SET` | Show only rows whose type is in a char set (`d`=deleted, `a`=added, `c`=changed, `s`=same, e.g. `dac`); prefix `^` to invert (e.g. `^s`). Summary counts reflect the filtered rows. Default (no `-T`): show all types, with `same` rows sorted after every other row. |
| `-I` | Hide terminal-status items (`x`, `-`, `&`, `»`, `«`) on the A side only — suppresses deleted/unchanged rows whose A-status is terminal; added and changed rows are always shown. (A hard-coded negative special case of `-S`.) |
| `-S SET` | Show only rows whose effective status is in a char set (e.g. `x-#`); prefix `^` to invert (e.g. `^x`). Filters on the displayed status (B's for added/changed/same, A's for deleted); summary counts reflect the filtered rows. Bare `-S -` needs `-S=-` |
| `--no-color` | Disable ANSI colors |
| `--no-summary` | Suppress summary line |

## Configuration

Task-status behavior (dedup priority, `-I`'s ignore set, structural "project" exclusion) is driven by a TOML config file — not hardcoded in the script.

**Location** (first match wins): `--config PATH` flag → `$TDIFF_CONFIG` env var → `$XDG_CONFIG_HOME/tdiff/config.toml` → `~/.config/tdiff/config.toml`.

If the resolved path doesn't exist, `tdiff` creates it (with parent dirs) pre-populated with the built-in defaults, and prints a one-time note to stderr. Once the file exists it is the **complete** status table — a status char not listed in it falls back to `priority=0, ignore=false, project=false`.

```toml
[statuses.x]
name = "done"
priority = 5     # dedup precedence: higher wins when a task appears multiple times (ties break by most recent day)
ignore = true    # hidden by -I on the settled (A) side of a diff
# project = true # (not set here) structural marker; always excluded from every output

[vault]
daily_folder = ""    # optional explicit folder prefix; empty = resolve by filename anywhere in the vault
weekly_folder = ""
```

`load_config()` in `tdiff` populates the module-level `STATUS_PRIORITY`, `DEFAULT_IGNORE`, `PROJECT_STATUSES`, `DAILY_FOLDER`, `WEEKLY_FOLDER` globals right after arg parsing, before anything reads them.

## Date formats

`YYYY-MM-DD`, `today`, `yesterday`, `0` (today), `-N` (N days ago), `w##` or `w2026-W##` (weekly planning file, e.g. `w20` = week 20 of current year). Negative numbers use an internal `__NEG__` token workaround to avoid argparse conflicts. Weekly file refs resolve to side type `('weekly_file', 'YYYY-W##')` and are supported in day-vs-day, `-w`, and `-D` modes (not `-o`). With `-F`, either side (day or weekly file) is further expanded to side type `('week', ...)` — its containing full Sun-Sat week.
