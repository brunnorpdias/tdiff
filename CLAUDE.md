# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project does

`tdiff` is a single-file Python CLI that diffs tasks between Obsidian daily notes (`<YYYY-MM-DD>`) and weekly notes (`YYYY-W##`) and reports what was added, removed, changed, or kept between two of them, grouped under the projects they belong to. A date names a day, a `w##` names a week, and `-D`/`-W` narrow a week to one of its two sources.

File lookups are folder-agnostic: `obsidian read file=<name>` resolves a file by its bare filename anywhere in the vault (wikilink-style resolution), so `tdiff` never hardcodes which folder daily notes or weekly notes live in. An optional `[vault]` section in the config file (see Configuration below) can pin explicit folder prefixes if name resolution ever becomes ambiguous.

External dependency: the `obsidian` CLI (note reading). Requires Python 3.11+ (uses stdlib `tomllib`).

## Running

```bash
# Run directly (no build step)
tdiff date_a [date_b] [flags]

# Common invocations during development
python3 tdiff 2026-04-14 2026-04-15 --no-color
python3 tdiff yesterday today --no-color -T s
python3 tdiff today -I --no-color          # today vs the rest of its week (both sources)
python3 tdiff today -D -I --no-color       # ...daily notes only
python3 tdiff today -W --no-color          # ...the weekly note only: plan vs actual
python3 tdiff wednesday -D --no-color      # the week stops *before* the anchor
python3 tdiff 2026-05-17 w20 --no-color    # a day vs the whole of week 20
python3 tdiff w20 --no-color               # a week's weekly note vs its own dailies
python3 tdiff w23 w24 --no-color           # full Sun-Sat week vs full Sun-Sat week
python3 tdiff w23 w24 -W --no-color        # weekly note vs weekly note
python3 tdiff today --json | jq .          # machine-readable output
python3 tdiff monday today --no-color      # weekday names resolve backwards
```

No test suite — testing is manual via CLI invocation.

**The regression check that matters most is dedup ordering**, because it is silent when
wrong. Capture `python3 tdiff today --json` before a change touching the config,
`materialize()` or the project scoping, and diff it after; it must be byte-identical
unless you meant otherwise. Scoping is the newest way to get this wrong: it may only ever
*add* rows (a task under two projects keeps a row in each), so a falling total means a
scope is closing too eagerly.

## Architecture

The entire tool lives in a single executable file: `tdiff` (~1170 lines, Python 3).

The script processes tasks in this pipeline:

1. **Read** — `_run_obsidian()` execs `obsidian read file=<name>` directly (no shell, no pipe), returning the note's **raw markdown**. Failures are fatal, never silent: a missing `obsidian` binary, a non-zero exit, an `Error:` line, or a stalled call all print `tdiff: ...` to stderr and exit 2. Only obsidian's `Error: File "X" not found.` is treated as an empty note (days without notes and missing weekly notes stay silent) — and only on the **first** line, because on raw markdown a note that happens to open a line with `Error:` would otherwise be misread as a failed read. `obsidian_lines()` memoizes per file path and `prefetch()` warms that cache concurrently (`FETCH_WORKERS` threads), submitting `obsidian_lines` itself so the vault read has exactly one seam — the concurrent path used to call `_run_obsidian` directly, leaving a second entry point that anything hooking the read would miss — every vault read for both sides of a comparison is issued in one batch, which is where most of the wall-clock went. See **Obsidian CLI stalls** below.

   **`read`, not `tasks`.** The flat task list `obsidian tasks` returns has already thrown away the headings, and the ladder markers under `### *Actio*` are the only thing that says which of a weekly note's tasks are the week's plan. `tcat` has always read raw markdown for that reason; moving `tdiff` onto it is what stopped the two tools disagreeing about what a note contains.

