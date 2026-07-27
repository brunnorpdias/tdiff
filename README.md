# tdiff

Diff tasks between Obsidian daily notes.

A single-file Python CLI that pulls tasks from your Obsidian daily notes (`<YYYY-MM-DD>`) and weekly planning files (`YYYY-W##`) and reports what was added, removed, changed, or kept between two dates — or between a day/weekly-file and an aggregated week.

File lookups are folder-agnostic: notes are resolved by filename anywhere in the vault (like an Obsidian wikilink), so `tdiff` doesn't care which folder your daily notes or weekly files actually live in. See [Configuration](#configuration) if you ever need to pin explicit folders.

## Breaking changes

Three changes land together, all in the name of `tdiff` and its sibling [`tcat`](../tcat) behaving alike:

1. **The status table moved, and is no longer created for you.** `tdiff` now reads the shared vault vocabulary at `~/.config/obsidian-tasks/statuses.toml`, in the same layered scheme `tcat` uses, and never writes a config file. With no config at all it exits with an error instead of silently ranking every status 0. Install [`statuses.example.toml`](statuses.example.toml) to get going. The old `[statuses.X]` schema is gone — see [Configuration](#configuration).
2. **`&`, `»` and `«` are hidden by default**, via `[roles] hide`. Name them in `-S` to bring them back (`-S '&»«'`).
3. **`--config` now merges rather than replaces.** Pointing it at a partial file used to wipe the whole table without saying so.

## Requirements

- Python 3.11+ (uses the stdlib `tomllib` for config parsing)
- The `obsidian` CLI (used to dump tasks per file)

## Install

Drop `tdiff` somewhere on your `$PATH`. It's executable (`#!/usr/bin/env python3`). Then install the status table:

```sh
mkdir -p ~/.config/obsidian-tasks
cp statuses.example.toml ~/.config/obsidian-tasks/statuses.toml
```

## Configuration

Task-status behaviour (dedup precedence, `-I`'s settled set, structural "project" exclusion, the hidden set) is driven by TOML — not hardcoded in the script.

The status table describes **your vault's notation**, not this tool, so it lives at `~/.config/obsidian-tasks/statuses.toml` — a directory neither `tdiff` nor `tcat` owns. Both read it, each ignoring what it has no use for, and neither needs the other installed. `tdiff` reads `[roles]`, `[dedup]` and `[vault]`; `tcat` reads `[order]`, `[roles]`, `[theme.*]` and `[vault]`.

**Layers**, lowest precedence first — each *merges* over the ones below, so a partial file never erases what a lower layer set:

1. `~/.config/obsidian-tasks/statuses.toml` — the shared vault vocabulary
2. `~/.config/tdiff/config.toml` — tdiff-only overlay (see [`config.example.toml`](config.example.toml))
3. `$TDIFF_CONFIG`
4. `--config PATH`

Nothing is ever bootstrapped. With none of these present `tdiff` exits with an error rather than guessing: without `[roles] project` it would leak project headers into every diff, and without `[dedup]` it would pick an arbitrary status for any task stated twice. Sections that are present but incomplete degrade with a note on stderr.

```toml
[dedup]
# Which status wins when one task appears more than once. Tiers, highest first;
# the inner list is a tie (broken by most recent day). Unlisted chars rank 0.
priority = [["x"], ["-"], ["!", "*", " ", "#", "o"], ["/"], ["&", "»", "«", ">", "=", "~"]]

[roles]
project  = ["p", "i", "u"]              # structural; always excluded from every output
hide     = ["&", "»", "«"]              # never shown unless -S names them
settled  = ["x", "-", "&", "»", "«"]    # hidden by -I on the settled (A) side

[vault]
daily_folder = ""    # optional explicit folder prefix; empty = resolve by filename anywhere in the vault
weekly_folder = ""
```

## Usage

```
tdiff date_a [date_b] [flags]
```

Dates accept `YYYY-MM-DD`, `today`, `yesterday`, `tomorrow`, `0` (today), `-N` / `+N` for N days either way, a weekday name (`monday`..`sunday`, or `mon`..`sun`, resolving to its most recent occurrence at or before today), or `w##` / `w2026-W##` for a weekly planning file (e.g. `w20` = week 20 of the current year). `tcat` accepts exactly the same set — the resolver is shared code.

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

tdiff monday today              # weekday names resolve backwards
tdiff -E 'tcat today -P' 'tcat today -I'   # what I planned vs what is still open
tdiff -E 'tcat w30 -A -P' 'tcat w30 -A'    # a whole week, plan vs actual
tdiff -E -D 'tcat w30 -A'                  # duplicates across a week, via tcat
```

### Comparing external task sets (`-E`)

`-E` reads both positionals as shell commands instead of dates. Anything that emits tasks can be a side, which is what makes **plan vs actual** reachable — `tdiff` has no idea how to read a weekly note's Actio allocation, but [`tcat -P`](../tcat) does:

```sh
tdiff -E 'tcat today -P' 'tcat today -I'
```

`--json` is appended to each command unless it already asks for it. Output is parsed as JSON — `tcat`'s envelope (project children flattened) or a bare `[{"status", "name"}]` array — falling back to `- [x] name` lines when it isn't JSON, so non-`tcat` sources work too. A non-zero exit is fatal; empty output is just an empty side.

`-E` composes with `-D`, `-T`, `-S`, `-I` and `--json`. It's rejected with `-o`, `-w` and `-F`: those are date arithmetic, and the equivalent belongs inside the command (`tcat w30 -A`, not `tdiff -F`). It's all-or-nothing — mixing a date with a command isn't supported; put the date in the command instead.

## Flags

| Flag | Meaning |
| --- | --- |
| `-T SET` / `--types SET` | Show only rows whose type is in a char set (`d`=deleted, `a`=added, `c`=changed, `s`=same, e.g. `dac`); prefix `^` to invert (e.g. `^s`) |
| `-I` / `--ignore`  | Hide settled tasks — the statuses named by `[roles] settled` — on the A side |
| `-S SET` / `--status SET` | Show only rows whose effective status is in a char set (e.g. `x-#`); prefix `^` to invert (e.g. `^x`) |
| `-D` / `--dupes`   | Duplicate finder for a single date |
| `-o N` / `--offset N` | Compare anchor with anchor + N days (N nonzero, may be negative) |
| `-w N` / `--week-offset N` | Compare anchor day with the Sun-Sat week N weeks away (N is any integer: negative, 0, or positive) |
| `-F` / `--full-week` | Expand day/weekly-file arguments to their full Sun-Sat week aggregation before comparing (also works with `-D`) |
| `-W` / `--no-weekly` | With `-w` or `-F`, exclude the weekly planning file from the week aggregation |
| `-E` / `--exec` | Read both positionals as shell commands emitting tasks, not as dates |
| `--routines` | Include `#routine` tasks (excluded by default) |
| `--json` | Emit a JSON document instead of text (implies `--no-color`) |
| `--config PATH` | Additional (merging) config layer, applied last |
| `--no-color` | Disable colored output |
| `--no-summary` | Suppress the summary line |

By default all types are shown, with `same` rows sorted after every other row. Use `-T` to filter to specific types.

## Notes

- Daily notes (`<YYYY-MM-DD>`) are resolved by filename anywhere in the vault via the `obsidian` CLI; `#routine` tasks are filtered out unless you pass `--routines`. Pin an explicit folder with `[vault] daily_folder` in the config file if name resolution ever becomes ambiguous.
- Weekly planning files (`YYYY-W##`) resolve the same way (or via `[vault] weekly_folder`). In `-w` or `-F` mode, the weekly planning file for the target week is automatically merged into the week aggregation (losing dedup ties against any daily note in that week). Missing weekly files are silently ignored. Use `-W` to exclude it from the aggregation.
- `w##` / `w2026-W##` can also be used as a direct date argument for day-vs-file or file-vs-file comparisons, as an anchor for `-w`, or as the target of `-D`.
- `-F` expands either side of a plain two-date or `-D` comparison (day or weekly file) into its containing full Sun-Sat week — unlike `-w`, both sides are independent weeks, so there's no self-exclusion logic.
- `-S` filters on each row's **effective status** — the status actually shown in the row (`B`'s status for added/changed/same rows, `A`'s status for deleted rows). The summary line's counts (and total) reflect whatever `-S`, `-T`, and `-I` leave visible. A leading `^` inverts the whole set (`-S ^x` = everything except done). It composes with the `-T` type filter and with `-I`. Note: a bare `-S -` (only cancelled) looks like a flag to the parser — write it as `-S=-` or fold it into a set (`-S 'x-'`).
- `-I` hides rows whose effective status is in `[roles] settled`, on the A side only (deleted and unchanged rows); added and changed rows are always shown.
- Three filters compose under one rule, the same one `tcat` uses: **a positive `-S` wins outright for the statuses it names.** So `-S '»'` shows postponed rows even though `[roles] hide` lists them, and `-I -S x` shows done rows rather than nothing. A negated `-S` only says what to drop, so `hide` and `-I` still apply to everything it doesn't name.
- Project items (statuses `p`, `i`, `u`) are always excluded from output.
- Section suffixes (`task name - section`) are stripped before comparison so the same task under different daily sections still matches.
- Week aggregation uses **US Sun-Sat**, not ISO Mon-Sun. A week is labelled by the year it *ends* in: week 1 is the week containing Jan 1, so Sun 2026-12-27 → Sat 2027-01-02 is `2027-W01`, matching the vault templates' moment `gggg[-W]ww`. Only the week straddling New Year is ever affected.
- For `-w`, sign controls display order: `N <= 0` puts the week on the left (older-or-same), `N > 0` puts the anchor on the left.

### Task statuses

Each task carries a single-character status (the `[ ]` marker in Obsidian). These are the characters `-S` and `-I` operate on. The table below reflects the shipped `statuses.example.toml` — all of it (dedup precedence, the role sets, and which chars exist at all) is configurable; see [Configuration](#configuration).

| Status | Meaning | Notes |
| --- | --- | --- |
| `x` | Done | Settled; sticky (wins dedup ties) |
| `-` | Cancelled | Settled |
| `&` | Overrun | Settled; hidden by default (`[roles] hide`) |
| `»` | Postponed | Settled; hidden by default (`[roles] hide`) |
| `«` | Advanced | Settled; hidden by default (`[roles] hide`) |
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
