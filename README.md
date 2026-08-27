# claude-skills

Personal Claude Code skills, kept in git so they can be shared across machines and accounts.

## Install on a new machine / account

```sh
git clone <this-repo> ~/Code/claude-skills
mkdir -p ~/.claude/skills
ln -s ~/Code/claude-skills/brief ~/.claude/skills/brief
ln -s ~/Code/claude-skills/pr-review ~/.claude/skills/pr-review
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