2. **Parse** — `parse_note()` turns markdown into `(indent, status_char, name, seq)` tuples; `_parse_obsidian()` adapts those to `(scope_key, scope_name, scope_status, base, status, day_index)` records. `parse_note` is **vendored verbatim into `tcat`** alongside the date core (see **Date formats**); `clean_text` used to be and no longer is (see **Task names** below). `#routine` lines are dropped unless `--routines` is passed. Statuses are stored **bracketed** (`'[x]'`, not `'x'`); `parse_note` yields the bare char, so `_parse_obsidian` re-brackets — every consumer downstream indexes `[1]` for the char, so a record source that forgets this fails silently rather than loudly.

   **Project grouping happens here, per note.** A task at indent 0 whose status is in `[roles] project` opens a scope and is not itself yielded; indented tasks join it; any other indent-0 task closes it and joins the bare scope (`scope_key = None`). Doing it inside the per-note loop is not incidental: a week side concatenates eight notes, and a header left open at the end of one would otherwise adopt the next note's children. `tcat`'s `build_groups` never faces that, seeing one note at a time.

   **How a note is read travels with it** as parse kwargs in `side_entries()`, because what a note means depends on which side asked for it, not on its name. `DAY_MODE` reads a whole daily note; `WEEK_DAY_MODE` reads one inside a week aggregation, minus `**future**`; `weekly_mode(until)` reads a `YYYY-W##` note's `### *Actio*` section. See **Reading the weekly note** below.

3. **Deduplicate** — `cluster_records()` uses union-find to merge task variants across days. Two bases are the same task if their token sets are identical OR one is a strict subset sharing the same first word. Rather than scanning all pairs, it buckets bases by token-set (equality merges) and by first token (subset merges) — same clusters, far fewer comparisons. `materialize()` picks the canonical form (most recent day, longest on tie) and winning status by `STATUS_PRIORITY`, which comes from `[dedup].priority` in the config.

   **Dedup is scoped.** It runs **once per project scope**, never once per side. Scopes never merge, exactly as in `tcat`: a task written both bare and under a project keeps a row in each, because the header is real context and a bare occurrence should not swallow a project's copy. `fetch_side()` returns `{scope_key: (display_name, header_status, tasks, order)}`. The invariant to hold onto is that scoping only ever *adds* rows.


4. **Week aggregation** (`side_entries()` / `week_entries()`, materialized by `fetch_side()`) — a `('week', (sunday, anchor, want_dailies, want_weekly))` side reads up to 7 daily notes plus the `YYYY-W##` weekly note at `day_index=0` (lowest dedup priority among the 7 days — losing ties against any daily note). `want_dailies` / `want_weekly` come from `-D`/`-W`; `anchor` bounds the week strictly before a date. Missing notes are silently skipped.

5. **Diff** — `diff_scope()` runs **per project scope**, over the union of both sides' scopes. Pass 1 does exact name matching; Pass 2 uses `best_match()` with Jaccard and prefix similarity at 0.7 threshold to relabel close add/delete pairs as "changed". Candidates carry precomputed token sets, and the expensive `prefix_sim()` (difflib) is skipped whenever it cannot change the outcome — different first char, or a similarity ceiling below the running best. Fuzzy matching never crosses a scope, so a task that moved between projects reads as deleted from one and added to the other rather than silently staying put; a scope only one side has yields all-deleted or all-added, which is how a project appearing or disappearing shows up at all.

6. **Output** — Project groups first, bare rows after. Projects sort by their own *header* status (`u` before `i` before `p`, from `[order].statuses`) and never by their children's — sorting a project by the statuses inside it would let one urgent task drag a whole project to the top. Rows inside a group go **type, then status, then name**: type outranks status because the question a diff answers is what moved, not what state it is in, and reading straight down the added block then the deleted block is the point of the tool. `TYPE_RANK` fixes that order as added, deleted, changed, same — the same order the summary line lists. Status orders within a type, which is where `[order].statuses` does its work; the lowercase name breaks the last tie. Coloured by diff type: deleted=RED, added=GREEN, changed=YELLOW, same=DIM; the project header is **uncoloured**, because the diff type belongs to the rows beneath it and a header dimmed to match nothing read as less structural than it is. A header prints only when a row under it survived the filters. Rows are built as dicts by `row()` and turned into text by `render()` at print time, so the text and `--json` modes are rendered from one source and can't drift. The summary line gains an `N projects` field, counting the headers that actually printed — it is text-only, since `--json` already names the project on every row.

## Key flags

A **date** names its daily note; a **`w##`** names its Sun-Sat week. A week has exactly
two sources — its seven dailies and its `YYYY-W##` weekly note — and `-D`/`-W` narrow it
to one. Naming neither reads both, which is why argparse makes them mutually exclusive:
together they would be a second spelling of the default. **`-D`/`-W` never promote a date
to its week**, so a command where no side is a week rejects them outright.

