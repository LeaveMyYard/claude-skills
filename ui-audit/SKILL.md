---
name: ui-audit
description: Audit a web app's real rendered UI against its design and against hard visual-quality rules, then fix what is found, through a multi-agent flow (state matrix → automated checks → independent visual review → component-level fix PRs → CI gate). Use when asked to audit, polish or "find everything wrong" with how an app looks on desktop/mobile, when a design port looks off, or before shipping UI to customers. Judges screenshots and measurements, never the code or copy alone.
---

# UI audit

Earlier "audits" compared strings and structure ("is the element there, is the copy right") and missed what a
person sees in ten seconds: touching elements, a desktop page sized for a phone, missing skeletons, weak contrast,
jumpy transitions. This skill makes the audit about **the rendered result**, in every state, at real sizes.

## Non-negotiable rules (pass these to every subagent verbatim)

1. **Judge the rendered screenshot or a measurement, never the code or copy.** "Looks right in the DOM" is not
   evidence.
2. **Desktop first.** Primary viewports 1440×900 and 1920×1080; also 1280, 2560, 1024, 768, 390. Light and dark.
3. **Every screen in every state:** empty (new account), realistic data, heavy data (long names, many rows, big
   numbers), loading (API slowed to 2–3 s), error (500/403/timeout), each role. A screen only checked with seed
   data is not checked. Always include the **first-login states** a new customer actually sees: a brand-new
   account, one test item, a handful of items below every threshold, and sample/demo data switched on, each at
   1440, 1920 and the widest viewport.
4. **Fix at the token or component level.** A spacing bug on one page is almost always a bug in a shared
   container, card, header or token. One-off pixel nudges on a page are rejected in review.
5. **Evidence or it didn't happen.** Every finding: screen, state, viewport, theme, a cropped screenshot, the rule
   it breaks, severity (S1 broken/unreadable, S2 clearly off, S3 polish). Every fix PR: before/after screenshots at
   1440 and 1920 in light and dark, plus the automated checks green.
6. **Direction changes go to the owner.** New layouts, new empty-state designs, new motion language, moving a
   control to a new place: design options first (the project's design flow), the owner picks. Clear defects
   (spacing, contrast, alignment, missing skeleton, broken state) are fixed directly.
7. **No debug or design-tool artefacts in the product** (theme toggles from mockups, grid overlays, "TODO" copy,
   lorem ipsum, placeholder avatars).
8. **Reproducible:** run browsers in the project's pinned Playwright container, fixed fonts, deterministic seed
   data, frozen clock where time is shown.

## Flow

### 1. Matrix (one agent)
- Enumerate every route from the router (not from memory), every role, and every state in rule 3. Find the
  project's seed/demo data command and its API mock or slow-down hooks; add a dev-only latency/error injection if
  none exists.
- Capture: full-page and viewport screenshots per route × state × viewport × theme; a short video (or frame
  strip at 60 fps) for each navigation, menu, dialog, toast, tab switch and data load; the matching design
  artboard where one exists (same size).
- Output a browsable index (static HTML or an artifact) the reviewers and the owner can open.

### 2. Automated checks (hard pass/fail, run per screen and state)
| Check | Rule |
|---|---|
| Touching / cramped | sibling or adjacent boxes with < the smallest spacing token between them (excluding intentional joins: segmented controls, table cells) |
| Off-scale values | computed margin/padding/gap, font-size, line-height, radius, colour not in the design tokens |
| Desktop width use | at ≥1440 the main content column < ~60 % of the viewport, or a fixed phone-width max-width on a desktop page |
| Readability | body text < 13 px on desktop (< 12 px for dense tables/captions), line-height < 1.3 for body |
| Contrast | WCAG AA: 4.5:1 text, 3:1 large text, icons, borders of inputs and focus rings; plus a full axe scan |
| Loading | with API slowed, a skeleton/placeholder within 150 ms; no spinner-only full pages; CLS < 0.05 when data lands |
| Empty / error states | title + one-line explanation + a next action; no raw error strings, no blank panels |
| Motion | list every transition/animation: duration and easing within the motion tokens; only transform/opacity animated; no animation on first paint of data tables; `prefers-reduced-motion` respected |
| Alignment | icon+label pairs vertically centred within 1 px (compare glyph box centre to icon centre); baseline alignment across a row |
| Affordances | logo links home; every clickable has pointer cursor, hover and visible focus; hit targets ≥ 24 px desktop / 44 px touch |
| Overflow | clipped or overflowing text (scrollWidth > clientWidth without an ellipsis/tooltip design), horizontal page scroll |
| Debug artefacts | rule 7 |
| Page grid | top-level blocks of a page share the same left and right edges, or sit on the page's declared grid columns; flag any block whose right edge differs from its siblings' by more than one spacing token |
| Rule density | two horizontal rules/borders closer than the section-spacing token (a table's header rule + column-label rule, stacked card tops, a chart rule under a tile row) |
| Repeated placeholders | the same empty/"not enough data" text in two or more sibling tiles or panels; one shared empty state is expected instead |
| Consistent figures | the same metric shown in two places on a screen (page, sidebar, header) must agree, and copy must not refer to UI that isn't on screen |
Write these as a reusable test suite in the project (Playwright + axe + small DOM/computed-style lints), not as
throw-away scripts: step 6 turns them into a CI gate.

### 3. Visual review (independent agents, fresh context)
- Two reviewers per screen group, each given only the screenshots, videos, the design artboards and this rubric —
  not the code and not each other's notes. Score 1–5 and list findings for: hierarchy, balance and density;
  alignment and spacing rhythm; typography (scale, weight, line length); colour and contrast; consistency with
  other screens; quality of empty/loading/error states; motion (from frame strips); fidelity to the design.
- **Whole-page first impression, judged separately from components.** One reviewer per screen × state sees only
  the full viewport screenshot at 1920 and the widest size, cold, and answers: does this look finished and
  balanced to a paying customer seeing it for the first time? Anything below 4/5 is a finding against the page's
  composition (grid, density, hierarchy), even when every component on it passes. Components passing is not
  evidence that the page passes.
- A finding survives if both reviewers raise it, or a third adjudicator confirms it from the evidence.
- Compare with the design artboards side by side; list every deviation as "intentional?" for the owner when the
  design and the product disagree and neither is clearly wrong.

### 4. Triage into component clusters
Group surviving findings by root cause (e.g. page container width, card spacing, header, type scale, empty-state
component, skeleton component, motion tokens, table density, dialog). One GitHub issue per cluster with the
evidence, affected screens and the fix approach. Show the owner the cluster list and the S1/S2 counts before the
fix wave starts; flag the rule-6 items that need design options.

### 5. Fix wave
One author agent per cluster (parallel where clusters don't share files), each PR through the project's review
loop (e.g. review-cycle). The reviewer judges the before/after screenshots, re-runs the automated checks on every
affected screen (not just the one in the issue), and rejects page-level patches for component problems.

### 6. Gate
Add the automated checks and a visual-regression snapshot set (primary viewports, both themes, key states) to CI
for the app. Snapshot updates must be deliberate and reviewed with images. Re-run steps 1–3 after the wave and
report the before/after scores per screen.

## Report to the owner
Per cluster: what was wrong (one image), what changed, before/after scores; the list of rule-6 decisions still
open; what the CI gate now enforces.
