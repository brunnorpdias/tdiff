# tdiff

Diff tasks between Obsidian daily notes.

A Python CLI that pulls tasks from your Obsidian daily notes (`<YYYY-MM-DD>`) and weekly notes (`YYYY-W##`) and reports what was added, removed, changed, or kept between two of them — grouped under the projects they belong to.

Notes are resolved by filename anywhere in the vault (like an Obsidian wikilink), so `tdiff` doesn't care which folder they live in.

## Requirements

- Python 3.11+ (stdlib `tomllib`)
- The `obsidian` CLI (reads note markdown)

## Install

Drop `tdiff` somewhere on your `$PATH` — it's executable. It reads the vault through [`tnotes`](https://github.com/brunnorpdias/tnotes), a module it shares with `tcat`, which goes in `~/.local/lib`:

```sh
mkdir -p ~/.local/lib ~/.config/tconfig
ln -s "$PWD/../tnotes/tnotes.py" ~/.local/lib/tnotes.py
cp ../tnotes/notation.example.toml ~/.config/tconfig/notation.toml
cp ../tnotes/statuses.example.toml ~/.config/tconfig/statuses.toml
```

Then edit `notation.toml` to match how you write your notes. Nothing is bootstrapped: with no config present `tdiff` exits with an error rather than guessing.

## Usage

```
tdiff date_a [date_b] [flags]
```

Dates accept `YYYY-MM-DD`, `today`, `yesterday`, `tomorrow`, `0`, `-N` / `+N` for N days either way, a weekday name (`monday`..`sunday` or `mon`..`sun`, resolving backwards to its most recent occurrence), or `w##` / `w2026-W##` for a Sun-Sat week. Weeks can be named relative to this one: `w0`, `w-1`, `w+1`.

**A date names its daily note; a `w##` names its week.** A week has two sources — its seven daily notes and its `YYYY-W##` weekly note — and `-D` / `-W` read just one. Naming neither reads both, so the two flags are mutually exclusive.

**Given one positional, it is compared against the week around it, minus itself.** A date is compared against the days before it; a `w##` against its own dailies, weekly note on the left. Given two, both are read whole and the order you wrote them is the order you get.

```
tdiff 2026-05-01 2026-05-08    # day vs day, explicit dates
tdiff today yesterday          # day vs day, shorthands
tdiff -7 today                 # today vs a week ago

tdiff today                    # today vs the rest of its week, weekly note included
tdiff today -D                 # ...daily notes only
tdiff today -W                 # ...the weekly note only: what I planned vs today
tdiff today -I                 # what is still open this week and not on today's list
tdiff wednesday -D             # any day works, and the week stops before it

tdiff w20                      # week 20's weekly note vs its own daily notes
tdiff w-1 w0                   # last week vs this week
tdiff w23 w24                  # full Sun-Sat week 23 vs full week 24
tdiff w23 w24 -W               # same two weeks, weekly notes only
tdiff 2026-05-17 w20           # a day vs the whole of week 20

tdiff today -T a -I            # only additions, hiding settled items
tdiff yesterday today -S x     # only rows that are now done
tdiff yesterday today -S '^x'  # everything except done
tdiff today --json | jq .      # machine-readable output
```

## Flags

| Flag | Meaning |
| --- | --- |
| `-T SET` / `--types SET` | Show only rows whose type is in a char set (`d`=deleted, `a`=added, `c`=changed, `s`=same, e.g. `dac`); prefix `^` to invert |
| `-S SET` / `--status SET` | Show only rows whose effective status is in a char set (e.g. `x-#`); prefix `^` to invert. A bare `-S -` looks like a flag — write `-S=-` |
| `-I` / `--ignore` | Hide settled tasks — any row whose displayed status is named by `[roles] settled` |
| `-D` / `--dailies` | Read only a week's seven daily notes, not its weekly note |
| `-W` / `--weekly` | Read only a week's `YYYY-W##` weekly note. On a date-derived week it narrows to that date's own allocation |
| `--all` | Ignore the whole `[exclude]` table for one run (fenced blocks are still skipped) |
| `--json` | Emit a JSON document instead of text (implies `--no-color`) |
| `--config PATH` | Additional (merging) config layer, applied last |
| `--no-color` | Disable colored output |
| `--no-summary` | Suppress the summary line |

`-D`/`-W` narrow a week, so they need one: `tdiff yesterday today -D` is an error rather than promoting two dates into a week. All three status filters compose under one rule — **a positive `-S` wins outright for the statuses it names**, so `-I -S x` shows done rows rather than nothing.

## Configuration

