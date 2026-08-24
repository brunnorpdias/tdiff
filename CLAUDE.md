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

## Sharing with `tcat`

Everything both tools need lives in **`tnotes`** (`~/Projects/tnotes/tnotes.py`,
~830 lines), symlinked to `~/.local/lib/tnotes.py` the same way both scripts are
symlinked into `~/.local/bin/`. There is no packaging and no install step; `tdiff`
starts by putting that directory on `sys.path` and importing:

```python
sys.path.insert(0, str(Path.home() / '.local' / 'lib'))
import tnotes as tn
tn.init('tdiff')
```

A missing symlink is caught and reported in a sentence, because a bare `ImportError`
traceback says nothing about what to do.

**This replaced vendoring, which had failed in the way vendoring always does.** `tcat`
carried a marked copy of the shared block and `tools/check-core-sync.sh` compared it
against a pinned `tdiff` commit. The pin went stale (`92f195c`, two commits behind by
the end), `parse_note` and `clean_text` drifted, and the check never covered
`materialize` or `load_config` at all — so the two tools quietly disagreed about which
status a deduped task carries and about what a missing config means. **The check script
is deleted**; the drift it existed to catch cannot occur.

What moved: name normalisation (`clean_text` and friends), `parse_note` and its
regexes, the whole vault reader (`obsidian_lines`, `prefetch`, `die`, the stall
handling), config loading, the date core, and `cluster_records`/`materialize`. What
stayed: the diff itself, the week/side model, the filters, rendering, and argparse.

**Three seams the module needs that the vendored copies did not**, each of them a place
the two tools genuinely differ:

- **Tool identity.** `tn.init(tool)` names the caller. Every message the module writes
  is prefixed with it, and `config_paths()` reads it to find that tool's overlay and its
  `$<TOOL>_CONFIG`. The environment variables are the *module's* (`TNOTES_*`) because
  they are read at import, before `init()` has been called.
- **`resolve_date` raises rather than reports.** It used to call `parser.error`, which
  is exactly why it sat outside `tcat`'s vendor block and drifted: the message belongs
  to the caller's argument parser, the one thing a shared module cannot own. It now
  raises `tn.DateError` and each CLI catches it.
- **`parse_note` takes `only_days` as well as `skip_days`.** `tdiff` bounds a week by
  dropping days at or after an anchor; `tcat -W <date>` wants the opposite, only that
  weekday's allocation. Both are the caller's to supply, so neither is a whitelist the
  module holds.

`load_config(…, required=)` is the fourth difference and is now an argument rather than
two divergent copies — see **Configuration**.

## Architecture

`tdiff` itself is one executable file (~620 lines, Python 3) on top of `tnotes`.

The script processes tasks in this pipeline:

1. **Read** — `_run_obsidian()` execs `obsidian read file=<name>` directly (no shell, no pipe), returning the note's **raw markdown**. Failures are fatal, never silent: a missing `obsidian` binary, a non-zero exit, an `Error:` line, or a stalled call all print `tdiff: ...` to stderr and exit 2. Only obsidian's `Error: File "X" not found.` is treated as an empty note (days without notes and missing weekly notes stay silent) — and only on the **first** line, because on raw markdown a note that happens to open a line with `Error:` would otherwise be misread as a failed read. `obsidian_lines()` memoizes per file path and `prefetch()` warms that cache concurrently (`FETCH_WORKERS` threads), submitting `obsidian_lines` itself so the vault read has exactly one seam — the concurrent path used to call `_run_obsidian` directly, leaving a second entry point that anything hooking the read would miss — every vault read for both sides of a comparison is issued in one batch, which is where most of the wall-clock went. See **Obsidian CLI stalls** below.

   **`read`, not `tasks`.** The flat task list `obsidian tasks` returns has already thrown away the headings, and a heading is what an exclude list names; the day markers it also drops are what say when a planned task was due. `tcat` has always read raw markdown for that reason; moving `tdiff` onto it is what stopped the two tools disagreeing about what a note contains.

