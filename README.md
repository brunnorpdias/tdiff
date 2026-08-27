# tdiff

Diff tasks between Obsidian daily notes.

A Python CLI that pulls tasks from your Obsidian daily notes (`<YYYY-MM-DD>`) and weekly notes (`YYYY-W##`) and reports what was added, removed, changed, or kept between two of them — grouped under the projects they belong to.

File lookups are folder-agnostic: notes are resolved by filename anywhere in the vault (like an Obsidian wikilink), so `tdiff` doesn't care which folder your daily notes or weekly notes actually live in. See [Configuration](#configuration) if you ever need to pin explicit folders.

## Breaking changes

The flag surface was rebuilt around one idea: **a `w##` names a week, a date names a day, and a week has two sources you can narrow to.**

1. **`-F` and `-o` are gone; `-W` is new.** A week is now read whole by default — `tdiff w23 w24` is what `tdiff w23 w24 -F` used to be. `-D` reads only that week's dailies, `-W` only its `YYYY-W##` weekly note, and naming neither reads both. `-o N` was sugar over naming both dates; write `tdiff -7 today` instead of `tdiff today -o -7`.
2. **`-E` is gone.** It existed because `tdiff` couldn't read a weekly note's plan section and `tcat` could. `tdiff` reads the whole note now, so `-W` covers plan-vs-day directly.
3. **Projects are shown.** Tasks indented under a project header are grouped under it, and each project is its own dedup and diff scope — a task written both bare and under a project keeps a row in each.
4. **`w0` means this week.** Weeks can now be named relative to the current one — `w0`, `w-1`, `w+1` — which is what `-o` was reaching for. `w0` previously fell through to the bare-number form and resolved to `YYYY-W00`, a label no calendar produces; nothing useful is lost.
5. **A week derived from a date stops before that date.** `tdiff wednesday` compares Wednesday against Sunday–Tuesday, not against the whole week. Previously the later days leaked into the comparison; it only looked right for `today`, where the future is empty anyway.

## Requirements

- Python 3.11+ (uses the stdlib `tomllib` for config parsing)
- The `obsidian` CLI (used to read note markdown)

## Install

Drop `tdiff` somewhere on your `$PATH`. It's executable (`#!/usr/bin/env python3`). It reads the vault through [`tnotes`](https://github.com/brunnorpdias/tnotes), a module it shares with `tcat`, which goes in `~/.local/lib`:

```sh
mkdir -p ~/.local/lib ~/.config/tconfig
ln -s "$PWD/../tnotes/tnotes.py" ~/.local/lib/tnotes.py
cp ../tnotes/notation.example.toml ~/.config/tconfig/notation.toml
cp ../tnotes/statuses.example.toml ~/.config/tconfig/statuses.toml
```

Then edit `notation.toml` to match how you write your notes — which sections are not really task lists, how you mark a weekday, which character cuts a comment off a task name. There is no install step and nothing is created for you.

## Configuration

