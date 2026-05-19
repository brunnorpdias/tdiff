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

Dates accept `YYYY-MM-DD`, `today`, `yesterday`, `0` (today), `-N` for N days ago, or `w##` / `w2026-W##` for a weekly planning file (e.g. `w20` = week 20 of the current year).

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

tdiff 2026-05-17 w20           # day vs weekly planning file for week 20
tdiff w19 w20                  # weekly planning file week 19 vs week 20
tdiff w20 -D                   # find duplicates within a weekly planning file
tdiff w20 -w -1                # weekly planning file W20 vs aggregated week W19
tdiff w20 -w 0                 # weekly planning file W20 vs W20 daily notes only
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
| `-w N` / `--week-offset N` | Compare anchor day with the Sun-Sat week N weeks away (N is any integer: negative, 0, or positive) |
| `--no-color` | Disable colored output |
| `--no-summary` | Suppress the summary line |

If no `-d/-a/-c/-s` is given, all types except `same` are shown.

## Notes

- Daily notes are read from `01 Daily/<YYYY-MM-DD>` via the `obsidian` CLI; `#routine` tasks are filtered out.
- Weekly planning files live at `02 Weekly/YYYY-W##`. In `-w` mode, the weekly planning file for the target week is automatically merged into the week aggregation (those tasks get highest dedup priority). Missing weekly files are silently ignored.
- `w##` / `w2026-W##` can also be used as a direct date argument for day-vs-file or file-vs-file comparisons, as an anchor for `-w`, or as the target of `-D`.
- Project items (statuses `p`, `i`, `u`) are always excluded from output.
- Section suffixes (`task name - section`) are stripped before comparison so the same task under different daily sections still matches.
- Week aggregation uses **US Sun-Sat**, not ISO Mon-Sun. Internal label format: `%Y-W%U`.
- For `-w`, sign controls display order: `N <= 0` puts the week on the left (older-or-same), `N > 0` puts the anchor on the left.

### Deduplication

A single unified predicate decides whether two task strings refer to the same logical task. It's used everywhere — within a day, across days in a week aggregation, and in the `-D` duplicate finder.

Two tasks merge if **any** of these holds (after wikilink-path and trailing-punctuation normalization):

1. Their token sets are identical (e.g. `derivables..` vs `derivables`).
2. One token set is a strict subset of the other AND they share the same first word (e.g. `start imperial application` vs `start new imperial application`).

Wikilink folder paths are stripped inside `[[...]]` so `[[01 Daily/2026-05-03#recensio|alias]]` and `[[2026-05-03#recensio|alias]]` are treated as the same link.

When a cluster forms, the **canonical name** is the most recent day's wording (tie-break: longest), and the **winning status** is the highest `STATUS_PRIORITY` across the cluster (`x` > `-`/`#` > `!`/`*`/` ` > `/` > intra-day markers); status ties resolve to the latest day. The cross-note diff still uses Jaccard/prefix fuzzy matching at threshold 0.7 to label renamed tasks as "changed" rather than "added"+"deleted". In `-D` mode, `!!` marks exact duplicates (same text) and `~~` marks variant duplicates (different wording that resolved to the same task).
