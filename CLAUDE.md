# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project does

`tdiff` is a single-file Python CLI that diffs tasks between Obsidian daily notes and weekly planning files. It reads tasks from `01 Daily/<YYYY-MM-DD>` and `02 Weekly/YYYY-W##` vault files and reports what was added, removed, changed, or kept between two dates — or between a day/weekly-file and an aggregated week.

External dependencies: `obsidian` CLI (task extraction) and `rg` (ripgrep, filters `#routine` tasks).

## Running

```bash
# Run directly (no build step)
tdiff date_a [date_b] [flags]

# Common invocations during development
python3 tdiff 2026-04-14 2026-04-15 --no-color
python3 tdiff yesterday today --no-color --same
python3 tdiff --dupes today --no-color
python3 tdiff today -w -1 --no-color
python3 tdiff 2026-05-17 w20 --no-color    # day vs weekly planning file
python3 tdiff w20 -w -1 --no-color         # weekly file as anchor for week offset
```

No test suite — testing is manual via CLI invocation.

## Architecture

The entire tool lives in a single executable file: `tdiff` (444 lines, Python 3).

The script processes tasks in this pipeline:

1. **Parse** — `_parse_lines()` shells out to `obsidian tasks | rg -Fv '#routine'` and returns `(base, status, day_index)` records. Project items (statuses `p`, `i`, `u`) are always excluded.

2. **Deduplicate** — `cluster_records()` uses union-find to merge task variants across days. `is_same_task()` merges if token sets are identical OR one is a strict subset sharing the same first word. `materialize()` picks the canonical form (most recent day, longest on tie) and winning status (priority: `x > -/# > !/*/space > / > &«»>=~`).

3. **Week aggregation** (`fetch_week()`) — Groups 7 daily notes into one virtual note using US Sun-Sat weeks, and merges in the `02 Weekly/YYYY-W##` file for that week at `day_index=7` (highest dedup priority, so weekly file wording wins ties). Missing weekly files are silently skipped. In `-w 0` mode the anchor is excluded from its own week aggregation: if the anchor is a day, that day is excluded from daily notes; if it's a weekly file, the weekly file is excluded from the agg side.

4. **Diff** — Pass 1 does exact name matching; Pass 2 uses `best_match()` with Jaccard and prefix similarity at 0.7 threshold to relabel close add/delete pairs as "changed".

5. **Output** — Sorted by lowercase task name, colored by type: deleted=RED, added=GREEN, changed=YELLOW, same=DIM.

## Key flags

| Flag | Effect |
|------|--------|
| `-t N` | Compare anchor vs anchor ± N days |
| `-w N` | Compare anchor vs Sun-Sat week N weeks away |
| `-W` | With `-w`, exclude the `02 Weekly/YYYY-W##` planning file from the aggregation |
| `-D` | Find duplicates within a single date |
| `-d/-a/-c/-s` | Show deleted/added/changed/same tasks |
| `-I` | Hide terminal-status items (`x`, `-`, `&`, `»`, `«`) |
| `--no-color` | Disable ANSI colors |
| `--no-summary` | Suppress summary line |

## Date formats

`YYYY-MM-DD`, `today`, `yesterday`, `0` (today), `-N` (N days ago), `w##` or `w2026-W##` (weekly planning file, e.g. `w20` = week 20 of current year). Negative numbers use an internal `__NEG__` token workaround to avoid argparse conflicts. Weekly file refs resolve to side type `('weekly_file', 'YYYY-W##')` and are supported in day-vs-day, `-w`, and `-D` modes (not `-t`).
