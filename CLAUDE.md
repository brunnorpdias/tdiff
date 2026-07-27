# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project does

`tdiff` is a single-file Python CLI that diffs tasks between Obsidian daily notes (`<YYYY-MM-DD>`) and weekly planning files (`YYYY-W##`) and reports what was added, removed, changed, or kept between two dates — or between a day/weekly-file and an aggregated week.

File lookups are folder-agnostic: `obsidian tasks file=<name>` resolves a file by its bare filename anywhere in the vault (wikilink-style resolution), so `tdiff` never hardcodes which folder daily notes or weekly files live in. An optional `[vault]` section in the config file (see Configuration below) can pin explicit folder prefixes if name resolution ever becomes ambiguous.

External dependency: the `obsidian` CLI (task extraction). Requires Python 3.11+ (uses stdlib `tomllib`).

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
python3 tdiff today -w -1 --json | jq .    # machine-readable output
python3 tdiff monday today --no-color      # weekday names resolve backwards
python3 tdiff -E 'tcat today -P' 'tcat today -I' --no-color   # plan vs actual
```

No test suite — testing is manual via CLI invocation.

**The regression check that matters most is dedup ordering**, because it is silent when
wrong. Capture `python3 tdiff today -w -1 --json` before a change touching the config or
`materialize()`, and diff it after; it must be byte-identical unless you meant otherwise.

## Architecture

The entire tool lives in a single executable file: `tdiff` (~1090 lines, Python 3).

The script processes tasks in this pipeline:

1. **Read** — `_run_obsidian()` execs `obsidian tasks file=<name>` directly (no shell, no pipe); `#routine` lines are filtered in Python unless `--routines` is passed. Failures are fatal, never silent: a missing `obsidian` binary, a non-zero exit, an unexpected `Error:` line on stdout, or a stalled call all print `tdiff: ...` to stderr and exit 2. Only obsidian's `Error: File "X" not found.` is treated as an empty note (days without notes and missing weekly files stay silent). `obsidian_lines()` memoizes per file path and `prefetch()` warms that cache concurrently (`FETCH_WORKERS` threads), submitting `obsidian_lines` itself so the vault read has exactly one seam — the concurrent path used to call `_run_obsidian` directly, leaving a second entry point that anything hooking the read would miss — every vault read for both sides of a comparison is issued in one batch, which is where most of the wall-clock went. See **Obsidian CLI stalls** below.

2. **Parse** — `_parse_obsidian()` turns those lines into `(base, status, day_index)` records. Statuses are stored **bracketed** (`'[x]'`, not `'x'`); every consumer downstream indexes `[1]` for the char, so an alternative record source that forgets this fails silently rather than loudly. Project items (`[roles] project`) are always excluded.

   **Or not from the vault at all** — with `-E`, `side_records()` dispatches to `extern_records()`, which runs a shell command and reads its task list instead. That is the whole of the coupling: everything below this step is pure and never touches a path or a subprocess.

3. **Deduplicate** — `cluster_records()` uses union-find to merge task variants across days. Two bases are the same task if their token sets are identical OR one is a strict subset sharing the same first word. Rather than scanning all pairs, it buckets bases by token-set (equality merges) and by first token (subset merges) — same clusters, far fewer comparisons. `materialize()` picks the canonical form (most recent day, longest on tie) and winning status by `STATUS_PRIORITY`, which comes from `[dedup].priority` in the config.

4. **Week aggregation** (`side_entries()` / `week_entries()`, materialized by `fetch_side()`) — Groups 7 daily notes into one virtual note using US Sun-Sat weeks, and merges in the `YYYY-W##` weekly planning file for that week at `day_index=0` (lowest dedup priority among the 7 days — losing ties against any daily note). Missing weekly files are silently skipped. In `-w 0` mode the anchor is excluded from its own week aggregation: if the anchor is a day, that day is excluded from daily notes; if it's a weekly file, the weekly file is excluded from the agg side. `-F/--full-week` (`to_full_week_side()`) expands *both* sides of a plain two-date or `--dupes` comparison into their containing full week independently — no anchor-exclusion, since neither side is "inside" the other.

5. **Diff** — Pass 1 does exact name matching; Pass 2 uses `best_match()` with Jaccard and prefix similarity at 0.7 threshold to relabel close add/delete pairs as "changed". Candidates carry precomputed token sets, and the expensive `prefix_sim()` (difflib) is skipped whenever it cannot change the outcome — different first char, or a similarity ceiling below the running best.

6. **Output** — Sorted by lowercase task name, colored by type: deleted=RED, added=GREEN, changed=YELLOW, same=DIM. Rows are built as dicts by `row()` and turned into text by `render()` at print time, so the text and `--json` modes are rendered from one source and can't drift.

## Key flags

