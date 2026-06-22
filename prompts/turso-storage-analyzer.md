# Task: Turso in-tree storage / space analyzer

Category: Other (backend) — storage / forensic analysis tooling. In-tree.
Not feature dev / debugging / testing / correctness-verification.
Mode: Interactive (5+ turns, 6h+).
Repo: tursodatabase/turso. Development centered INSIDE the source (core/storage + a new
tool/CLI subcommand). Not greenfield.

Why this axis: the correctness/differential/metamorphic space is already saturated in-tree
(testing/differential-oracle, testing/sqlancer, testing/simulator, conformance, sqltests),
which made the earlier differential task a cheat/assembly job. Storage space-analysis is a
real gap: tools/ has only dbhash, `.dbinfo` is not implemented, dbstat is incomplete. The
on-disk format is SQLite-compatible (core/storage/sqlite3_ondisk.rs, btree.rs, pager.rs),
so SQLite's sqlite3_analyzer / dbstat is an authoritative objective oracle.

## Opening prompt (model-facing)

This is the Rust rewrite, `tursodatabase/turso`. We keep the on-disk format compatible with
SQLite, the b-tree layout, the page types, all of that lives in `core/storage`. What we
don't have is any way to actually look at how a database is spending its space. The SQLite
world has `sqlite3_analyzer` and the dbstat stuff for exactly this, where did the bytes go,
how many pages is each table and index eating, how much of that is real data versus
overhead versus dead space. We've got nothing, `.dbinfo` isn't even wired up, and people
putting real data on us are starting to ask.

I want a proper storage analyzer, in-tree. You point it at a database and it walks the whole
thing and reports, per table and per index, the page counts, how the space breaks down,
payload versus overhead versus unused, the depth of the trees, the full picture. Build it on
top of our own storage layer in `core/storage`, not as a separate thing that parses the file
blind, it should be reading the database the way the engine does, including anything that's
only made it into the WAL so far.

What I actually care about: the numbers have to be right. Not roughly right on a clean little
test database, dead right on the messy real-world ones, databases that have been live a long
time, not something freshly created for a test. If a column in my report is subtly off I'd
rather not ship it at all, a storage tool whose numbers you can't trust is worse than having
nothing. Handy thing is SQLite's own analyzer will give you the truth for the same file, so
you've always got something to hold yourself against.

Make it build and run in-tree for real, against actual database files. I want to point it at
one and get numbers I'd stake a capacity decision on.

## Overall goal (re-runner explainer)

The goal is a new in-tree tool for Turso that analyzes a database file and reports where its
space is going: per table and per index, how many pages, how those bytes split between
actual payload, structural overhead, and unused or dead space, plus b-tree depths and
overall totals. It reads through Turso's own storage layer in `core/storage` rather than
parsing the file as an outside observer, and it accounts for state that currently lives only
in the WAL. The work lives inside the Turso source, most likely a new tool under `tools/` or
a CLI subcommand, plus whatever traversal/accounting code it needs.

What a correct result looks like: the analyzer walks every page through the pager, correctly
identifies what each page is and how its bytes are used, and produces a report whose key
figures match what SQLite's `sqlite3_analyzer` (and the dbstat virtual table) report for the
same file, across a range of databases, not just a clean toy one. Treat SQLite's analyzer as
ground truth and get the numbers to line up.

The hard part, which the prompt deliberately does not enumerate, is the long tail of the
SQLite storage format. Clean rowid b-trees are easy. The numbers drift once you account for
overflow page chains and the payload-versus-overflow threshold, freelist trunk and leaf
pages left by deletes, pointer-map pages under auto_vacuum, WITHOUT ROWID tables and the
different index cell formats, the per-page reserved-bytes region, fragmented free bytes
inside a page, the lock-byte page, varint serial-type sizing, and reconciling WAL frames
against the main file. A strong solution handles all of these and matches the oracle on
databases that exercise them; most attempts get the happy path and miss several.

Where models go wrong: missing accounting cases so numbers drift off the oracle on
non-trivial databases, parsing the file blindly instead of going through core/storage,
ignoring WAL state, or delegating computation to sqlite3_analyzer/dbstat instead of
computing natively. No change to engine behavior, only added tooling.

## Grading / objective verifier (NOT shown to the model)

Success requires all of:
1. Builds and runs in-tree against real database files.
2. Native computation only: does not shell out to or delegate to sqlite3_analyzer / dbstat
   for the figures.
3. Reads through core/storage (the pager), and reflects latest committed state including WAL
   frames, not a blind external file parse.
4. Key figures (per-btree page counts, payload/overhead/unused bytes, overflow pages, tree
   depth, totals) match sqlite3_analyzer within tolerance across a HELD-OUT battery of
   databases spanning the hard cases (auto_vacuum/ptrmap, overflow-heavy blobs, WITHOUT
   ROWID, post-delete freelist, large indexes, non-zero reserved bytes, uncheckpointed WAL).
5. Preflight: confirm the oracle is reproducible and the battery is solvable before use.

The thresholds, held-out battery, and oracle mechanism stay out of the prompt. The
constraints they test (numbers must be dead-right on messy real databases, in-tree, via
core/storage, WAL-aware, trustworthy) are stated in the prompt in natural words.

## Follow-up direction (interactive)

T1 from reading core/storage to a running analyzer matching the oracle on a simple DB. T2
hand it a database where its numbers don't reconcile with sqlite3_analyzer; make it find why
the totals are off without naming the cause. T3 a second awkward case. T4 correctness on
uncheckpointed WAL state. T5 tighten the report into something every column is trustworthy.
