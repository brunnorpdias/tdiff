# tdiff

Diff tasks between Obsidian daily notes.

A single-file Python CLI that pulls tasks from your Obsidian daily notes (`<YYYY-MM-DD>`) and weekly notes (`YYYY-W##`) and reports what was added, removed, changed, or kept between two of them — grouped under the projects they belong to.

File lookups are folder-agnostic: notes are resolved by filename anywhere in the vault (like an Obsidian wikilink), so `tdiff` doesn't care which folder your daily notes or weekly notes actually live in. See [Configuration](#configuration) if you ever need to pin explicit folders.

## Breaking changes

The flag surface was rebuilt around one idea: **a `w##` names a week, a date names a day, and a week has two sources you can narrow to.**

1. **`-F` and `-o` are gone; `-W` is new.** A week is now read whole by default — `tdiff w23 w24` is what `tdiff w23 w24 -F` used to be. `-D` reads only that week's dailies, `-W` only its `YYYY-W##` weekly note, and naming neither reads both. `-o N` was sugar over naming both dates; write `tdiff -7 today` instead of `tdiff today -o -7`.
2. **`-E` is gone.** It existed because `tdiff` couldn't read a weekly note's *Actio* section and `tcat` could. `tdiff` reads it now, so `-W` covers plan-vs-day directly.
3. **Projects are shown.** Tasks indented under a project header are grouped under it, and each project is its own dedup and diff scope — a task written both bare and under a project keeps a row in each.
4. **`w0` means this week.** Weeks can now be named relative to the current one — `w0`, `w-1`, `w+1` — which is what `-o` was reaching for. `w0` previously fell through to the bare-number form and resolved to `YYYY-W00`, a label no calendar produces; nothing useful is lost.
5. **A week derived from a date stops before that date.** `tdiff wednesday` compares Wednesday against Sunday–Tuesday, not against the whole week. Previously the later days leaked into the comparison; it only looked right for `today`, where the future is empty anyway.

## Requirements

- Python 3.11+ (uses the stdlib `tomllib` for config parsing)
- The `obsidian` CLI (used to read note markdown)

## Install

Drop `tdiff` somewhere on your `$PATH`. It's executable (`#!/usr/bin/env python3`). Then install the status table:

```sh
mkdir -p ~/.config/obsidian-tasks
cp statuses.example.toml ~/.config/obsidian-tasks/statuses.toml
```

## Configuration

