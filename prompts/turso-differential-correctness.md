# Task: Turso vs SQLite semantic-equivalence correctness harness (in-tree)

Category: Other (backend) — conformance / verification tooling. In-tree extension of
existing test infrastructure. Not feature dev / debugging / testing-of-own-code.
Mode: Interactive (5+ turns, 6h+).
Repo: tursodatabase/turso. Development is centered INSIDE the source tree
(extend `simulator/` and `sql_generation/`, hook internal engine APIs). Not greenfield.

## Opening prompt (model-facing)

This is for the Rust rewrite, `tursodatabase/turso`. We've already got the simulator doing
deterministic runs against the engine's own invariants, and `sql_generation` produces the
workloads it feeds on. What's missing is the thing that actually catches silent wrong
answers: nowhere in there do we check a run against what SQLite would have returned.
Crashes and invariant violations we catch; wrong-but-doesn't-crash we don't.

I want you to build that into the existing setup rather than off to the side. Wire a real
SQLite in as an oracle so a simulated run can be checked against it, and lean on the
generator and the deterministic harness we already have instead of reinventing them. The
hard part you'll hit fast: a lot of the generated queries don't have one right answer to
compare against, ordering without ORDER BY, anything where the engine's allowed to make
its own choices, so a straight row-compare is either blind or all noise. You'll need
something smarter than equality to get at the real logic bugs in joins, aggregates,
where-clauses.

When it flags something it has to come out as a minimal, replayable case off the seed, not
a thousand-step session. And don't surface things already in COMPAT.md, those are known.

Work in-tree, make it build and run through the existing harness for real. I want actual
divergences coming out, not just plumbing.

## Grading / objective verifier (NOT shown to the model)

Deliverable is a diff against the Turso tree (new/modified files under `simulator/` and
`sql_generation/`, integrating internal APIs), not a standalone repo.

Success requires all of:
1. Builds and runs through the existing simulator/harness entry point against a fresh
   tursodatabase/turso checkout plus a real SQLite oracle. No mock/stub engine.
2. Seed-deterministic: a fixed seed reproduces an identical run and identical findings
   (reuses the existing DST determinism; must integrate it correctly, including the SQLite
   side and any new randomness).
3. >= N minimized, reproducible divergences that survive an independent re-run and are not
   in the COMPAT.md / known-issues denylist.
4. No false-positive oracle: a held-out set of semantically equivalent queries must not be
   flagged (catches a broken metamorphic/NULL implementation).
5. Preflight existence check done once before using the task: confirm >= N qualifying
   divergences are reachable, so success is guaranteed-reachable.

## Why it is hard

Work is centered in an unfamiliar, nontrivial Rust codebase: the model must understand the
existing simulator's deterministic execution model and the `sql_generation` crate, then
extend both and hook the engine's internal connection/execution APIs. The "no single right
answer" line is load-bearing: the only way to test those queries is metamorphic oracles
(TLP ternary partitioning, NoREC optimization-equivalence). TLP's three-valued-logic / NULL
handling is where models silently produce false positives that poison the report. Stacked
with: reusing (not reinventing) the in-tree generator and determinism, minimization that
preserves repro validity, COMPAT denylisting, root-cause dedup. Verifier runs the harness,
not the writeup, so partial credit is closed.

## Re-runner explainer

Goal in plain terms: extend Turso's own test infrastructure so a simulated run is checked
against real SQLite, to find where Turso quietly returns different answers (not crashes).
Report each disagreement as a tiny replayable case off the seed. Decisive part is
metamorphic checks for queries with no fixed expected answer. Work lives inside the source
tree (simulator/, sql_generation/), reusing the existing deterministic harness and
generator. Likely failures: broken metamorphic oracle (NULL/3VL), reinventing instead of
reusing the in-tree generator/determinism, never running through the real harness, weak
minimization, re-flagging known COMPAT gaps.

## Follow-up direction (interactive)

T1 push from planning/reading to a running integration through the existing harness. T2
steer from straight row-compare toward metamorphic oracles via the no-single-answer cases.
T3 hand back a false positive, make it fix oracle correctness. T4 demand minimization +
seed reproducibility through the existing DST machinery. T5 filter COMPAT gaps + dedupe
into a clean catalog.