2. **Parse** — `parse_note()` turns markdown into `(indent, status_char, name, seq)` tuples; `_parse_obsidian()` adapts those to `(scope_key, scope_name, scope_status, base, status, day_index)` records. `parse_note` and `clean_text` live in `tnotes`, shared with `tcat` (see **Sharing with `tcat`**). Lines carrying a tag named in `[exclude] tags` are dropped. Statuses are stored **bracketed** (`'[x]'`, not `'x'`); `parse_note` yields the bare char, so `_parse_obsidian` re-brackets — every consumer downstream indexes `[1]` for the char, so a record source that forgets this fails silently rather than loudly.

   **Fenced blocks are skipped.** ` ``` ` / `~~~` toggle `in_fence`, and nothing
   inside is parsed — not tasks, not headings, not day markers. Fencing is one way a
   vault freezes a task list, a whole section of it at a time. `parse_note` used to ignore fences outright, which was harmless only
   while the reader was `obsidian tasks` — that CLI never handed the fenced lines over.
   Reading raw markdown made them live tasks again: `tdiff 2026-05-17 2026-05-18`
   reported 89 rows where 19 are real, the other 70 being that day's frozen copy of the
   week. Every fence in the vault is balanced and none nests, so a plain toggle is
   enough; an unbalanced one would swallow the rest of a note, and nothing detects that.

   **`[exclude] sections` is the filter for everything that is not fenced**, and the
   durable one: a section is skipped because of what it is called, not because of how
   it happens to be formatted. It applies to every note, daily and weekly — `exclude`
   rides on every `parse_note` call — and takes everything nested inside a named
   section with it. **`[exclude] tags` is the same idea for a line rather than a
   section** — a tag is the other way a vault marks a line as not-really-a-task, and
   the match ends on a word boundary so `#routine` does not also take `#routine/daily`.
   `--all` ignores the whole `[exclude]` table for one run; it does **not** lift the
   fence.

   It is also the *only* filter of its kind. A weekly note used to be pinned to its
   one heading and to a whitelist of day markers, both spelled into the source, which
   decided on the tool's behalf that a plan lives under a heading written one
   particular way. A plan parked in a sibling section was then unreachable however the
   config was written. Both whitelists are gone; a note is read whole and the config
   says what to leave out. **No section name appears anywhere in the script.**

   **The outline is headings only.** `HEAD_RE` (`^(#{1,6})\s+(.+?)\s*$`) makes every
   heading a node, at its own level. `_norm_heading` strips emphasis, backticks and
   case, so `## *Old Notes*` and `## old notes` are one name. `_excluded` matches a
   pattern as an ancestor chain with intervening levels allowed, so inserting a heading
   above one does not break it. A section ends at the next heading at the same or
   shallower level, and the opening line is swallowed so an excluded section cannot set
   the day ladder either.

   **Bold text was briefly a node too, and should not have been.** It read well —
   `actio > future` addressing a `**future**` marker — until you ask where such a
   section *ends*. Bold carries no level: two in a row are siblings, one meant to nest
   inside another is indistinguishable from one following it, and a vault that bolds an
   ordinary paragraph grows a section by accident. It also needed a fence-nesting hack
   to stop a marker inside a fenced block ending the section above it. A heading has a
   level and therefore an unambiguous end; that is the whole argument.

   `HEAD_RE` replaced a pair of matchers (`H2_RE`/`H3_RE`) that wanted a single
   italicised word and drove the section whitelist. They could not see a two-word
   heading or an unemphasised one — precisely the sections a config needs to name — so
   widening was never the fix; deleting the whitelist they fed was. `MARK_RE` is gone
   with them: which lines are day markers now comes from `[days]`, matched by
   `_match_day`.

   **Project grouping happens here, per note.** A task at indent 0 whose status is in `[roles] project` opens a scope and is not itself yielded; indented tasks join it; any other indent-0 task closes it and joins the bare scope (`scope_key = None`). Doing it inside the per-note loop is not incidental: a week side concatenates eight notes, and a header left open at the end of one would otherwise adopt the next note's children. `tcat`'s `build_groups` never faces that, seeing one note at a time.

   **How a note is read travels with it** as parse kwargs in `side_entries()`, because what a note means depends on which side asked for it, not on its name. `DAY_MODE` reads a whole daily note, and one inside a week aggregation too — identically, because dropping `**future**` there hardcoded both that such a bucket exists and what it means; `weekly_mode(until)` reads a `YYYY-W##` note whole too, adding only the anchor's day bound. See **Reading the weekly note** below.

3. **Deduplicate** — `cluster_records()` uses union-find to merge task variants across days. Two bases are the same task if their token sets are identical OR one is a strict subset sharing the same first word. Rather than scanning all pairs, it buckets bases by token-set (equality merges) and by first token (subset merges) — same clusters, far fewer comparisons. `materialize()` picks the canonical form (most recent day, longest on tie) and winning status by `STATUS_PRIORITY`, which comes from `[dedup].priority` in the config.

   **Dedup is scoped.** It runs **once per project scope**, never once per side. Scopes never merge, exactly as in `tcat`: a task written both bare and under a project keeps a row in each, because the header is real context and a bare occurrence should not swallow a project's copy. `fetch_side()` returns `{scope_key: (display_name, header_status, tasks, order)}`. The invariant to hold onto is that scoping only ever *adds* rows.


