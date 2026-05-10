# tdiff

Diff tasks between Obsidian daily notes.

A single-file Python CLI that pulls tasks from your Obsidian daily notes (path: `01 Daily/<YYYY-MM-DD>`) and reports what was added, removed, changed, or kept between two dates — or between a day and an aggregated week.

## Requirements

- Python 3
- The `obsidian` CLI (used to dump tasks per file)
- `rg` (ripgrep, used to filter `#routine` tasks)

## Install

Drop `tdiff` somewhere on your `$PATH`. It's executable (`#!/usr/bin/env python3`).

## Usage

```
tdiff date_a [date_b] [flags]
```

Dates accept `YYYY-MM-DD`, `today`, `yesterday`, `0` (today), or `-N` for N days ago.

### Examples

```
tdiff 2026-05-01 2026-05-08    # day vs day, explicit dates
tdiff today yesterday          # day vs day, shorthands
tdiff today -1                 # same as above (-1 = yesterday)

tdiff today -t -1              # anchor + day offset; chronological order auto-corrected
tdiff today -t 7               # anchor vs anchor + 7 days

tdiff today -w -1              # anchor day vs aggregated previous Sun-Sat week
tdiff yesterday -w 0           # anchor day vs the rest of its own Sun-Sat week
                               # (week on the left, day on the right)
tdiff -14 -w 1                 # anchor day vs the Sun-Sat week one week later

tdiff today -D                 # find duplicates within a single date
tdiff today -a -I              # only additions, hiding terminal-status items
```

## Flags

| Flag | Meaning |
| --- | --- |
| `-d` / `--deleted` | Show deleted tasks |
| `-a` / `--added`   | Show added tasks |
| `-c` / `--changed` | Show changed tasks |
| `-s` / `--same`    | Show unchanged tasks (hidden by default) |
| `-I` / `--ignore`  | Hide tasks in terminal statuses (`x`, `-`, `&`, `»`, `«`) |
| `-D` / `--dupes`   | Duplicate finder for a single date |
| `-t N` / `--offset N` | Compare anchor with anchor + N days (N nonzero, may be negative) |
| `-w N` / `--week-offset N` | Compare anchor day with the Sun-Sat week N weeks away (N may be 0 or negative) |
| `--no-color` | Disable colored output |
| `--no-summary` | Suppress the summary line |

If no `-d/-a/-c/-s` is given, all types except `same` are shown.

## Notes

- Notes are read from `01 Daily/<YYYY-MM-DD>` via the `obsidian` CLI; `#routine` tasks are filtered out.
- Project items (statuses `p`, `i`, `u`) are always excluded from output.
- Within a single day — and within a week aggregation — duplicate task names are deduped via `STATUS_PRIORITY`: the highest-priority status wins, with ties resolved by the most recent occurrence.
- Section suffixes (`task name - section`) are stripped before comparison so the same task under different daily sections still matches.
- Fuzzy matching (Jaccard or shared-prefix at threshold 0.7) catches renamed/edited tasks.
- Week aggregation uses **US Sun-Sat**, not ISO Mon-Sun. Internal label format: `%Y-W%U`.
- For `-w`, sign controls display order: `N <= 0` puts the week on the left (older-or-same), `N > 0` puts the day on the left.
