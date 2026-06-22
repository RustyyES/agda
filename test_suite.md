# Test Suite — Prompt Authoring Rules

Standing checklist. Run every new prompt through this before delivering.

## What the prompt MUST be
- Reads like real engineering communication (Slack message / ticket), not a polished spec.
- Substantial work: senior/staff level, keeps a frontier model busy 1.5+ hours (target the investigation phase alone at ~6 hours).
- Grounded in real work: if a repo is assigned, the task must force engaging with that codebase (refactor, debug, extend, integrate). If no repo, it must be ambitious build-from-scratch (real service/library, not a toy).
- Goal is clear; approach is open. Multiple valid approaches fine.
- Conversational, sometimes incomplete, assumes shared context. Natural and human, not LLM-perfect punctuation/grammar. Professional and serious but not contrived.
- Realistic: something a real Turso customer or maintainer might actually ask, even if only theoretically.
- Verifier is easy/objective even when the solution path is hard.

## What the prompt MUST NOT be
- No enumerated requirements, acceptance criteria, or Step 1/2/3 procedures.
- Not so vague the goal is unclear ("fix the problem in this repo").
- No changes to files/apps the model cannot access.
- No write access to a GitHub repo (no pushing commits, no Actions).
- No publicly available PR that already solves it.
- No meta commentary, no traps, no "goblin" language. Calm, confident, substantial.

## Specificity calibration
- Too vague: "We need a notifications system. Can you build it?"
- Reasonable: states goal + key constraints, leaves the how open.
- Too specific: line numbers, exact edits, full test-case lists.
- Reasonable: describe the symptom and the desired outcome, let the model find the fix.

## Interactive mode (this is interactive)
- Conversation should reach 5+ meaningful turns over 1.5+ hours.
- Opening prompt can lean loose; interpretation work is part of the eval.
- Follow-ups: respond naturally to what the model produced. Point out mistakes, clarify misunderstandings, ask for refinement.
- Do NOT pivot mid-task or bolt on unrelated features. Follow-ups refine the original task.

## Deliverable alongside each prompt (for non-expert re-runners)
Plain English, concise, human, NO em dashes, no LLM giveaways. Cover:
- Overall goal (do not just restate the prompt).
- What the ideal finished outcome looks like; behaviors that change.
- What is expected to be viewed/modified/removed/added (files, functions, components).
- For review-only tasks: which issues/suggestions the model should surface.
- Areas the model is likely to miss or get wrong.
- Interactive: the intended direction of the follow-up turns.

## Domain focus for this batch
"Other" category (backend), per the working taxonomy: activities that are not Debugging, DevOps, Code Review, Feature Dev, Product/UX, Refactoring, Testing, System Design, Requirements, or (code/dep) Migration. Anchor in real Turso DB / SQLite-rewrite engineering.