4. **Week aggregation** (`side_entries()` / `week_entries()`, materialized by `fetch_side()`) — a `('week', (sunday, anchor, want_dailies, want_weekly))` side reads up to 7 daily notes plus the `YYYY-W##` weekly note at `day_index=0` (lowest dedup priority among the 7 days — losing ties against any daily note). `want_dailies` / `want_weekly` come from `-D`/`-W`; `anchor` bounds the week: the dailies stop strictly before the date, the weekly note keeps that date's own allocation. Missing notes are silently skipped.

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
| `--all` | Ignore the whole `[exclude]` table — sections and tags alike — for one run. Fenced blocks are still skipped |
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
- **"Up to" is exclusive — of the *notes*, not of the plan.** A week derived from a date
  reads dailies Sunday→date−1, so nothing is compared against itself. The weekly note's
  allocation *for* that date is kept: it is not a copy of the day's list, it is what the
  day was supposed to be, and it is the only thing that makes a task the plan set for
  today and never carried over show up at all.

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

**A week derived from a date stops before it, and the two sources stop at different
places.** The dailies run Sunday→anchor−1: the anchor's own note is the B side, so
reading it into A would compare it against itself. `weekly_mode(until)` passes
`skip_days = WEEK_DAYS[idx + 1:]`, dropping only what is allocated *after* the anchor —
nothing dated after a day can be outstanding as of it.

**The anchor's own allocation is kept, and that asymmetry is the point.** It used to be
dropped too, for symmetry with the dailies, which reads as principled and is wrong: the
weekly note's block for today is not a copy of today's list, it is what today was
*supposed* to be. Dropping it made every task the plan set for today read as `added`,
and silently hid the one case `tdiff today` exists to catch — a task the plan assigned
to today that never made it onto today's list, which now shows as `deleted`. On
2026-08-24 that was the difference between `11 added · 2 same` and `2 added · 11 same`.

This stayed invisible until `[days]` worked. `MARK_RE` matched `**monday**` and not the
wikilink form, so no day ever opened and the weekly note contributed its whole plan
whatever the anchor — the right output for entirely the wrong reason. This is not just about self-comparison: under the old
anchor-only exclusion `tdiff wednesday -D` read Thursday and Friday into the A side, and
that looked correct for a bare `today` purely because tomorrow's note is usually empty. A
`w##` has no date to stop at, so its week is read whole.

With two positionals neither side sits inside the other, so both are read whole, nothing
is excluded and nothing truncates. Two `w##` naming the same week are rejected: the
comparison would be a no-op.

### Reading the weekly note

A `YYYY-W##` note is **read whole**, exactly like a daily one. It used to be pinned to
one heading and, within that, to a whitelist of day markers — both spelled into the
script. Each decided on the tool's behalf where a plan lives, so a plan parked under a
heading the source had never heard of was unreachable however the config was written.
Nothing about a vault's vocabulary is in the script now: not a section name, not a
marker spelling.

What survives is the **day ladder**, and only to answer one question: was a task
allocated to a day at or after the anchor? `weekly_mode(until)` returns
`{'skip_days': WEEK_DAYS[idx + 1:]}` and nothing else; with no anchor it returns `{}`.

**`[days]` says what a marker looks like.** The keys are the seven canonical day names,
because the code has to order them to know what "at or after" means; every value is a
glob the vault supplies, matched by `_match_day` against the whole lowercased line. A
day runs until the next marker or the next heading, and a task under no marker is never
dropped — most of a week being planned is unallocated, and an unallocated task has no
day to be late for.

A value may be **a list of globs**, which is what makes a vault's own history readable
after a notation change. This one is mid-migration: `2026-W32` writes `**tuesday**` and
`2026-W35` writes `[[2026-08-25|tuesday]]`, so both spellings are listed and both weeks
read. That is also how the ladder came to be silently dead — `MARK_RE` matched
`**monday**` and nothing else, so the wikilink form scanned as ordinary text and every
weekly note contributed its whole plan whatever the anchor.

**With no `[days]` nothing ever opens a day**, so a weekly note is never bounded by the
anchor. This is not an error and earns no notice: the daily notes still truncate, since
their bound is the filename rather than anything inside them, and a vault that never
allocates tasks to days has nothing to configure. It is worth knowing rather than
guessing at, which is why it is written here.

