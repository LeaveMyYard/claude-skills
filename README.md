# claude-skills

Personal Claude Code skills, kept in git so they can be shared across machines and accounts.

## Install on a new machine / account

```sh
git clone <this-repo> ~/Code/claude-skills
mkdir -p ~/.claude/skills
ln -s ~/Code/claude-skills/brief ~/.claude/skills/brief
ln -s ~/Code/claude-skills/pr-review ~/.claude/skills/pr-review
ln -s ~/Code/claude-skills/review-cycle ~/.claude/skills/review-cycle
ln -s ~/Code/claude-skills/desloppify-comments ~/.claude/skills/desloppify-comments
```

Claude Code follows symlinks in `~/.claude/skills/`, and loads a target only once
even if it is reachable from several locations. If `~/.claude/skills` did not exist
when the session started, restart Claude Code once. Verify with `/skills`.

Skills here are personal-scope: available in every project, and to subagents.

## Skills

- `brief` — compress current state into status + numbered decisions + next step.
- `pr-review` — deep PR review (bugs, DRY, YAGNI, SOLID, KISS, tests) published as a
  severity-ranked artifact with code snippets. Named `pr-review` to avoid clashing
  with the built-in `/code-review` command.
- `review-cycle` — drive a PR to merge through an author/reviewer subagent loop,
  severity-tagged reviews posted to the PR as comments, then merge on green CI.
- `desloppify-comments` — delete comments that say what the code does or how it works,
  keeping only why it must be this way and what breaks otherwise.

`pr-review` and `review-cycle` share review lenses but differ in shape: `pr-review`
is one deep pass producing a report for a person to read; `review-cycle` is an
automated loop between two agents that ends in a merge.
