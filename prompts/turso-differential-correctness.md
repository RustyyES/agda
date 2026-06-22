# Task: Turso vs SQLite semantic-equivalence correctness harness

Category: Other (backend) — verification tooling, not feature dev / debugging / testing-of-own-code.
Mode: Interactive (5+ turns, 6h+).
Repo (read-only for the model): tursodatabase/turso.

## Opening prompt (model-facing)

This is for the Rust rewrite, `tursodatabase/turso`, not libSQL. We've already got
reasonable infra for the crash and I/O-fault side of things, the simulator hammers the
engine against its own invariants and `sql_generation` feeds it. What we don't have, and
what keeps coming back to us as user reports, is anything that systematically checks we
return the same answers SQLite would. Wrong-but-doesn't-crash is the category I trust
least right now.

I want a standalone correctness harness that uses a real SQLite as the oracle and goes
hunting for queries where Turso disagrees with it. Build schemas, build data, build
queries, run both, compare. The reason this isn't a weekend script: a big chunk of the
interesting queries don't have a single "right" answer you can diff against directly, row
order without an ORDER BY, anything where the result legitimately depends on choices the
engine is allowed to make. A naive diff either sails right past real bugs or buries you in
false positives. The harness has to be cleverer than straight comparison to actually shake
out logic bugs in joins, where-clauses, aggregates, the parts with subtle semantics.

And when it finds something it has to be useful. A 400-statement random session that
disagrees somewhere is not a bug report. I need it cut down to the smallest case that
still shows the problem, and I need to re-run it and get exactly the same thing back. Stuff
already in COMPAT.md I know about, don't bother re-flagging those.

It has to build and run against a fresh checkout for real, linking the actual engine, not a
mock or a stub. Take it as far as you can. I'd much rather see real divergences fall out the
end than a gorgeous framework that's never once been run.

## Grading / objective verifier (NOT shown to the model)

Success requires all of:
1. Builds and runs end to end against a fresh tursodatabase/turso checkout and a real
   SQLite, linking the actual engine.
2. Seed-deterministic: a fixed seed reproduces an identical session and identical findings.
3. >= N minimized, reproducible divergences that survive an independent re-run and are not
   in the COMPAT.md / known-issues denylist.
4. No false-positive oracle: a held-out set of semantically equivalent queries must not be
   flagged (catches a broken metamorphic/NULL implementation).
5. Preflight existence check done once before using the task: confirm >= N qualifying
   divergences are reachable, so success is guaranteed-reachable.

## Why it is hard

The "no single right answer" line is load-bearing: the only way to test those queries is to
reinvent metamorphic oracles (TLP ternary partitioning, NoREC optimization-equivalence).
TLP's three-valued-logic / NULL handling is where models silently produce false positives
that poison the report. Stacked with: valid deep-SQL generation, automated reduction that
preserves repro validity, full seed-determinism, real engine linkage, COMPAT denylisting,
root-cause dedup. Verifier re-runs the tool, not the writeup, so partial credit is closed.

## Re-runner explainer

See chat delivery / repo history. Goal in plain terms: find where Turso quietly returns
different answers than real SQLite, report each as a tiny reproducible case. Decisive part
is metamorphic checks for queries with no fixed expected answer. Likely failures:
broken metamorphic oracle (NULL/3VL), trivial or broken SQL generation, never running
against the real engine, weak minimization, non-reproducible runs, re-flagging known
COMPAT gaps.

## Follow-up direction (interactive)

T1 push from planning to building+running. T2 steer from differential-only toward
metamorphic oracles via the no-single-answer cases. T3 hand back a false positive, make it
fix oracle correctness. T4 demand minimization + seed reproducibility. T5 filter COMPAT
gaps + dedupe into a clean catalog.