| Flag | Effect |
|------|--------|
| `-D` | Read only a week's seven daily notes, not its weekly note |
| `-W` | Read only a week's `YYYY-W##` weekly note, not its dailies |
| `-T SET` | Show only rows whose type is in a char set (`d`=deleted, `a`=added, `c`=changed, `s`=same, e.g. `dac`); prefix `^` to invert (e.g. `^s`). Summary counts reflect the filtered rows. Default (no `-T`): show all types, ordered by `[order].statuses`. |
| `-I` | Hide settled items — any row whose displayed status is in `[roles] settled`, whichever side it came from. |
| `--routines` | Include `#routine` tasks (excluded by default) |
| `-S SET` | Show only rows whose effective status is in a char set (e.g. `x-#`); prefix `^` to invert (e.g. `^x`). Filters on the displayed status (B's for added/changed/same, A's for deleted); summary counts reflect the filtered rows. Bare `-S -` needs `-S=-` |
| `--json` | Emit a JSON document instead of text (implies `--no-color`; `--no-summary` does not apply) |
| `--no-color` | Disable ANSI colors |
| `--no-summary` | Suppress summary line |

### Three principles the surface answers to

Worth stating, because each one decided a case that would otherwise look arbitrary:

- **One way to spell a command.** A flag that would be a no-op on a shape is rejected
  there, never quietly accepted. `tdiff w23 -D` errors because `tdiff w23` already *is*
  weekly-vs-dailies, and `tdiff yesterday today -D` errors rather than promoting two
  dates into one week.
- **Earliest on the left.** Where the user did not fix the order, the side that came
  first takes A: the weekly note precedes the week it plans, the days before a date
  precede that date. Two positionals are always taken in the order written.
- **"Up to" is exclusive.** A week derived from a date stops strictly before it.

### One positional: exclusion and truncation

With one positional the comparison is **that note against the week around it**:

```sh
tdiff today -I      # what is still open this week that is not on today's list
tdiff today -D -I   # ...ignoring what the weekly note planned
tdiff today -W      # ...only what the weekly note planned
tdiff w20           # a week's weekly note against its own dailies
```

Two rules do the work, and both live in `week_entries`' `anchor` parameter:

**A positional is never compared against itself.** The anchor leaves the week built
around it. A date drops out of the dailies. A `w##` drops out as the weekly note — which
leaves exactly the two sources, one per side, and is why a lone `w##` *rejects* `-D`/`-W`:
either would empty a side. The weekly note takes the A side because it is what was
written first.

**A week derived from a date stops strictly before it**, on both sources: the dailies run
Sunday→anchor−1, and `weekly_mode(until)` truncates Actio's whitelist to
`(None,) + WEEK_DAYS[:idx]`, one marker short of the anchor. Nothing dated after a day
can be outstanding as of it. This is not just about self-comparison: under the old
anchor-only exclusion `tdiff wednesday -D` read Thursday and Friday into the A side, and
that looked correct for a bare `today` purely because tomorrow's note is usually empty. A
`w##` has no date to stop at, so its week is read whole.

With two positionals neither side sits inside the other, so both are read whole, nothing
is excluded and nothing truncates. Two `w##` naming the same week are rejected: the
comparison would be a no-op.

### Reading the weekly note

A `YYYY-W##` note is read as its **`### *Actio*` section only** — the week's plan. The
rest (Impressio, Relatio, Cultus, Fixa) is structure and reference, not work assigned to
this week, and it used to leak into every week diff back when the reader returned a flat
task list with the headings already discarded.

Within Actio the day whitelist is `(None,) + WEEK_DAYS`, and **the leading `None` is the
one deliberate divergence from `tcat`'s `-A -P`.** Actio's seven `**weekday**` markers are
an *optional* second pass: a week is planned by listing tasks under the heading, and only
some of them ever get allocated to a named day.