Task-status behaviour (dedup precedence, `-I`'s settled set, which statuses mark a project, the hidden set) is driven by TOML — not hardcoded in the script.

The config describes **your vault**, not this tool, so it lives in `~/.config/tconfig/` — a folder neither `tdiff` nor `tcat` owns. Both read it, each ignoring what it has no use for, and neither needs the other installed. It is split by concern rather than by tool, because that is the axis along which a file actually changes: `notation.toml` says how the vault *writes* things (`[exclude]`, `[days]`, `[comment]`, `[vault]`), `statuses.toml` says what the statuses *mean* (`[order]`, `[dedup]`, `[roles]`, `[theme.*]`).

**Layers**, lowest precedence first — each *merges* over the ones below, so a partial file never erases what a lower layer set:

1. `~/.config/tconfig/notation.toml` — how the vault writes things
2. `~/.config/tconfig/statuses.toml` — what the statuses mean
3. `~/.config/tconfig/tdiff.toml` — tdiff-only overrides (see [`tdiff.example.toml`](tdiff.example.toml))
4. `$TDIFF_CONFIG`
5. `--config PATH`

`$TCONFIG_DIR` relocates the folder. If `tconfig/` holds none of these, the pre-`tconfig` layout — `~/.config/obsidian-tasks/statuses.toml` and `~/.config/tdiff/config.toml` — is read instead, with one notice naming the new home. That is a fallback for the first run after the move, not a layer: a `tconfig/` that exists wins outright.

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

[exclude]
# Headings never read, at any level, along with everything under them. ">"
# separates ancestors, so "archive" names that heading anywhere and
# "plan > deferred" only the one under "plan". Emphasis and case are ignored.
# Headings only — bold text carries no level, so where such a section ends
# would be a guess.
sections = ["archive", "plan > deferred"]

[days]
# How your vault writes each weekday marker, as a **literal** matched against the
# whole line — anchored, so the marker must be alone on it. Nothing is a wildcard:
# asterisks, brackets and pipes mean themselves. `{date}` is the one placeholder
# and stands for an ISO date. Read for one purpose: a week derived from a date
# drops tasks allocated to that day or later. A list gives a day several spellings,
# which keeps your own history readable after you change the notation. Omit the
# section and a weekly note is never bounded by the date.
monday = ["**monday**", "[[{date}|monday]]"]
# ... and the other six

[comment]
# Characters that cut a trailing comment off a task name, so
#   - [ ] do blood screening – will complete saturday
# is the task `do blood screening`. Only counts between spaces and outside every
# bracket and parenthesis, so a linked title keeps its own dashes. Name only what
# your vault uses as notation: listing the ASCII hyphen alongside the en dash will
# truncate names that were only ever prose. Omit it and nothing is stripped.
separators = ["–"]
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
| `-W` / `--weekly` | Read only a week's `YYYY-W##` weekly note, not its dailies. On a date-derived week it narrows further, to that date's own allocation |
| `--all` | Ignore the whole `[exclude]` table — sections and tags alike — for one run (fenced blocks are still skipped) |
| `--json` | Emit a JSON document instead of text (implies `--no-color`) |
| `--config PATH` | Additional (merging) config layer, applied last |
| `--no-color` | Disable colored output |
| `--no-summary` | Suppress the summary line |

By default all types are shown, with `same` rows sorted after every other row. Use `-T` to filter to specific types.

## Notes

- Daily notes (`<YYYY-MM-DD>`) are resolved by filename anywhere in the vault via the `obsidian` CLI, which returns the note's raw markdown; task lines carrying a tag named in `[exclude] tags` are filtered out unless you pass `--all`. Pin an explicit folder with `[vault] daily_folder` in the config file if name resolution ever becomes ambiguous.
- Weekly notes (`YYYY-W##`) resolve the same way (or via `[vault] weekly_folder`). Merged into a week, the weekly note loses dedup ties against any daily note in that week. Missing notes are silently ignored.
- A weekly note is **read whole**, like a daily one. It used to be pinned to one heading and to a whitelist of day markers, both spelled into the script, which decided for you where a plan lives — a plan parked in a sibling section was unreachable however you configured things. Name what you don't want in `[exclude] sections` instead. No section name and no marker spelling appears anywhere in the script.
- **Day markers come from `[days]`**, and are read for one purpose: a week derived from a date drops tasks allocated to that day or later. A task under no marker is never dropped — on a week you are still drafting that is usually all of them. With no `[days]` nothing opens a day, so a weekly note is never bounded by the date; the daily notes still truncate, since their bound is the filename.
- **A marker is matched exactly and must be alone on its line, and markers are only ever looked for in weekly notes.** A task or a comment that happens to mention a weekday is not a marker. A daily note carries its date in its filename, so a marker found there could only be a false one. `[days]` values used to be globs, where `**saturday**` meant *any line containing "saturday"* — 394 false matches against 172 real markers in this vault, and 22 task lines swallowed whole because the marker test ran before the task test. Both are fixed; `{date}` is now the entire wildcard vocabulary.
- **Tasks inside a fenced code block are never read**, in any note. Fencing is one way a task list gets frozen, and a frozen copy of the week is not work of its own.
- **`[exclude] sections` filters everything that isn't fenced.** Name a heading and it is never read, along with everything under it — the durable version of the same idea, since a section is skipped for what it is called rather than for how it happens to be formatted. Nothing is special-cased by name.
- `--all` ignores the whole `[exclude]` table for one run — sections and tags alike. It does not lift the fence.
- **Projects group the output.** A task indented under a project header is shown beneath it, and each project is its own dedup and diff scope — scopes never merge, so a task written both bare and under a project keeps a row in each, and one that moves between projects reads as deleted from the first and added to the second. A project whose every row is filtered out prints no header. Project headers themselves are never rows — they don't count towards the summary's added/deleted/changed/same tallies, though the summary does report how many project headers were printed.
- **Given one positional, it is compared against the week around it.** One rule underpins this: a positional is never compared against itself, so it leaves the week built around it. A date drops out of the dailies; a `w##` drops out as the weekly note, which leaves exactly the two sources, one per side — and is why a lone `w##` rejects `-D`/`-W`, since either would empty a side.
- **A week derived from a date stops before it**, but the two sources stop at different places. The dailies run Sunday→date−1, so the day's own note stays on the B side and is never compared against itself. The weekly note drops only what is allocated *after* that date, keeping the date's own block: that block is not a copy of the day's list, it is what the day was supposed to be — and it is what makes a task the plan set for today and never carried over show up, as `deleted`. A `w##` has no date to stop at, so its week is read whole.
- **`-W` on a date narrows to that date's allocation alone**, which is the one place a flag changes more than which source gets read. Without the dailies nothing is left to say a task got done, so a task planned for Sunday and finished on Sunday would sit on the A side at its planned status and read as `deleted` — a false alarm the bare form never shows, because Sunday's daily deduped it to `[x]`. Asking the narrower question instead — *what did the plan want from this day* — removes the whole class. Work carried over from an earlier day then reads as `added` rather than `same`: the same fact, stated in the less alarming direction.
- Given two positionals, neither sits inside the other, so both are read whole and nothing is excluded. Two `w##` naming the same week are rejected — the comparison would be a no-op.
- `-D`/`-W` narrow a week, so they need one: `tdiff yesterday today -D` is an error rather than quietly promoting both dates. Name the weeks (`tdiff w33 w34 -D`) or give a single date.
- `-S` filters on each row's **effective status** — the status actually shown in the row (`B`'s status for added/changed/same rows, `A`'s status for deleted rows). The summary line's counts (and total) reflect whatever `-S`, `-T`, and `-I` leave visible. A leading `^` inverts the whole set (`-S ^x` = everything except done). It composes with the `-T` type filter and with `-I`. Note: a bare `-S -` (only cancelled) looks like a flag to the parser — write it as `-S=-` or fold it into a set (`-S 'x-'`).
- `-I` hides any row whose effective status is in `[roles] settled`, whichever side it came from — a task finished on B is as settled as one finished on A, so `++ [x]` and `~~ [/] → [x]` go too.
- Three filters compose under one rule, the same one `tcat` uses: **a positive `-S` wins outright for the statuses it names.** So `-S '»'` shows postponed rows even though `[roles] hide` lists them, and `-I -S x` shows done rows rather than nothing. A negated `-S` only says what to drop, so `hide` and `-I` still apply to everything it doesn't name.
- **Trailing comments (`task name – a note about it`) are stripped before comparison**, so the same task annotated differently on two days still matches. Which characters count is yours to set in `[comment] separators`; the separator only counts between spaces and outside every bracket *and* parenthesis, so a linked title keeps its own dashes. Name nothing and nothing is stripped. This was hardcoded to take the ASCII hyphen along with the dashes, which truncated 30 names in this vault that were only ever prose.
- Week aggregation uses **US Sun-Sat**, not ISO Mon-Sun. A week is labelled by the year it *ends* in: week 1 is the week containing Jan 1, so Sun 2026-12-27 → Sat 2027-01-02 is `2027-W01`, matching the vault templates' moment `gggg[-W]ww`. Only the week straddling New Year is ever affected.
- In the one-positional form the earlier side goes on the left, so the diff reads as "what came before that the anchor doesn't have". With two positionals the order you wrote them is the order you get.

### Task statuses

Each task carries a single-character status (the `[ ]` marker in Obsidian). These are the characters `-S` and `-I` operate on. The table below reflects the shipped `statuses.example.toml` (in the `tnotes` repo) — all of it (dedup precedence, the role sets, and which chars exist at all) is configurable; see [Configuration](#configuration).

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

**Two tasks are the same task iff their names are identical after normalisation, ignoring case.** There is no similarity metric anywhere in the tool.

That is a deliberate reversal. Dedup used to merge two names whose token sets were equal, or where one was a strict subset sharing a first word; the diff had a second pass that relabelled close add/delete pairs as `changed` using Jaccard and prefix similarity at 0.7. Both guessed wrong often enough to matter — `purchase coffee` swallowed `purchase new coffee grinder`, `create new plan (3/8)` was matched to `(5/8)` — and no threshold separates the bad merges from the good ones, which are structurally identical. Exact matching costs **+1.4% rows** across 158 measured runs and buys a tool that never claims two tasks are one. A task you reworded now reads as deleted and added, which is what actually happened to the note.

Names are normalised first: Obsidian's `\[` escapes are undone, a trailing ` – section` suffix is dropped *while the brackets are still there* (so `[[a title – with a dash]]` survives intact), wikilinks keep their brackets and lose only their folder path (`[[01 Daily/2026-05-03|alias]]` → `[[2026-05-03|alias]]`), and markdown links reduce to their display text (`[label](https://…)` → `label`).

`tdiff` and `tcat` share one copy of all of this, in `tnotes`, so they cannot disagree about it.

When a cluster forms, the **canonical name** is the most recent day's wording (tie-break: longest) — which, since the cluster members differ only in case, is really a choice of spelling — and the **winning status** is the highest `STATUS_PRIORITY` across the cluster (`x` > `-` > `!`/`*`/`#`/` ` > `/` > intra-day markers); status ties resolve to the latest day. Case is folded because a vault spells a wikilink both ways (25 names here differ only by case) and project scope keys have always been lowercased.

A `changed` row therefore means exactly one thing: **same name, different status.**
