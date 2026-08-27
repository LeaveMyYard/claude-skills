---
name: pr-review
description: Deep review of a GitHub PR (bugs, DRY, YAGNI, SOLID, KISS, test coverage) delivered as a severity-ranked Claude artifact with code snippets. Use when the user asks to review a PR / branch / diff and wants a report or artifact, e.g. "review this PR", "review PR 514 and create an artifact with issues ranked by severity". For posting inline findings to the PR itself, the built-in /code-review may fit better; this skill produces the shareable report.
---

# PR Review → severity-ranked artifact

Review the whole PR yourself — read every line of the diff — then publish one
artifact page with findings ranked by severity, each backed by a code snippet.
The artifact is the deliverable; the chat message is a short verdict + link.

## 1. Gather

```sh
gh pr view <N> --repo <owner>/<repo> --json title,body,author,files,state,url,baseRefName,headRefName
gh pr diff <N> --repo <owner>/<repo> > <scratchpad>/pr<N>.diff
```

Read the diff in full, in chunks (Read with offset/limit). Do not sample or
delegate the reading for PRs under ~10k diff lines — the best findings come from
cross-file connections that samplers miss. When surrounding context is needed
(a function the diff calls, a config the code reads), fetch the file:

```sh
gh api repos/<owner>/<repo>/contents/<path> --jq '.content' | base64 -d
```

## 2. Review lenses

Run every lens over the whole diff, not one lens per file:

- **Bugs / correctness** — logic errors, dead branches, swallowed errors,
  behavior changes not mentioned in the PR description.
- **DRY** — copy-pasted blocks, the same literal at N call sites, parallel
  files that should share a template/helper.
- **YAGNI** — vestigial return values, dead config values, code kept "just in
  case" after its consumer was removed.
- **SOLID** — functions acquiring second responsibilities (a `validate_*` that
  now also loads and mutates), leaky abstractions.
- **KISS** — stale comments contradicting code, copy-paste docs, leftover
  empty sections, scope creep of the PR itself.
- **Tests** — deleted tests whose behavior still exists, new features with zero
  tests, tests that only verify their own fixtures (circular), weakened
  assertions.
- **Config / deploy** — hardcoded namespaces/hosts/ports, per-environment
  drift, secrets, TLS flags.

The highest-value techniques, in order of how often they find real bugs:

1. **Cross-file consistency.** When the PR changes a pattern in one place,
   grep the diff for every sibling that should have changed the same way. A
   migration applied to service A but not service B, a chart that hardcodes
   what its sibling chart takes from values — these are the merge-blockers.
2. **Code ↔ config compatibility.** When code changes an interpreter (a
   delimiter, a schema, a topic format), check every config/data file the PR
   ships against the *new* interpretation, including files added by the same PR.
3. **Deleted test = deleted behavior?** For every removed test, ask whether
   the behavior it pinned was intentionally removed or silently lost.
4. **Sample data vs. rules.** When the PR ships both matching rules and sample
   payloads, verify the rules actually match the payloads (key casing, nesting).
5. **Comments vs. code.** A removed or contradicted comment ("can't use
   nonroot because X") often marks a regression the author forgot about.

## 3. Rank

- **Critical** — blocks merge: broken at runtime, broken deploy, data loss.
- **High** — real defect or risk, but survivable or environment-specific.
- **Medium** — should fix; design debt, missing tests, risky pins.
- **Low** — cleanup, docs, nits, process notes.

Tag each finding with its lens (Bug / DRY / YAGNI / SOLID / KISS / Tests /
Config / Security). Findings that are high-confidence reads of the diff but
need a runtime check to confirm get an extra **Verify** tag — never present
those as confirmed. Every finding ends with a one-line **Fix**.

## 4. Build the artifact

Load the `artifact-design` skill before writing HTML. Structure:

- Header: repo eyebrow, PR title, link, author, size, review date; severity
  count chips.
- Verdict box: approve / request changes, with the 2–3 blocking reasons.
- One section per severity band; each finding is a card with a colored
  severity stripe, category tags, prose (what breaks, why), and **code
  snippets**.
- Footer noting what "Verify" means.

Snippet rules — this is what makes the report comprehensible:

- Every finding (except pure process notes) gets at least one snippet in a
  `<figure>` with a `<figcaption>` showing the exact file path.
- Use diff coloring (red `-` / green `+` full-width lines) when the PR's own
  change is the problem; use an amber `mark` highlight on the exact offending
  token when the problem is a line that exists.
- Add short inline `# comments` in the snippet explaining why the marked line
  fails — the snippet should be readable without the prose.
- Where one file does it right and a sibling does it wrong, show both snippets
  back to back ("broken vs. correct sibling"). This is the most convincing
  snippet form.
- Snippets are hand-marked spans (no external highlighter — CSP blocks CDNs):
  line spans `display: block; white-space: pre`, tokens `.mk` (mark), `.cm`
  (comment), `.st` (string), `.kw` (keyword), lines `.add` / `.del`.
- `overflow-x: auto` on the `pre`; never let the page body scroll sideways.
- Define all snippet colors as CSS tokens with light and dark values (bare
  `:root`, `@media (prefers-color-scheme: dark)` guarded with
  `:root:not([data-theme="light"])`, and `:root[data-theme="dark"]`).

Publish with the Artifact tool from a scratchpad file; keep the same file path
on every update so the URL is stable; pick one favicon and keep it.

## 5. Deliver

Final chat message: verdict first, the critical findings in 2–4 sentences
each, one line on what the mediums/lows cover, the artifact link, and an offer
to post comments to the PR. If the user asks for PR comments: short,
human-toned, use "-" not em dashes, lead with the hold/approve stance.