Measured on this vault: an **organised** week allocates everything — W30 and W32 have zero
tasks sitting at `day=None` — so on those the `None` entry matches nothing and the two
tools agree exactly. A week still being **drafted** is the opposite: W33 has 37 unallocated
tasks and not one allocated, so asking for `WEEK_DAYS` alone reports an empty plan against
a full section. Reading the unallocated ones is what lets `tdiff` see a week you are still
writing. This is not a `tcat` bug — `-A -P` is a retrospective view of a finished week, and
that is a coherent thing to be.

`**future**` stays excluded — it is the deferral bucket, not this week — and the whitelist
does that on its own, since `future` is simply not in the tuple. Daily notes inside a week
aggregation exclude it too (`skip_future`), because folding seven days together would
otherwise pull deferred work in beside what actually happened, and a pre-split Sunday's
bucket holds a whole week of it. A **bare day side keeps its future bucket**: that is the
day's own list, and deferring something is part of the day.

### How the three status filters compose

`hide`, `-I` and `-S` all live in `suppressed()`, under one rule borrowed from `tcat`'s
`status_match()`: **a positive `-S` wins outright for the statuses it names.** So
`-S '»'` shows postponed rows even though `[roles] hide` lists them, and `-I -S x` shows
done rows rather than nothing. A negated `-S` says only what to drop, so `hide` and `-I`
still apply to everything it doesn't name.

All three ask about the row's **displayed** status (`eff`) and nothing else, which is what
makes them compose at all. `-I` used to be the exception, restricted to types 0 and 3 on
the theory that only deleted and same rows read their status off the settled A side — but
that left `++ [x]` and `~~ [/] → [x]` on screen, which is exactly the work `-I` exists to
get rid of. A task finished on B is as settled as one finished on A, so the restriction is
gone and `-I` now hides any settled row.

Filtering happens **after** dedup, so an earlier `[»]` never suppresses a later `[x]`.

## Projects

A task indented under a project header (`[roles] project` — `p`/`i`/`u` — at indent 0)
belongs to that project. `_parse_obsidian` turns that into a `scope_key`, and the scope is
a **dedup and diff scope**, not a decoration:

- **Scopes never merge.** A task written both bare and under a project keeps a row in
  each; the header is real context, and a bare occurrence should not swallow a project's
  copy. `--flat`-style collapsing does not exist here. The invariant that follows is that
  scoping only ever **adds** rows — a falling total is a bug, not a simplification.
- **The diff runs per scope**, fuzzy pass 2 included. A task that moved between projects
  therefore reads as deleted from one and added to the other. That is the intended
  reading, not an artefact: the header is part of what the note said.
- **A header is never a row.** It carries a project status, is not yielded by
  `_parse_obsidian`, and bypasses the `PROJECT_STATUSES` filter that still excludes a
  project status appearing on an ordinary (indented) row. It prints only when a row under
  it survived every filter, and it never enters the summary counts.
- Scope keys are **lowercased**, so `[[Project Alpha]]` and `[[project alpha]]` are one
  project; the display name is the first spelling seen, and `PROJECTS` prefers the B
  side's, matching the effective-status rule rows follow.

One consequence worth knowing before it surprises you: a header that differs only by a
wikilink anchor — `[[2026 master's applications#imperial]]` vs
`[[2026 master's applications]]` — is **two** scopes, so its tasks read as deleted from
one and added to the other. Nothing strips `#section` from a scope key. Whether it should
is a vault-convention question, not a code one.

## Task names

`clean_text` normalises a name in this order, and **the order is the whole point**:

1. undo Obsidian's `\[` / `\]` escapes — first, or `\[\[foo]]` never registers as
   bracket depth at all;
2. `strip_section_suffix` — drops a trailing ` – ...` **while the brackets are still
   there**, so its depth tracking can see that a dash inside `[[...]]` is part of a title;
3. `normalize_wikilinks` — keeps the brackets, shortens only the path
   (`[[a/b|c]]` → `[[b|c]]`);
4. `MDLINK_RE` — reduces `[label](url)` to `label`.

**This diverges from `tcat` on two counts, and `check-core-sync.sh` does not catch
either**, because `clean_text` and `parse_note` are still absent from its `FUNCS` list.
`tcat` reduces links *first* and strips the suffix last, which truncates any linked title
containing a dash — 18 of 455 names in this vault, some to a third of their length
(`read: [startup school – yc](…), [essays – pg](…)` became `read: startup school`). And
`tcat` reduces a wikilink to its display text where `tdiff` keeps the brackets, which is
115 of 455 names. `tdiff` is the correct side of both; reconciling `tcat` is a later
session's job, and until then this paragraph is the only thing recording the drift.