| Flag | Effect |
|------|--------|
| `-o N` | Compare anchor vs anchor ± N days |
| `-w N` | Compare anchor vs Sun-Sat week N weeks away |
| `-F` | Expand day/weekly-file arguments to their full Sun-Sat week aggregation (works with day-vs-day, weekly-file-vs-weekly-file, or mixed; also with `-D`) |
| `-W` | With `-w` or `-F`, exclude the `YYYY-W##` weekly planning file from the aggregation |
| `-D` | Find duplicates within a single date |
| `-T SET` | Show only rows whose type is in a char set (`d`=deleted, `a`=added, `c`=changed, `s`=same, e.g. `dac`); prefix `^` to invert (e.g. `^s`). Summary counts reflect the filtered rows. Default (no `-T`): show all types, with `same` rows sorted after every other row. |
| `-I` | Hide settled items — the statuses in `[roles] settled` — on the A side only; added and changed rows are always shown. |
| `-E` | Read both positionals as shell commands emitting tasks, rather than as dates. See **External task sets** below. |
| `--routines` | Include `#routine` tasks (excluded by default) |
| `-S SET` | Show only rows whose effective status is in a char set (e.g. `x-#`); prefix `^` to invert (e.g. `^x`). Filters on the displayed status (B's for added/changed/same, A's for deleted); summary counts reflect the filtered rows. Bare `-S -` needs `-S=-` |
| `--json` | Emit a JSON document instead of text (implies `--no-color`; `--no-summary` does not apply) |
| `--no-color` | Disable ANSI colors |
| `--no-summary` | Suppress summary line |

### How the three status filters compose

`hide`, `-I` and `-S` all live in `suppressed()`, under one rule borrowed from `tcat`'s
`status_match()`: **a positive `-S` wins outright for the statuses it names.** So
`-S '»'` shows postponed rows even though `[roles] hide` lists them, and `-I -S x` shows
done rows rather than nothing. A negated `-S` says only what to drop, so `hide` and `-I`
still apply to everything it doesn't name.

`-I` is restricted to types 0 and 3 (deleted, same) — the rows whose displayed status
comes from the settled A side. Dupes mode passes `typ=0`: there is only one side and it
is the A side. Both callers go through the one function; the diff loop used to re-derive
the same rule inline from a separate `ignore_set`.

Filtering happens **after** dedup, so an earlier `[»]` never suppresses a later `[x]`.

## External task sets (`-E`)

`-E` reinterprets both positionals as shell commands, adding a fourth side kind
`('extern', command_string)` alongside `('day', …)`, `('week', …)`, `('weekly_file', …)`.
It exists so plan-vs-actual is reachable: `tdiff` has no idea how to read a weekly note's
Actio allocation, and shouldn't — `tcat -P` already does.

```sh
tdiff -E 'tcat today -P' 'tcat today -I'    # planned vs still open
tdiff -E -D 'tcat w30 -A'                   # dupes across a week
```

`side_records(side)` (not `side_entries`) is the dispatch point; `side_entries()` stays
vault-only and is undefined for `extern`. `extern_records()` appends `--json` unless the
command already asks for it, parses stdout as JSON — `tcat`'s `{"tasks": […]}` envelope
with project children flattened, or a bare `[{"status", "name"}]` array — and falls back
to `- [c] name` / `  [c] name` line parsing when stdout isn't JSON, which makes any task
emitter usable. Names go through the same `normalize_wikilinks` / `strip_section_suffix`
cleanup a vault line gets, so both sides are compared on equal terms.

Two things that will look like oversights and are not:

- **Every extern record gets `day_index = 0`.** An external list has already been
  deduplicated by whatever produced it, and `tcat` sorts alphabetically rather than
  chronologically, so position carries no date meaning to tie-break on. Letting
  `STATUS_PRIORITY` decide alone is the correct reading, not a missing `enumerate()`.
- **`-E` is all-or-nothing.** Mixing a date with a command is deliberately unsupported —
  put the date in the command (`tdiff -E 'tcat 2026-07-20' 'tcat today'`). One rule for
  reading a positional beats two that have to be told apart.

Rejected with `-o`, `-w` and `-F`: those are date arithmetic, and the equivalent belongs
inside the command (`tcat w30 -A`, not `tdiff -F`). `side_json()` emits `kind`, `label`
and `command` with **no `files` key** — nothing in the vault was read, and the command is
the whole provenance.

`subprocess.run(..., shell=True, stdin=DEVNULL)`. `stdin=DEVNULL` matters for the same
reason it does in the obsidian reader: a child that drains stdin would silently eat
whatever is piped into the calling script. No timeout — it is the user's command.

## Obsidian CLI stalls

