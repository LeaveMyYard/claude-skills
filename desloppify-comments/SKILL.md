---
name: desloppify-comments
description: Strip AI-slop comments and docstrings from code — delete anything that says what the code does or how it works, keeping only why it must be this way and what breaks otherwise. Use when the user says comments are too verbose, overloaded, "captain obvious", that a PR is mostly comments, or asks to clean up / trim / desloppify comments or docstrings. Also use before handing an agent-written PR to human reviewers.
---

# Desloppify comments

Agent-written code is badly overcommented, and the usual failure is not inaccuracy —
it is that the comment restates the code. It reads as a transcript of the reasoning that
produced the line, not as information for someone reading the line later.

## The rule

**A comment may not say what the code does, or how it does it. Only why it must be this
way, or what breaks otherwise.**

**The default is no comment.** Add one only if omitting it would lead someone to make a
wrong change. Then one line. Two only if the reason needs a condition attached. Three or
more needs a stated reason.

Test: delete the comment, then ask whether a competent reader gets the same information
from the line and its identifiers. If yes, it stays deleted.

## Delete; never relocate

Moving prose into a docstring is not a cut. If content fails the rule it goes — it does
not migrate.

This is the trap to watch for, because it looks like progress. On one 13-PR stack, three
passes instructed to "compress, don't delete" moved total prose by **186 lines out of
6,700** — comments fell 60 lines while docstrings *grew* 38, because every cut was
relocated. Measure total prose before and after, not the comment count.

## Worked examples

Five lines to two, keeping only what a reader might otherwise "fix":

```python
# Deliberately unhashed: six digits are guessable anyway, and confirm only matches
# within the caller's own claim.
confirmation_code: Mapped[str | None] = mapped_column(String(CODE_DIGITS), nullable=True)
```

Cut: "The six digits shown on the claimed phone" (the field name), "Not unique, because
collisions are ordinary" (justifies a visible absence), "`_activate` clears it in the same
statement that leaves pending" (mechanism — belongs on `_activate` or nowhere).

Seventeen comment lines to six in a settings module. Cut: every justification of a chosen
value ("12 hours covers a long shift plus handover"; a four-line argument for why a TTL is
five minutes — that belongs in the ADR), and every mechanism statement ("Refused at
startup, not once per send" when the `Field` pattern says so; "Absolute http(s) only" when
the type is `AnyHttpUrl`; "prepended to the path above" when adjacency says so). Kept: that
a TTL is *idle* rather than absolute, and a no-navigation-guard consequence explaining an
otherwise unreadable regex.

## Never delete

- A consequence: what breaks, what silently degrades, what an editor would get wrong.
- A fact invisible at the site: another service's behaviour, a library or platform quirk,
  a browser bug, a DB engine detail, an external timeout.
- A warning that stops someone reintroducing a fixed bug, where the mechanism is subtle.
- Schema-drift blind spots, lock ordering and acquisition order, partial-index arbitration,
  fail-closed auth reasoning.

## Exemptions

- **Section-divider banners** in test files (`# ----- # The two legs -----`). Navigation,
  not explanation. Leave them. An identifier-overlap detector will flag these as
  restatements; they are false positives.
- **Scenario descriptions for concurrency or interleaving tests**, where the body is
  fixtures and the interleaving is not visible in it. Compress, do not cut.

## Decision records

Keep the pointer, drop the restatement. `(ADR 013)` or `(ADR 010 decision 6)` is enough —
the record owns the argument. Never keep a paragraph explaining why a chosen value is
right; that is the record's job, and duplicating it means two places to update.

## Docstrings

Same rule. One-line summary, then at most two or three lines of why. Delete `Args:` /
`Returns:` sections that restate the signature and types, and paragraphs narrating internal
mechanics.

These phrases each mean a second claim was bolted on — grep for them:
`anyway`, `as well as`, `not a second way of`, `which matters more`, `the whole point of`,
`and it is also`.

Also delete, wherever it appears:

- **Stack-relative phrasing** — "at this point in the stack", "before X existed", "not `''`
  any more". True only while a chain is unmerged.
- **Claims about another PR's state** — "has not merged", "today's answer", "…yet". These go
  false on merge day.
- **Explaining by contrast with a version that never shipped.** Grep the history before
  trusting one: a comment justifying a design against "the anonymous router it replaced"
  described something that had never existed in the repo.
- **Authoring-session evidence** in PR descriptions — "green locally", "on the same
  machine", "seen five times as…".

## Finding the work

Judge only what the PR introduces. Attribute with `git blame` plus
`git merge-base --is-ancestor <commit> origin/main`, and skip anything already on `main` —
pre-existing slop is a separate decision.

Three mechanical detectors, in yield order:

```python
# 1. Attached comment blocks of 3+ lines (highest yield by far).
#    A run of comment-only lines whose next non-blank line is code.
# 2. Decision-record restatement: blocks of 3+ lines matching /\bADR\s*0?\d/.
# 3. Oversized docstrings: triple-quoted blocks over 8 lines.
```

Also measure the prose ratio per file — comment + docstring lines over non-blank lines.
Anything over ~35% needs a pass; healthy is 12-18% for source.

Do **not** rely on identifier-overlap detection to find worthless comments. It sounds
right and yields almost nothing: on one stack it produced 11 candidates of which 7 were
section banners and 0 were real. The defect is rarely a short restatement — it is a long
block of true statements where two of seven claims matter. That needs judgement per claim,
not per comment.

## Procedure

1. Run the detectors; report counts and the prose ratio before starting.
2. Judge claim by claim, not comment by comment. A block may keep one sentence and lose four.
3. For stacked PRs: work bottom-up, one commit per PR, rebase each onto the fixed tip below
   it before editing (`git rebase --onto <new base> <old base>`), push with
   `--force-with-lease`.
4. Commit message: **one line**, imperative, no body, no attribution.
5. Verify.

## Verify

- Linters and formatters for every touched language; type checks too.
- **Before shortening any docstring, grep its text.** Some ship verbatim into generated
  output (`openapi.json`) and some are asserted on by tests. A passing OpenAPI-export hook
  confirms the spec did not move. Migration module docstrings can surface in
  `alembic history`.
- Confirm the diff is prose-only: no executable line changed. State that you checked.
- Report prose lines before and after with the percentage cut, and how many comments were
  deleted outright versus shortened.

## Report

Give the numbers, then two lists that matter more than the numbers:

- anything kept at three or more lines, and why
- **anything deleted you are less than fully confident about**, explicitly, so it can be
  reviewed

A wrong length call costs readability. A wrong deletion loses a fact permanently, and some
comments that look redundant are not — one that reads like it restates a `WHERE` clause was
guarding a deadlock.

Expect 40-45% of prose to go on agent-written code.