## Obsidian CLI stalls

Roughly **one `obsidian` call in a few hundred wedges and never returns** — measured at 180s with no output, while sibling calls kept answering in ~10ms. It happens at the same rate reading serially or concurrently (1/480 at 1 worker, 2/480 at 4, 4/480 at 8), so it is not caused by tdiff's threading; concurrency only widens the window because week modes issue 16 calls instead of 2. This is why week modes appeared to hang while day-vs-day rarely did.

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

`--json` writes one JSON object to stdout. Filters (`-T`, `-S`, `-I`, and the always-on exclusion of a project status on an ordinary row) are applied before serializing, so `rows` is exactly what text mode would print; `summary` always reflects those filtered rows (`--no-summary` is text-only). Statuses are bare chars — `"x"`, `" "`, `"/"` — not the bracketed `[x]` form used in text.

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
     "old_status": "x", "new_name": "complete [[2026-W29]]"},
    {"type": "added",   "name": "read luan's final report", "status": "!",
     "project": "[[forex fintech]]"}
  ],
  "summary": {"added": 61, "deleted": 17, "changed": 7, "same": 3, "total": 88}
}
```

- `status` is the *effective* status — B's for added/changed/same, A's for deleted — i.e. the one `-S` filters on and the one text mode prints.
- `old_status` appears on `changed` rows only; `new_name` appears only when pass-2 fuzzy matching paired two differently-worded names.
- `project` names the project a row sits under, and is **absent** on a bare row. `rows` stays flat rather than nesting children the way `tcat`'s envelope does: the filters and the summary then need no shape of their own, and grouping stays purely a rendering concern. `tcat`'s nested form had exactly one consumer, `-E`, which is gone.
- `files` lists the vault files actually read for that side. It is the only place `-D`/`-W`'s narrowing, the one-positional form's anchor-exclusion, and the anchor truncation are observable — none of them show up anywhere else in the output, which is what makes this field worth keeping.

`"mode"` is always `"diff"`. It used to distinguish a second `"dupes"` document, which is
gone — dupes belongs to `tcat`, and dropping it here is what freed `-D`.

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

**What still differs from `tcat`, and why.** `config_paths()`, `_merge()` and the
no-bootstrap rule are byte-identical modulo the tool name, and the layer list is the same
four. Three deliberate divergences remain, and none of them is drift:

- **Zero layers is fatal here, survivable there.** `tcat` degrades to unranked and
  uncoloured and says so; `tdiff` cannot, so it `parser.error`s. The config is a
  **required file**, not an optional one.
- **`tdiff` reads `[dedup]`, `tcat` reads `[theme.*]` and `[roles] full_row`.** Each tool
  ignores what it has no use for; the shared table is the superset.
- **`display_rank()` uses `tcat`'s `UNRANKED = 10 ** 6` sentinel**, same value, so the two
  order an unlisted status the same way rather than merely both "last".

Keys read: `[dedup] priority`, `[order] statuses`, `[roles] project|hide|settled`,
`[vault] daily_folder|weekly_folder`. `[theme.*]` is `tcat`'s and is deliberately ignored
here — in `tdiff` the diff type owns the row colour, so a status colour would have nothing
to paint. `[order]` used to be ignored for a weaker reason (rows just sorted by name) and
is now read: it is the one display convention both tools genuinely share, and a `tdiff`
that sorted differently from `tcat` made the same vault look like two.

```toml
[order]
# Display rank; position in the list is the rank, so `x` sorts last. Unlisted statuses
# rank after everything, as a block.
statuses = ["u", "!", "i", "*", ">", "=", "o", "p", " ", "/", "#", "~", "&", "»", "«", "-", "x"]

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
this table has three. Both are read now, at opposite ends of the pipeline: `[dedup]`
decides which status a row *carries*, `[order]` decides where that row *prints*. A task
deduped to `[x]` therefore sorts to the bottom, which is the point of having two keys.