Roughly **one `obsidian tasks` call in a few hundred wedges and never returns** — measured at 180s with no output, while sibling calls kept answering in ~10ms. It happens at the same rate reading serially or concurrently (1/480 at 1 worker, 2/480 at 4, 4/480 at 8), so it is not caused by tdiff's threading; concurrency only widens the window because week modes issue 16 calls instead of 2. This is why week/`-F` modes appeared to hang while day-vs-day rarely did.

Because the wedge is **per-invocation** and the app stays healthy, the fix is a short deadline plus a fresh call, not a longer wait:

- `OBSIDIAN_TIMEOUT` (default 1s, `TDIFF_TIMEOUT=N`) — a healthy call is ~10ms (20ms worst of 320 measured), so this is ~100x headroom, and it keeps a stall inside the one second the whole tool should take. A merely slow call isn't lost by the tight deadline; the retry catches it.
- `OBSIDIAN_ATTEMPTS` (3) — a retry after a stall practically always succeeds. A stall therefore costs ~1s, and only reports `tdiff: obsidian stalled on <file>, retrying` on stderr. Giving up entirely needs all 3 attempts to stall.
- Other transient failures retry too; a missing binary does not (`ObsidianError.retry`).
- If reads exceed `SLOW_NOTICE_AFTER` (2.5s), `prefetch()` writes `waiting on obsidian (N files)…` to stderr so slow never looks like hung.

Two related traps, both to do with wedged reads leaving a thread blocked:

- `die()` uses `os._exit()`, and `KeyboardInterrupt` is handled by `_on_uncaught()` which also uses `os._exit(130)`. A plain `sys.exit()` would hang at interpreter shutdown joining the non-daemon pool thread — which is exactly what an interrupted run used to do.
- Calls pass `stdin=DEVNULL`; a CLI that ever read the terminal would block every reader thread behind it.

Env overrides: `TDIFF_WORKERS` (default 4, `1` = serial), `TDIFF_TIMEOUT` (seconds), `TDIFF_DEBUG=1` (per-call timings on stderr).

## JSON output

`--json` writes one JSON object to stdout. Filters (`-T`, `-S`, `-I`, and the always-on project exclusion) are applied before serializing, so `rows` is exactly what text mode would print; `summary` always reflects those filtered rows (`--no-summary` is text-only). Statuses are bare chars — `"x"`, `" "`, `"/"` — not the bracketed `[x]` form used in text.

Diff mode:

```json
{
  "mode": "diff",
  "a": {"kind": "week", "label": "2026-W28", "start": "2026-07-05", "end": "2026-07-11",
        "files": ["2026-07-05", "...", "2026-W28"]},
  "b": {"kind": "day", "label": "2026-07-27", "files": ["2026-07-27"]},
  "rows": [
    {"type": "deleted", "name": "create `tday`", "status": ">"},
    {"type": "added",   "name": "write thoughts", "status": "*"},
    {"type": "same",    "name": "pack luggage", "status": "x"},
    {"type": "changed", "name": "complete [[2026-W28]]", "status": "x",
     "old_status": "x", "new_name": "complete [[2026-W29]]"}
  ],
  "summary": {"added": 61, "deleted": 17, "changed": 7, "same": 3, "total": 88}
}
```

- `status` is the *effective* status — B's for added/changed/same, A's for deleted — i.e. the one `-S` filters on and the one text mode prints.
- `old_status` appears on `changed` rows only; `new_name` appears only when pass-2 fuzzy matching paired two differently-worded names.
- `files` lists the vault files actually read for that side, which is where `-W` and `-w 0` exclusions are observable. An `-E` side has `"kind": "extern"` and carries `command` instead — it read nothing from the vault.

Dupes mode (`-D`):

```json
{
  "mode": "dupes",
  "a": {"kind": "day", "label": "2026-07-20", "files": ["2026-07-20"]},
  "duplicates": [
    {"status": "/", "exact": true, "count": 4,
     "variants": [{"name": "rework the templates", "count": 4, "statuses": ["&", "&", "&", "/"]}]}
  ]
}
```

`exact` mirrors the text sigils: `true` is `!!` (one wording repeated), `false` is `~~` (variant wordings clustered together). `status` is the cluster's winning status; `count` is total occurrences across all variants.

## Configuration

Task-status behaviour (dedup precedence, `-I`'s settled set, the hidden set, structural
"project" exclusion) is driven by TOML — not hardcoded in the script.

The status table describes the **vault's notation**, not this tool, so it lives at
`~/.config/obsidian-tasks/statuses.toml` — a directory neither `tdiff` nor `tcat` owns.
That is what lets the two share one table while neither depends on the other being
installed. Both ship a byte-identical `statuses.example.toml`; they must stay that way.

**Layers**, lowest precedence first — each *merges* over the ones below (`_merge()`:
tables merge, lists replace wholesale):