`WEEK_DAYS` is the one thing left, and it is a list of days of the week, not a claim
about notation. The old `LADDER` — `('promissum',) + WEEK_DAYS + ('future',)` — carried
two vault-specific names and is gone with the whitelist that used it.

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

**This once diverged from `tcat` on two counts.** Both are moot now — there is one
copy, in `tnotes` — but the history is worth keeping because it says which side was
canonical, and why the vendoring it replaced could not hold. `tcat` reduced links *first* and stripped the suffix last,
which truncates any linked title containing a dash
(`read: [startup school – yc](…), [essays – pg](…)` became `read: startup school`) — 18 of
455 names in this vault, some to a third of their length. And `tcat` reduced a wikilink to
its display text where `tdiff` keeps the brackets, 115 of 455 names. `tdiff` was the
correct side of both, which is why the shared copy is `tdiff`'s.

## Obsidian CLI stalls

Roughly **one `obsidian` call in a few hundred wedges and never returns** — measured at 180s with no output, while sibling calls kept answering in ~10ms. It happens at the same rate reading serially or concurrently (1/480 at 1 worker, 2/480 at 4, 4/480 at 8), so it is not caused by tdiff's threading; concurrency only widens the window because week modes issue 16 calls instead of 2. This is why week modes appeared to hang while day-vs-day rarely did.

Because the wedge is **per-invocation** and the app stays healthy, the fix is a short deadline plus a fresh call, not a longer wait:

- `OBSIDIAN_TIMEOUT` (default 1s, `TNOTES_TIMEOUT=N`) — a healthy call is ~10ms (20ms worst of 320 measured), so this is ~100x headroom, and it keeps a stall inside the one second the whole tool should take. A merely slow call isn't lost by the tight deadline; the retry catches it.
- `OBSIDIAN_ATTEMPTS` (3) — a retry after a stall practically always succeeds. A stall therefore costs ~1s, and only reports `tdiff: obsidian stalled on <file>, retrying` on stderr. Giving up entirely needs all 3 attempts to stall.
- Other transient failures retry too; a missing binary does not (`ObsidianError.retry`).
- If reads exceed `SLOW_NOTICE_AFTER` (2.5s), `prefetch()` writes `waiting on obsidian (N files)…` to stderr so slow never looks like hung.

Two related traps, both to do with wedged reads leaving a thread blocked:

- `die()` uses `os._exit()`, and `KeyboardInterrupt` is handled by `_on_uncaught()` which also uses `os._exit(130)`. A plain `sys.exit()` would hang at interpreter shutdown joining the non-daemon pool thread — which is exactly what an interrupted run used to do.
- Calls pass `stdin=DEVNULL`; a CLI that ever read the terminal would block every reader thread behind it.

Env overrides: `TNOTES_WORKERS` (default 4, `1` = serial), `TNOTES_TIMEOUT` (seconds), `TNOTES_DEBUG=1` (per-call timings on stderr). The prefix is the module's rather than either tool's: they are read at import, before `tn.init()` has been told who is calling, and a vault reader that behaved differently per tool would be a fresh way for the two to disagree.

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

The config describes the **vault**, not this tool, so it lives in `~/.config/tconfig/` —
a folder neither `tdiff` nor `tcat` owns. That is what lets the two share it while
neither depends on the other being installed. `tnotes` ships the example files; the
tools do not carry copies, so there is no pair to keep byte-identical.

It is split **by concern, not by tool**, because that is the axis along which a file
actually changes: `notation.toml` is rewritten when the vault changes how it writes a
note, `statuses.toml` when a status changes meaning. Splitting by tool would have meant
writing the same section twice and letting the two copies drift, which is the vendoring
mistake in config form.

**Layers**, lowest precedence first — each *merges* over the ones below (`_merge()`:
tables merge, lists replace wholesale):

1. `~/.config/tconfig/notation.toml` — `[exclude]`, `[days]`, `[vault]`
2. `~/.config/tconfig/statuses.toml` — `[order]`, `[dedup]`, `[roles]`, `[theme.*]`
3. `~/.config/tconfig/tdiff.toml` — tdiff-only overrides
4. `$TDIFF_CONFIG`
5. `--config PATH`

`$TCONFIG_DIR` relocates the folder. The env var's name comes from `tn.init(tool=…)`, so
`tcat` reads `$TCAT_CONFIG` from the same code.