`load_config()` populates `STATUS_PRIORITY`, `DISPLAY_ORDER`, `SETTLED_STATUSES`, `HIDDEN_STATUSES`,
`PROJECT_STATUSES`, `DAILY_FOLDER`, `WEEKLY_FOLDER`, `CONFIG_FOUND` right after arg
parsing, before anything reads them. `DEFAULT_IGNORE` was renamed `SETTLED_STATUSES` to
match the `[roles]` key it now comes from.

### Notices

`notice()` collects complaints; `flush_notices()` writes them to stderr at the very end,
via `finish()`, which every exit path goes through. Suppressed under `--json` and
`--no-summary` — both mean "output with nothing around it". `report_unlisted()` names
status chars a run met that the config does not rank, and reports the **two tables
separately** because a config can easily have one and not the other: missing from
`[dedup]` a status ranks 0 and loses every tie silently, missing from `[order]` it sorts
last, which just looks like a choice. It subtracts `PROJECT_STATUSES` from both, since
those are excluded from output and never reach a tie-break or a sort. `tcat` says the
same thing about `[order]` in its own `report_unlisted()`.

## Date formats

`YYYY-MM-DD`, `today`, `yesterday`, `tomorrow`, `0` (today), `-N`/`+N`, a weekday name (`monday`..`sunday` or `mon`..`sun`, resolving **backwards** to the most recent occurrence at or before today), `w##` or `w2026-W##` (a Sun-Sat week, e.g. `w20` = week 20 of current year). Negative numbers use an internal `__NEG__` token workaround to avoid argparse conflicts.

**A `w##` resolves to the whole week**, `('week', (sunday, anchor, want_dailies, want_weekly))`, not to the weekly note on its own — the `('weekly_file', …)` side kind is gone, folded into `('week', …)` with `want_dailies=False`. This is the one place the two tools' positional grammars now differ in *scope* rather than spelling, and it is what makes `tdiff w23 w24` a full-week comparison with no flag. A date resolves to `('day', …)` and is never promoted; `-D`/`-W` narrow a week that is already there.

**Week labels are named for the year the week *ends* in.** Week 1 is the week containing Jan 1, so the Sun-Sat week straddling New Year belongs to the later year: Sun 2026-12-27 → Sat 2027-01-02 is `2027-W01`, matching the vault templates' moment `gggg[-W]ww`. `week_span()` anchors its label and its Jan-1 reference on the **Saturday** for exactly this reason — anchoring on the Sunday (as it did until July 2026) yielded `2026-W53`, a file that never exists, and left the vault's real `2027-W01` unaddressable. Only one week per year is affected; every other week is identical under either rule, which is why the bug stayed invisible for so long. `_week_label_to_sunday()` was always correct and needs no matching change.

`week_span`, `resolve_week_label`, `_week_label_to_sunday` and `resolve_date` are
**vendored verbatim into `tcat`**, which checks them with its own
`tools/check-core-sync.sh` (that script lives in the `tcat` repo, not this one) against a
pinned commit here. Any change to them has to land in both tools together and the pin
bumped, or the two will disagree about which note to read.

`parse_note` (with `TASK_RE`, `H2_RE`, `H3_RE`, `MARK_RE`, `MDLINK_RE`, `WEEK_DAYS` and
`LADDER`) joined that set when the reader was unified — it came *from* `tcat` and is
byte-identical to its copy, but `tdiff` is the canonical side, so `check-core-sync.sh`
still needs it added to its `FUNCS` list and its `PIN` bumped. **Until that happens the
script does not cover it.**

`clean_text` is the same story with a sharper edge: it is **no longer identical**, and
nothing checks that. See **Task names** above for what diverged and why `tdiff` is the
correct side. Adding `clean_text` to `FUNCS` before reconciling `tcat` would fail the
check by design — which is the right order to do it in, since the failure is the point.

`resolve_date` sits outside `tcat`'s marked vendor block in both files — it needs
`parser`, so it has to follow the argument parser — but it is checked all the same. The
promise that the two tools take the same date arguments is only true if it cannot drift.
`_WEEKDAY_NAMES` and `WEEKDAYS` are one-liners in both files specifically so the script's
constant check, which compares a single assignment line, actually covers them.

One deliberate divergence: `tcat` hard-errors on a future date without `-P` (the daily
note won't exist yet); `tdiff` accepts it, because an empty side is a legitimate diff.
