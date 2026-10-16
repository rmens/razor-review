# razor-review

Agent skill that takes pull requests from "works" to "a senior dev can read
it line by line".

By default it walks **all open PRs** of a GitHub repo. Pass one or more PR
links to limit it to those.

Per PR:

1. **Simplify pass**: four parallel reviewers (reuse, simplification,
   efficiency, altitude), fixes applied, behavior checked under non-default
   conditions.
2. **Review pass**: code review, silent-failure hunt, test analysis, comment
   analysis and type design, all in parallel.
3. **Codex second opinion** (`codex exec --yolo`, prompt on stdin, liveness
   check every 30 s) after each pass and before every push.
4. **Mutation check**: every behavior fix has a test that fails on the old
   code.
5. **Follow-ups**: out-of-scope findings become issues (after a duplicate
   check), and optionally codex PRs in separate worktrees.

## Install

```bash
git clone https://github.com/rmens/razor-review ~/.claude/skills/razor-review
```

## Use

```
/razor-review                                   # all open PRs of the current repo
/razor-review owner/repo                        # all open PRs of another repo
/razor-review https://github.com/owner/repo/pull/123
/razor-review review #123 focus on file permissions
```

Requires `gh` (authenticated) and `codex`. Commits are pushed only to your own
PRs; for other people's PRs it reports, and asks before pushing or commenting.
