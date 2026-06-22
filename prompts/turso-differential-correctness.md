# Task: Turso vs SQLite semantic-equivalence correctness harness (in-tree)

Category: Other (backend) — conformance / verification tooling. In-tree extension of
existing test infrastructure. Not feature dev / debugging / testing-of-own-code.
Mode: Interactive (5+ turns, 6h+).
Repo: tursodatabase/turso. Development is centered INSIDE the source tree
(extend `simulator/` and `sql_generation/`, hook internal engine APIs). Not greenfield.

## Opening prompt (model-facing)

This is for the Rust rewrite, `tursodatabase/turso`. The simulator already does
deterministic runs against the engine's own invariants and `sql_generation` produces the
workloads it runs. What's missing is the one that catches silent wrong answers: nothing in
there ever checks a run against what SQLite itself would have returned. Crashes and
invariant breaks we catch, wrong-but-doesn't-crash we're blind to, and that's the class
that actually burns users.

I want this built into the existing setup, not bolted on beside it. Bring a real SQLite in
as the oracle so a simulated run can be checked against it, and build on the generator and
the deterministic harness we already have rather than standing up your own. Whatever you
add has to run through the harness we've got and reproduce exactly off a seed the way the
rest of it does.

Here's the part that makes this real work and not an afternoon: a lot of the queries worth
running don't have a single right answer you can just compare against, row order when
there's no ORDER BY, anything where the engine is free to make its own call. So plain row
equality is useless, it'll either miss the bugs that matter or flag a hundred differences
that are all fine. I need it to actually get at logic bugs in the hard stuff, joins,
aggregates, nested where-clauses, and I need to trust a flag. If it cries wolf on queries
that are genuinely equivalent it's worse than nothing, I'll stop believing it on day one.

And a finding has to be usable. A thousand-statement session that disagreed somewhere isn't
a bug, I need it cut down to the smallest case that still shows it and replayable off the
seed so I can hand it to someone. Anything already written up in COMPAT.md I know about,
skip those.

Done, to me, looks like: it lives in-tree, it builds and runs through the existing harness
for real, and a run comes back with a short list of genuine wrong-answer cases against
SQLite, each one minimal and reproducible. I care a lot more about real divergences coming
out the far end than about how clean the plumbing looks.

## Overall goal (re-runner explainer)

The goal is to extend Turso's own test infrastructure so that a simulated run is checked
against a real SQLite, surfacing places where Turso returns different answers for the same
data and query. The focus is silent wrong answers, not crashes. Each disagreement should
come out as a tiny case that can be replayed exactly from a seed.

What a correct result looks like: the work lives inside the Turso source tree, mostly in
`simulator/` and `sql_generation/`, reusing the existing deterministic harness and query
generator rather than rebuilding them. A real SQLite is wired in as the oracle. The model
gets actual divergences out the other end, each minimized and reproducible, with known
COMPAT.md gaps filtered out.

The decisive, hard part is comparing queries that have no single correct answer to diff
against (ordering without ORDER BY, optimizer-dependent results, and so on). A strong
solution lands on metamorphic-style checks: transform a query into a form that must return
the same rows and compare those, instead of comparing to a fixed expected result. Common
families here are ternary-logic partitioning of the WHERE clause and optimization-
equivalence rewrites. Getting the NULL and three-valued-logic handling right is where this
goes wrong most often, and a slightly wrong oracle produces false positives that make the
whole report untrustworthy.

What gets viewed and modified: the existing simulator harness and generator, the engine's
internal connection/execution APIs, plus new code for the oracle, the legitimate-
nondeterminism normalization, and the minimizer. The model should not be changing the
database engine's behavior, only the test infrastructure around it.

Where models go wrong, most likely first: implementing the equivalence oracle incorrectly
so it flags equivalent queries as bugs (NULL/3VL), reinventing the generator and
determinism instead of reusing the in-tree ones, never actually running through the real
harness, weak or missing minimization, non-reproducible runs, and re-reporting known
COMPAT.md gaps as new findings.

## Grading / objective verifier (NOT shown to the model)

Deliverable is a diff against the Turso tree (new/modified files under `simulator/` and
`sql_generation/`, integrating internal APIs), not a standalone repo.

Success requires all of:
1. Builds and runs through the existing simulator/harness entry point against a fresh
   tursodatabase/turso checkout plus a real SQLite oracle. No mock/stub engine.
2. Seed-deterministic: a fixed seed reproduces an identical run and identical findings
   (reuses the existing DST determinism; must integrate the SQLite side and any new
   randomness correctly).
3. >= N minimized, reproducible divergences that survive an independent re-run and are not
   in the COMPAT.md / known-issues denylist.
4. No false-positive oracle: a held-out set of semantically equivalent queries must not be
   flagged (catches a broken metamorphic/NULL implementation).
5. Preflight existence check done once before using the task: confirm >= N qualifying
   divergences are reachable, so success is guaranteed-reachable.

These five (thresholds, held-out set, independent re-run, preflight) are the rating
mechanism and stay out of the prompt. The constraints they test (trustworthy flags,
minimal + replayable repros, in-tree + reuse harness, exclude COMPAT, real divergences out)
are all stated in the prompt in natural words so the rating maps to exact prompt language.

## Why it is hard

Work is centered in an unfamiliar, nontrivial Rust codebase: the model must understand the
existing simulator's deterministic execution model and the `sql_generation` crate, then
extend both and hook the engine's internal connection/execution APIs. The "no single right
answer" line is load-bearing: the only way to test those queries is metamorphic oracles
(ternary-logic partitioning, optimization-equivalence rewrites), and the prompt deliberately
states that difficulty without naming the technique. The NULL/3VL handling is where models
silently produce false positives that poison the report. Stacked with: reusing (not
reinventing) the in-tree generator and determinism, minimization that preserves repro
validity, COMPAT denylisting, root-cause dedup. Verifier runs the harness, not the writeup.

## Follow-up direction (interactive)

T1 push from planning/reading to a running integration through the existing harness. T2
steer from straight row-compare toward an equivalence oracle via the no-single-answer cases
(without naming the technique). T3 hand back a false positive, make it fix oracle
correctness. T4 demand minimization + seed reproducibility through the existing DST
machinery. T5 filter COMPAT gaps + dedupe into a clean catalog.