1. `~/.config/obsidian-tasks/statuses.toml` — the shared vault vocabulary
2. `~/.config/tdiff/config.toml` — tdiff-only overlay
3. `$TDIFF_CONFIG`
4. `--config PATH`

**Nothing is ever bootstrapped**, matching `tcat`. `DEFAULT_CONFIG_TOML` was deleted:
defaults belong in `statuses.example.toml`, which the user owns and edits, not in Python.
Unlike `tcat`, `tdiff` cannot degrade gracefully with *no* config — an empty
`PROJECT_STATUSES` leaks project headers into every diff — so zero layers is a hard
`parser.error`. A layer that exists but omits a section degrades with a `notice()`.

Keys read: `[dedup] priority`, `[roles] project|hide|settled`, `[vault]
daily_folder|weekly_folder`. `[order]` and `[theme.*]` are `tcat`'s and are deliberately
ignored here — in `tdiff` the diff type owns the row colour, so a status colour would
have nothing to paint.

```toml
[dedup]
# Tiers, highest precedence first; the inner list is a tie (broken by most recent day).
priority = [["x"], ["-"], ["!", "*", " ", "#", "o"], ["/"], ["&", "»", "«", ">", "=", "~"]]

[roles]
project  = ["p", "i", "u"]
hide     = ["&", "»", "«"]
settled  = ["x", "-", "&", "»", "«"]
```

**`[dedup]` is not `[order]` reused, and must not be merged into it.** `tcat`'s
`[order].statuses` is a display rank that sorts `x` *last*; `[dedup].priority` ranks `x`
*first*, because "done" is the truest thing you can say about a task also written as
`[/]` on Tuesday. Same char, opposite ends. A flat list also cannot express ties, and
this table has three.

`load_config()` populates `STATUS_PRIORITY`, `SETTLED_STATUSES`, `HIDDEN_STATUSES`,
`PROJECT_STATUSES`, `DAILY_FOLDER`, `WEEKLY_FOLDER`, `CONFIG_FOUND` right after arg
parsing, before anything reads them. `DEFAULT_IGNORE` was renamed `SETTLED_STATUSES` to
match the `[roles]` key it now comes from.

### Notices

`notice()` collects complaints; `flush_notices()` writes them to stderr at the very end,
via `finish()`, which every exit path goes through. Suppressed under `--json` and
`--no-summary` — both mean "output with nothing around it". `report_unlisted()` names
status chars a run met that no `[dedup]` tier ranks; it subtracts `PROJECT_STATUSES`
first, since those are excluded from output and never reach a tie-break.

## Date formats

`YYYY-MM-DD`, `today`, `yesterday`, `tomorrow`, `0` (today), `-N`/`+N`, a weekday name (`monday`..`sunday` or `mon`..`sun`, resolving **backwards** to the most recent occurrence at or before today), `w##` or `w2026-W##` (weekly planning file, e.g. `w20` = week 20 of current year). Negative numbers use an internal `__NEG__` token workaround to avoid argparse conflicts. Weekly file refs resolve to side type `('weekly_file', 'YYYY-W##')` and are supported in day-vs-day, `-w`, and `-D` modes (not `-o`). With `-F`, either side (day or weekly file) is further expanded to side type `('week', ...)` — its containing full Sun-Sat week.

**Week labels are named for the year the week *ends* in.** Week 1 is the week containing Jan 1, so the Sun-Sat week straddling New Year belongs to the later year: Sun 2026-12-27 → Sat 2027-01-02 is `2027-W01`, matching the vault templates' moment `gggg[-W]ww`. `week_span()` anchors its label and its Jan-1 reference on the **Saturday** for exactly this reason — anchoring on the Sunday (as it did until July 2026) yielded `2026-W53`, a file that never exists, and left the vault's real `2027-W01` unaddressable. Only one week per year is affected; every other week is identical under either rule, which is why the bug stayed invisible for so long. `_week_label_to_sunday()` was always correct and needs no matching change.

`week_span`, `resolve_week_label`, `_week_label_to_sunday` and `resolve_date` are
**vendored verbatim into `tcat`**, which checks them with its own
`tools/check-core-sync.sh` (that script lives in the `tcat` repo, not this one) against a
pinned commit here. Any change to them has to land in both tools together and the pin
bumped, or the two will disagree about which note to read.

`resolve_date` sits outside `tcat`'s marked vendor block in both files — it needs
`parser`, so it has to follow the argument parser — but it is checked all the same. The
promise that the two tools take the same date arguments is only true if it cannot drift.
`_WEEKDAY_NAMES` and `WEEKDAYS` are one-liners in both files specifically so the script's
constant check, which compares a single assignment line, actually covers them.

One deliberate divergence: `tcat` hard-errors on a future date without `-P` (the daily
note won't exist yet); `tdiff` accepts it, because an empty side is a legitimate diff.