**The pre-`tconfig` layout is a fallback, not a layer.** If `tconfig/` holds none of the
first three files, `config_paths()` reads `~/.config/obsidian-tasks/statuses.toml` and
`~/.config/<tool>/config.toml` instead, with one `notice()` naming the new home — so the
first run after the move is not a hard error. A `tconfig/` that exists wins outright,
because merging two homes would let a half-migrated setup pick up settings from a file
the user thought they had replaced.

**Nothing is ever bootstrapped.** `DEFAULT_CONFIG_TOML` was deleted: defaults belong in
the example files, which the user owns and edits, not in Python.

**`required=` is the one place the two tools differ, and it is now an argument.**
`tdiff` calls `tn.load_config(…, required=True)` — an empty `PROJECT_STATUSES` leaks
project headers into every diff, so zero layers is a hard `parser.error`. `tcat` passes
`False` and degrades to unranked and uncoloured, saying so. It used to be a difference
between two copies of the function, which is to say an accident waiting to be
reconciled the wrong way. A layer that exists but omits a section degrades with a
`notice()` either way.

Keys read: `[dedup] priority`, `[order] statuses`, `[roles] project|hide|settled`,
`[exclude] sections|tags`, `[days] sunday..saturday`, `[vault] daily_folder|weekly_folder`.
`[theme.*]` is `tcat`'s and is deliberately ignored here — in `tdiff` the diff type owns
the row colour, so a status colour would have nothing to paint. The merged table is the
superset and each tool reads what it has a use for. `[order]` used to be ignored for a
weaker reason (rows just sorted by name) and is now read: it is the one display
convention both tools genuinely share, and a `tdiff` that sorted differently from `tcat`
made the same vault look like two.

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
`PROJECT_STATUSES`, `EXCLUDED_SECTIONS`, `EXCLUDED_TAGS`, `DAY_PATTERNS`, `DAILY_FOLDER`, `WEEKLY_FOLDER`, `CONFIG_FOUND` right after arg
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

`YYYY-MM-DD`, `today`, `yesterday`, `tomorrow`, `0` (today), `-N`/`+N`, a weekday name (`monday`..`sunday` or `mon`..`sun`, resolving **backwards** to the most recent occurrence at or before today), `w##` or `w2026-W##` (a Sun-Sat week, e.g. `w20` = week 20 of current year), or `w0`/`w-1`/`w+1` for a week relative to this one. Negative numbers use an internal `__NEG__` token workaround to avoid argparse conflicts.

**The relative form is checked before the bare number, and the order is load-bearing.** `w0` matches `_WEEK_SHORT_RE` too, where it would resolve to `YYYY-W00` — a label no calendar produces. `_WEEK_REL_RE` is therefore tried first, between the full and short forms. `w-1` needs no `__NEG__` handling: it does not match `-\d+`, so the interception leaves it alone and argparse reads it as a positional rather than a flag. That the `w` prefix is what separates `w-1` (last week) from `-1` (yesterday) is precisely why the offset flags could be dropped.

**A `w##` resolves to the whole week**, `('week', (sunday, anchor, want_dailies, want_weekly))`, not to the weekly note on its own — the `('weekly_file', …)` side kind is gone, folded into `('week', …)` with `want_dailies=False`. This is the one place the two tools' positional grammars now differ in *scope* rather than spelling, and it is what makes `tdiff w23 w24` a full-week comparison with no flag. A date resolves to `('day', …)` and is never promoted; `-D`/`-W` narrow a week that is already there.

**Week labels are named for the year the week *ends* in.** Week 1 is the week containing Jan 1, so the Sun-Sat week straddling New Year belongs to the later year: Sun 2026-12-27 → Sat 2027-01-02 is `2027-W01`, matching the vault templates' moment `gggg[-W]ww`. `week_span()` anchors its label and its Jan-1 reference on the **Saturday** for exactly this reason — anchoring on the Sunday (as it did until July 2026) yielded `2026-W53`, a file that never exists, and left the vault's real `2027-W01` unaddressable. Only one week per year is affected; every other week is identical under either rule, which is why the bug stayed invisible for so long. `_week_label_to_sunday()` was always correct and needs no matching change.

`week_span`, `resolve_week_label`, `_week_label_to_sunday` and `resolve_date` all live
in `tnotes` now, so the two tools cannot disagree about which note to read. `tdiff`'s
only local piece is a four-line `resolve_date` wrapper that catches `tn.DateError` and
hands the message to its own `parser.error` — see **Sharing with `tcat`**.

One deliberate divergence: `tcat` hard-errors on a future date without `-P` (the daily
note won't exist yet); `tdiff` accepts it, because an empty side is a legitimate diff.