Task-status behaviour (dedup precedence, `-I`'s settled set, which statuses mark a project, the hidden set) is driven by TOML.

The config describes **your vault**, not this tool, so it lives in `~/.config/tconfig/` — a folder shared with `tcat`, split by concern rather than by tool. **Layers**, lowest precedence first; each *merges* over the ones below:

1. `~/.config/tconfig/notation.toml` — how the vault writes things (`[exclude]`, `[days]`, `[comment]`, `[vault]`)
2. `~/.config/tconfig/statuses.toml` — what the statuses mean (`[order]`, `[dedup]`, `[roles]`)
3. `~/.config/tconfig/tdiff.toml` — tdiff-only overrides (see [`tdiff.example.toml`](tdiff.example.toml))
4. `$TDIFF_CONFIG`
5. `--config PATH`

`$TCONFIG_DIR` relocates the folder.

```toml
[dedup]
# Which status wins when one task appears more than once. Tiers, highest first;
# the inner list is a tie (broken by most recent day). Unlisted chars rank 0.
priority = [["x"], ["-"], ["!", "*", " ", "#", "o"], ["/"], ["&", "»", "«", ">", "=", "~"]]

[order]
# Display rank; position in the list is the rank, so `x` sorts last.
statuses = ["u", "!", "i", "*", ">", "=", "o", "p", " ", "/", "#", "~", "&", "»", "«", "-", "x"]

[roles]
project  = ["p", "i", "u"]              # structural; a grouping header, never a row
hide     = ["&", "»", "«"]              # never shown unless -S names them
settled  = ["x", "-", "&", "»", "«"]    # hidden by -I

[vault]
daily_folder = ""    # optional explicit folder prefix; empty = resolve by filename
weekly_folder = ""

[exclude]
# Headings never read, at any level, along with everything under them. ">"
# separates ancestors, so "archive" names that heading anywhere and
# "plan > deferred" only the one under "plan". Emphasis and case are ignored.
sections = ["archive", "plan > deferred"]
tags     = ["#routine"]                 # task lines carrying one are dropped

[days]
# How your vault writes each weekday marker, as a **literal** matched against the
# whole line — anchored, so the marker must be alone on it. Nothing is a wildcard;
# `{date}` is the one placeholder and stands for an ISO date. Read for one purpose:
# a week derived from a date drops tasks allocated to that day or later. A list
# gives a day several spellings. Omit the section and a weekly note is never
# bounded by the date.
monday = ["**monday**", "[[{date}|monday]]"]
# ... and the other six

[comment]
# Characters that cut a trailing comment off a task name, so
#   - [ ] do blood screening – will complete saturday
# is the task `do blood screening`. Only counts between spaces and outside every
# bracket and parenthesis, so a linked title keeps its own dashes. Name only what
# your vault uses as notation. Omit it and nothing is stripped.
separators = ["–"]
```

## How it reads a note

- Notes are read as **raw markdown**. Tasks inside a fenced code block are never read; `[exclude] sections` skips a named heading and everything under it, `[exclude] tags` drops a single line. `--all` lifts the `[exclude]` table but not the fence.
- A weekly note is read **whole**, like a daily one — no section name and no marker spelling appears anywhere in the script.
- **Day markers come from `[days]`**, are matched exactly, must be alone on their line, and are looked for in weekly notes only. A task under no marker is never dropped.
- **Projects group the output.** A task indented under a project header is shown beneath it, and each project is its own dedup and diff scope — scopes never merge, so a task written both bare and under a project keeps a row in each, and one that moves between projects reads as deleted from the first and added to the second. Headers are never rows and never counted.
- **A week derived from a date stops before it**, but the two sources stop at different places: the dailies run Sunday→date−1, while the weekly note keeps that date's own block — what the day was *supposed* to be, which is what makes a task planned for today and never carried over show up at all.
- Week aggregation uses **US Sun-Sat**, and a week is labelled by the year it *ends* in: Sun 2026-12-27 → Sat 2027-01-02 is `2027-W01`.

## Task statuses

Each task carries a single-character status (the `[ ]` marker in Obsidian) — these are the characters `-S` and `-I` operate on. The table reflects the shipped `statuses.example.toml`; all of it is configurable.

| Status | Meaning | Notes |
| --- | --- | --- |
| `x` | Done | Settled; wins dedup ties |
| `-` | Cancelled | Settled |
| `&` | Overrun | Settled; hidden by default |
| `»` | Postponed | Settled; hidden by default |
| `«` | Advanced | Settled; hidden by default |
| `!` | Urgent | |
| `*` | Important | |
| ` ` (space) | Open | Quote it for `-S`: `-S ' '` |
| `#` | Blocked / waiting | |
| `o` | Recurrent | |
| `/` | Partial | |
| `>` | Current | Intra-day marker |
| `=` | Paused / switch | Intra-day marker |
| `~` | Snoozed | Intra-day marker |
| `p` / `i` / `u` | Project / important / urgent | A grouping header, never a row |

## Matching and names

**Two tasks are the same task iff their names are identical after normalisation, ignoring case.** There is no similarity metric anywhere in the tool, so a `changed` row means exactly one thing — same name, different status — and a task you reworded reads as deleted and added, which is what actually happened to the note.

Normalisation undoes Obsidian's `\[` escapes, drops a trailing ` – comment`, shortens wikilinks (folder path always; the anchor only where an alias already stands in for it, so an unaliased `[[calculus#multivariable]]` keeps it) and reduces markdown links to their display text.

**On screen, an aliased link shows only its alias, in a single bracket** — `[[…bodner ⟦book⟧.pdf|learning go (5/15) – functions]]` prints as `[learning go (5/15) – functions]`. The printed text is no longer a link, so it must not look like one; an unaliased link keeps both brackets, because there the target is the whole of what it says. This is display only — `--json`, dedup, the diff and project scope keys all keep the canonical name.

When a task appears more than once, the surviving spelling is the most recent day's (tie-break: longest) and the status is the highest `[dedup] priority` across the cluster.

`tdiff` and `tcat` share one copy of all of this, in `tnotes`, so they cannot disagree about it.

## JSON

`--json` writes one object to stdout with `mode`, `a`, `b`, `rows` and `summary`. Filters are applied before serializing, so `rows` is exactly what text mode would print. Statuses are bare chars; `old_status` appears on `changed` rows, `project` on rows under one, and each side's `files` lists the vault files actually read — the only place `-D`/`-W` and the anchor truncation are observable.

---

Design rationale and the history behind each rule live in [CLAUDE.md](CLAUDE.md).
