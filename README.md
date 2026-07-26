# tdiff

Diff tasks between Obsidian daily notes.

A single-file Python CLI that pulls tasks from your Obsidian daily notes (`<YYYY-MM-DD>`) and weekly planning files (`YYYY-W##`) and reports what was added, removed, changed, or kept between two dates — or between a day/weekly-file and an aggregated week.

File lookups are folder-agnostic: notes are resolved by filename anywhere in the vault (like an Obsidian wikilink), so `tdiff` doesn't care which folder your daily notes or weekly files actually live in. See [Configuration](#configuration) if you ever need to pin explicit folders.

## Requirements

- Python 3.11+ (uses the stdlib `tomllib` for config parsing)
- The `obsidian` CLI (used to dump tasks per file)
- `rg` (ripgrep, used to filter `#routine` tasks)

## Install

Drop `tdiff` somewhere on your `$PATH`. It's executable (`#!/usr/bin/env python3`).

## Configuration

Task-status behavior (dedup priority, `-I`'s ignore set, structural "project" exclusion) and vault folder overrides live in a TOML config file — not hardcoded in the script.

**Location** (first match wins): `--config PATH` flag → `$TDIFF_CONFIG` env var → `$XDG_CONFIG_HOME/tdiff/config.toml` → `~/.config/tdiff/config.toml`.

If the resolved path doesn't exist, `tdiff` creates it (with parent dirs) pre-populated with the built-in defaults the first time you run it, and prints a one-time note to stderr. A reference copy of those defaults also lives in this repo at [`config.example.toml`](config.example.toml) — copy it to the path above (or just run `tdiff` once) to get started, then edit to taste. Once the file exists it is the **complete** status table — a status char not listed in it falls back to `priority=0, ignore=false, project=false`.

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

tdiff today -o -1              # anchor + day offset; chronological order auto-corrected
tdiff today -o 7               # anchor vs anchor + 7 days

tdiff today -w -1              # anchor day vs aggregated previous Sun-Sat week
tdiff yesterday -w 0           # anchor day vs the rest of its own Sun-Sat week
                               # (week on the left, day on the right)
tdiff -14 -w 1                 # anchor day vs the Sun-Sat week one week later

tdiff today -D                 # find duplicates within a single date
tdiff today -T a -I            # only additions, hiding terminal-status items

tdiff yesterday today -S x     # only rows that are now done
tdiff yesterday today -S '^x'  # everything except done
tdiff yesterday today -S 'x-#' # only done, cancelled, or blocked rows

tdiff 2026-05-17 w20           # day vs weekly planning file for week 20
tdiff w19 w20                  # weekly planning file week 19 vs week 20
tdiff w20 -D                   # find duplicates within a weekly planning file
tdiff w20 -w -1                # weekly planning file W20 vs aggregated week W19
tdiff w20 -w 0                 # weekly planning file W20 vs W20 daily notes only

tdiff w23 w24 -F                # full Sun-Sat week 23 vs full Sun-Sat week 24
tdiff 2026-06-08 2026-06-15 -F  # same idea, using any day within each week
tdiff w23 -D -F                 # find duplicates across all of week 23 (daily notes + weekly file)
```

## Flags

| Flag | Meaning |
| --- | --- |
| `-T SET` / `--types SET` | Show only rows whose type is in a char set (`d`=deleted, `a`=added, `c`=changed, `s`=same, e.g. `dac`); prefix `^` to invert (e.g. `^s`) |
| `-I` / `--ignore`  | Hide tasks in terminal statuses (`x`, `-`, `&`, `»`, `«`) on the A side |
| `-S SET` / `--status SET` | Show only rows whose effective status is in a char set (e.g. `x-#`); prefix `^` to invert (e.g. `^x`) |
| `-D` / `--dupes`   | Duplicate finder for a single date |
| `-o N` / `--offset N` | Compare anchor with anchor + N days (N nonzero, may be negative) |
| `-w N` / `--week-offset N` | Compare anchor day with the Sun-Sat week N weeks away (N is any integer: negative, 0, or positive) |
| `-F` / `--full-week` | Expand day/weekly-file arguments to their full Sun-Sat week aggregation before comparing (also works with `-D`) |
| `-W` / `--no-weekly` | With `-w` or `-F`, exclude the weekly planning file from the week aggregation |
| `--no-color` | Disable colored output |
| `--no-summary` | Suppress the summary line |

By default all types are shown, with `same` rows sorted after every other row. Use `-T` to filter to specific types.

## Notes

- Daily notes (`<YYYY-MM-DD>`) are resolved by filename anywhere in the vault via the `obsidian` CLI; `#routine` tasks are filtered out. Pin an explicit folder with `[vault] daily_folder` in the config file if name resolution ever becomes ambiguous.
- Weekly planning files (`YYYY-W##`) resolve the same way (or via `[vault] weekly_folder`). In `-w` or `-F` mode, the weekly planning file for the target week is automatically merged into the week aggregation (losing dedup ties against any daily note in that week). Missing weekly files are silently ignored. Use `-W` to exclude it from the aggregation.
- `w##` / `w2026-W##` can also be used as a direct date argument for day-vs-file or file-vs-file comparisons, as an anchor for `-w`, or as the target of `-D`.
- `-F` expands either side of a plain two-date or `-D` comparison (day or weekly file) into its containing full Sun-Sat week — unlike `-w`, both sides are independent weeks, so there's no self-exclusion logic.
- `-S` filters on each row's **effective status** — the status actually shown in the row (`B`'s status for added/changed/same rows, `A`'s status for deleted rows). The summary line's counts (and total) reflect whatever `-S`, `-T`, and `-I` leave visible. A leading `^` inverts the whole set (`-S ^x` = everything except done). It composes with the `-T` type filter and with `-I`. Note: a bare `-S -` (only cancelled) looks like a flag to the parser — write it as `-S=-` or fold it into a set (`-S 'x-'`).
- `-I` is effectively the hard-coded negative special case of `-S`: it hides terminal-status rows (`x`, `-`, `&`, `»`, `«`) on the A side only (deleted and unchanged rows); added and changed rows are always shown.
- Project items (statuses `p`, `i`, `u`) are always excluded from output.
- Section suffixes (`task name - section`) are stripped before comparison so the same task under different daily sections still matches.
- Week aggregation uses **US Sun-Sat**, not ISO Mon-Sun. Internal label format: `%Y-W%U`.
- For `-w`, sign controls display order: `N <= 0` puts the week on the left (older-or-same), `N > 0` puts the anchor on the left.

### Task statuses

Each task carries a single-character status (the `[ ]` marker in Obsidian). These are the characters `-S` and `-I` operate on. The table below reflects the built-in defaults — all of it (names, priority, ignore/project flags, and which chars exist at all) is configurable via `~/.config/tdiff/config.toml`; see [Configuration](#configuration).

| Status | Meaning | Notes |
| --- | --- | --- |
| `x` | Done | Terminal; sticky (wins dedup ties) |
| `-` | Cancelled | Terminal |
| `&` | Overrun | Terminal (intra-day marker) |
| `»` | Postponed | Terminal (intra-day marker) |
| `«` | Advanced | Terminal (intra-day marker) |
| `!` | Urgent | |
| `*` | Important | |
| ` ` (space) | Task (no status / open) | Quote it for `-S`: `-S ' '` |
| `#` | Blocked / waiting | |
| `o` | Recurrent | |
| `/` | Partial | |
| `>` | Current | Intra-day marker |
| `=` | Paused / switch | Intra-day marker |
| `~` | Snoozed | Intra-day marker |
| `p` | Project | Always excluded from output |
| `i` | Important project | Always excluded from output |
| `u` | Urgent project | Always excluded from output |

Dedup priority (highest wins when the same task appears with different statuses): `x` > `-` > `!`/`*`/`#`/` ` > `/` > intra-day markers; ties resolve to the latest day.

### Deduplication

A single unified predicate decides whether two task strings refer to the same logical task. It's used everywhere — within a day, across days in a week aggregation, and in the `-D` duplicate finder.

Two tasks merge if **any** of these holds (after wikilink-path and trailing-punctuation normalization):

1. Their token sets are identical (e.g. `derivables..` vs `derivables`).
2. One token set is a strict subset of the other AND they share the same first word (e.g. `start imperial application` vs `start new imperial application`).

Wikilink folder paths are stripped inside `[[...]]` so `[[01 Daily/2026-05-03#recensio|alias]]` and `[[2026-05-03#recensio|alias]]` are treated as the same link.

When a cluster forms, the **canonical name** is the most recent day's wording (tie-break: longest), and the **winning status** is the highest `STATUS_PRIORITY` across the cluster (`x` > `-` > `!`/`*`/`#`/` ` > `/` > intra-day markers); status ties resolve to the latest day. The cross-note diff still uses Jaccard/prefix fuzzy matching at threshold 0.7 to label renamed tasks as "changed" rather than "added"+"deleted". In `-D` mode, `!!` marks exact duplicates (same text) and `~~` marks variant duplicates (different wording that resolved to the same task).