Task-status behaviour (dedup precedence, `-I`'s settled set, which statuses mark a project, the hidden set) is driven by TOML — not hardcoded in the script.

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
project  = ["p", "i", "u"]              # structural; a grouping header, never a row
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

Dates accept `YYYY-MM-DD`, `today`, `yesterday`, `tomorrow`, `0` (today), `-N` / `+N` for N days either way, a weekday name (`monday`..`sunday`, or `mon`..`sun`, resolving to its most recent occurrence at or before today), or `w##` / `w2026-W##` for a Sun-Sat week (e.g. `w20` = week 20 of the current year). Weeks can also be named relative to this one: `w0` is the current week, `w-1` the one before, `w+1` the one after. `tcat` accepts exactly the same set — the resolver is shared code.

**A date names its daily note; a `w##` names its week.** A week has exactly two sources — its seven daily notes and its `YYYY-W##` weekly note — and `-D` / `-W` read just one of them. Naming neither reads both, which is why the two flags are mutually exclusive: together they'd be a second way to spell the default.

**Given one positional, it is compared against the week around it, minus itself.** A date is compared against the days before it; a `w##` is compared against its own dailies, with the weekly note on the left because that is what was written first.

### Examples

```
tdiff 2026-05-01 2026-05-08    # day vs day, explicit dates
tdiff today yesterday          # day vs day, shorthands
tdiff today -1                 # same as above (-1 = yesterday)
tdiff -7 today                 # today vs a week ago

tdiff today                    # today vs the rest of its week, weekly note included
tdiff today -D                 # ...daily notes only
tdiff today -W                 # ...the weekly note only: what I planned vs today
tdiff today -I                 # what is still open this week and not on today's list
tdiff wednesday -D             # any day works, and the week stops before it

tdiff w20                      # week 20's weekly note vs its own daily notes
tdiff w-1 w0                   # last week vs this week
tdiff w-1 w0 -D                # ...daily notes only
tdiff w23 w24                  # full Sun-Sat week 23 vs full week 24
tdiff w23 w24 -D               # same two weeks, daily notes only
tdiff w23 w24 -W               # same two weeks, weekly notes only
tdiff 2026-05-17 w20           # a day vs the whole of week 20

tdiff today -T a -I            # only additions, hiding terminal-status items
tdiff yesterday today -S x     # only rows that are now done
tdiff yesterday today -S '^x'  # everything except done
tdiff yesterday today -S 'x-#' # only done, cancelled, or blocked rows

tdiff monday today             # weekday names resolve backwards
tdiff today --json | jq .      # machine-readable output
```

## Flags

| Flag | Meaning |
| --- | --- |
| `-T SET` / `--types SET` | Show only rows whose type is in a char set (`d`=deleted, `a`=added, `c`=changed, `s`=same, e.g. `dac`); prefix `^` to invert (e.g. `^s`) |
| `-I` / `--ignore`  | Hide settled tasks — any row whose displayed status is named by `[roles] settled` |
| `-S SET` / `--status SET` | Show only rows whose effective status is in a char set (e.g. `x-#`); prefix `^` to invert (e.g. `^x`) |
| `-D` / `--dailies` | Read only a week's seven daily notes, not its weekly note |
| `-W` / `--weekly` | Read only a week's `YYYY-W##` weekly note, not its dailies |
| `--routines` | Include `#routine` tasks (excluded by default) |
| `--json` | Emit a JSON document instead of text (implies `--no-color`) |
| `--config PATH` | Additional (merging) config layer, applied last |
| `--no-color` | Disable colored output |
| `--no-summary` | Suppress the summary line |

By default all types are shown, with `same` rows sorted after every other row. Use `-T` to filter to specific types.

## Notes

- Daily notes (`<YYYY-MM-DD>`) are resolved by filename anywhere in the vault via the `obsidian` CLI, which returns the note's raw markdown; `#routine` tasks are filtered out unless you pass `--routines`. Pin an explicit folder with `[vault] daily_folder` in the config file if name resolution ever becomes ambiguous.
- Weekly notes (`YYYY-W##`) resolve the same way (or via `[vault] weekly_folder`). Merged into a week, the weekly note loses dedup ties against any daily note in that week. Missing notes are silently ignored.
- A weekly note is read as its **`### *Actio*` section** — the week's plan — including tasks listed under the heading but never allocated to a `**weekday**` marker. On a week you are still drafting that is usually all of them. Everything else in the note (Impressio, Relatio, Cultus, Fixa) is structure and reference, not this week's work, and is never read.
- **Tasks inside a fenced code block are never read**, in any note. Fencing is how a task list gets frozen — a weekly note's *Fixa*, a daily's *Actio Fixa*, a `**future**` bucket parked in a ` ```markdown ` block — and a frozen copy of the week is not work of its own.
- `**future**` is excluded from a weekly note and from daily notes read as part of a week; a daily note read on its own keeps its future bucket, since that's the day's own list.
- **Projects group the output.** A task indented under a project header is shown beneath it, and each project is its own dedup and diff scope — scopes never merge, so a task written both bare and under a project keeps a row in each, and one that moves between projects reads as deleted from the first and added to the second. A project whose every row is filtered out prints no header. Project headers themselves are never rows — they don't count towards the summary's added/deleted/changed/same tallies, though the summary does report how many project headers were printed.
- **Given one positional, it is compared against the week around it.** One rule underpins this: a positional is never compared against itself, so it leaves the week built around it. A date drops out of the dailies; a `w##` drops out as the weekly note, which leaves exactly the two sources, one per side — and is why a lone `w##` rejects `-D`/`-W`, since either would empty a side.
- **A week derived from a date stops strictly before it**, on both sources: the dailies run Sunday→date−1, and the weekly note's *Actio* whitelist stops one marker short. Nothing dated after a day can be outstanding as of it. A `w##` has no date to stop at, so its week is read whole.
- Given two positionals, neither sits inside the other, so both are read whole and nothing is excluded. Two `w##` naming the same week are rejected — the comparison would be a no-op.
- `-D`/`-W` narrow a week, so they need one: `tdiff yesterday today -D` is an error rather than quietly promoting both dates. Name the weeks (`tdiff w33 w34 -D`) or give a single date.
- `-S` filters on each row's **effective status** — the status actually shown in the row (`B`'s status for added/changed/same rows, `A`'s status for deleted rows). The summary line's counts (and total) reflect whatever `-S`, `-T`, and `-I` leave visible. A leading `^` inverts the whole set (`-S ^x` = everything except done). It composes with the `-T` type filter and with `-I`. Note: a bare `-S -` (only cancelled) looks like a flag to the parser — write it as `-S=-` or fold it into a set (`-S 'x-'`).
- `-I` hides any row whose effective status is in `[roles] settled`, whichever side it came from — a task finished on B is as settled as one finished on A, so `++ [x]` and `~~ [/] → [x]` go too.
- Three filters compose under one rule, the same one `tcat` uses: **a positive `-S` wins outright for the statuses it names.** So `-S '»'` shows postponed rows even though `[roles] hide` lists them, and `-I -S x` shows done rows rather than nothing. A negated `-S` only says what to drop, so `hide` and `-I` still apply to everything it doesn't name.
- Section suffixes (`task name - section`) are stripped before comparison so the same task under different daily sections still matches — but only when the dash is outside `[[...]]`, so a wikilink whose title contains a dash keeps it.
- Week aggregation uses **US Sun-Sat**, not ISO Mon-Sun. A week is labelled by the year it *ends* in: week 1 is the week containing Jan 1, so Sun 2026-12-27 → Sat 2027-01-02 is `2027-W01`, matching the vault templates' moment `gggg[-W]ww`. Only the week straddling New Year is ever affected.
- In the one-positional form the earlier side goes on the left, so the diff reads as "what came before that the anchor doesn't have". With two positionals the order you wrote them is the order you get.

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
| `p` | Project | A grouping header, never a row |
| `i` | Important project | A grouping header, never a row |
| `u` | Urgent project | A grouping header, never a row |

Dedup priority (highest wins when the same task appears with different statuses): `x` > `-` > `!`/`*`/`#`/` ` > `/` > intra-day markers; ties resolve to the latest day.

### Deduplication

A single unified predicate decides whether two task strings refer to the same logical task. It's used everywhere — within a note, and across the notes of a week aggregation.

Two tasks merge if **any** of these holds (after link and trailing-punctuation normalization):

1. Their token sets are identical (e.g. `derivables..` vs `derivables`).
2. One token set is a strict subset of the other AND they share the same first word (e.g. `start imperial application` vs `start new imperial application`).

Names are normalised first: Obsidian's `\[` escapes are undone, a trailing ` – section` suffix is dropped *while the brackets are still there* (so `[[a title – with a dash]]` survives intact), wikilinks keep their brackets and lose only their folder path (`[[01 Daily/2026-05-03|alias]]` → `[[2026-05-03|alias]]`), and markdown links reduce to their display text (`[label](https://…)` → `label`).

This is currently the one place `tdiff` and `tcat` disagree: `tcat` reduces a wikilink to its display text and strips the suffix afterwards, which truncates any linked title containing a dash. `tdiff` is the correct side; `tcat` will be reconciled to it.

When a cluster forms, the **canonical name** is the most recent day's wording (tie-break: longest), and the **winning status** is the highest `STATUS_PRIORITY` across the cluster (`x` > `-` > `!`/`*`/`#`/` ` > `/` > intra-day markers); status ties resolve to the latest day. The cross-note diff still uses Jaccard/prefix fuzzy matching at threshold 0.7 to label renamed tasks as "changed" rather than "added"+"deleted".
