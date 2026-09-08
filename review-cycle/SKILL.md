---
name: review-cycle
description: Drive a PR to merge through an author/reviewer subagent loop — an independent reviewer agent posts a severity-tagged review as a PR comment, the author agent evaluates and responds or fixes, repeat until clean, then merge on green CI. Use after a subagent has opened a PR, or when asked to review-and-merge, run the review cycle, or take a PR to merge in an agentic flow.
---

# Review cycle

Two subagents, reused across rounds: one **author** (the agent that wrote the PR)
and one **reviewer** (independent). They alternate until the review is clean,
then the PR merges on green CI.

Reuse matters. Spawn each agent once and continue it with `SendMessage` for every
subsequent round, so neither rebuilds its understanding of the change from
scratch. A fresh reviewer each round re-reads the whole diff and re-litigates
settled points; a fresh author has to relearn its own PR.

## The loop

1. **Author** opens the PR. (Usually already done by the time this skill runs.)
2. **Reviewer** reviews the diff and posts the review as a PR comment.
3. If there are findings, the **author** evaluates each one and either fixes it or
   replies saying why not — as a PR comment.
4. **Reviewer** re-reviews, seeing its previous round and the author's responses.
5. Repeat from 3 until the reviewer signs off.
6. Merge on green CI.

## The reviewer

Spawn it as an independent subagent — it must not inherit the author's context,
or it will accept the author's framing along with it.

It checks for:

- **Bugs** — correctness, edge cases, error handling, concurrency, data loss.
- **Test coverage** — is the new behaviour actually tested, and would the tests
  fail without the change?
- **DRY** — duplicated logic that should be shared.
- **SOLID** — responsibilities, coupling, substitutability.
- **YAGNI** — speculative generality, unused abstraction, config nobody sets.
- **KISS** — complexity that does not earn its place.

Every finding carries a severity:

| Severity | Meaning |
|---|---|
| **Blocker** | Must not merge. Correctness, data loss, security, breaks a contract. |
| **High** | Should fix before merge. Real bug or a significant design problem. |
| **Medium** | Fix unless there is a reason not to. |
| **Low** | Worth doing, not worth blocking on. |
| **NIT** | Taste. Author may decline freely. |

The review is posted as a **PR comment** and **must start with the header
`REVIEWER AGENT`**. Findings should name file and line, state the problem
concretely, and say what would go wrong — not just cite a principle.

If there is nothing left to raise, the reviewer says so explicitly. Silence is
not a sign-off, and neither is a review made only of NITs with no verdict.

## The author

The author does **not** simply comply. It evaluates each finding and decides:

- **Agrees** → fix it, and say so.
- **Disagrees** → reply with the reasoning, in a PR comment. A reviewer can be
  wrong about intent, miss context in code it did not write, or propose a change
  that breaks something it has not read.

Reflexive agreement defeats the point of the cycle: it turns review into a second
author. A finding declined with a real reason is a good outcome.

Author replies also go on the PR as comments, so the disagreement and its
resolution are on the record next to the code.

## Merging

Merge when the reviewer has signed off and CI is green.

"Green enough" is permitted **only** for a known false-negative pipeline —
a check that is failing for reasons unrelated to the change and already known to
be broken. That is an exception, and it should be named explicitly when merging:
which check, and why it does not apply. A check nobody has looked at is not a
known false negative.

## Relationship to `pr-review`

`pr-review` is one deep pass by a single agent, delivered as a severity-ranked
artifact for a person to read. This skill is the automated loop: two agents,
findings posted to the PR itself, ending in a merge. Same lenses, different
shape. Use `pr-review` when a human wants the report; use this when the PR
should reach `main` without one.

## Notes

- Keep rounds tight. If the same finding survives two rounds unresolved, stop
  looping and put it to the user — the two agents are not going to converge.
- The reviewer reviews the PR, not the author. Findings are about the code.
- If the author's fix in round N introduces something new, the reviewer is
  expected to catch it in round N+1; do not skip the re-review because the
  previous round was clean.
