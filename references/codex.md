# Codex: running, watching, prompting

`S` is the session scratchpad. Always write prompts to a file and feed them on
stdin (`-`); never inline a long prompt as an argument.

## Run

```bash
git diff > $S/before.patch            # to prove codex changed nothing
codex exec --yolo -C "$REPO" -o $S/codex_last.md - < $S/codex_prompt.md \
  > $S/codex_run.log 2>&1; echo "EXIT=$?" >> $S/codex_run.log
```

Run it with `run_in_background: true`. `-o` writes only the final message, so
no need to grep the log for it.

## Watch (Monitor, every 30 s)

```bash
L=$S/codex_run.log
while true; do
  sleep 30
  if grep -q '^EXIT=' "$L" 2>/dev/null; then echo "codex finished: $(grep '^EXIT=' "$L")"; break; fi
  if pgrep -f 'codex exec --yolo' >/dev/null; then
    echo "codex active: log=$(wc -c < "$L" | tr -d ' ')B last: $(tail -c 300 "$L" | tr '\n' ' ' | tail -c 140)"
  else
    echo "codex NOT running and no EXIT marker"; break
  fi
done
```

For several parallel runs, give each its own log and match the process on
something unique in its command line (e.g. `pgrep -f "wt/$n -o"`), print one
status line per tick (`#354:active 221KB #355:done(0) ...`), and stop when all
are done or one is NOT-RUNNING. A log that does not grow for one tick is
usually a long test run; two or three ticks: look at the tail.

## After

```bash
cat $S/codex_last.md
git status --short && git diff | diff -q - $S/before.patch && echo "tree unchanged"
```

## Prompt: review a branch (start in parallel with the review agents)

```
Review the current branch against origin/main in this <lang> repo (run
`git diff origin/main...HEAD`). It is PR #<n> "<title>" (issue #<i>): <one
paragraph: the problem and the approach>. Read the touched code and trace the
real flow (<entry point> → <service> → <helper> → <repository>).

Be razor sharp. <mindset block from SKILL.md>

Run experiments where a claim depends on real tool behavior (file modes under
a strict umask, what the external tool does with an existing output, etc.).

Do NOT modify any files. Output a concise list of findings: severity
(critical/important/suggestion), file:line, concrete failure scenario, and the
minimal fix. Say explicitly if you find nothing that blocks merging.
```

## Prompt: second opinion on an applied simplify pass

```
You are a senior <lang> reviewer giving a second opinion. DO NOT modify,
create, stage or commit any files; read-only commands (git, grep, vet, tests)
are fine.

Repository: <path>. Branch <branch>. Commit under review: <sha> ("<subject>");
run `git show <sha>`. A simplify pass (quality only: reuse, simplification,
efficiency, altitude) made UNCOMMITTED changes on top; run `git diff`.

<mindset block>

APPLIED in the working tree:
A. <change, why, side effects>
B. ...
(state how each behavior-relevant item was verified, e.g. the mutation check)

SKIPPED deliberately:
S1. <finding> — <reason>
S2. ...
Sx. Follow-ups, out of scope: <list>

YOUR TASK: for EACH item A… and S…, answer AGREE or DISAGREE with a one- or
two-sentence justification grounded in the code (cite file:line). Then list
anything the pass MISSED that fits the mindset, and anything it made WORSE
(readability, test strength, behavior change, security, data loss). Be
concrete and terse. End with exactly one line: "VERDICT: AGREE",
"VERDICT: AGREE WITH CHANGES: <list>" or "VERDICT: DISAGREE: <reason>".
```

Listing the skips is what made codex catch a rejected alternative that was
actually right, and a deleted assertion that should have stayed.

## Prompt: check fixes before push

```
Second opinion on the UNCOMMITTED changes in this repo (`git diff`), on top of
PR #<n> (context: `git diff origin/main...HEAD`).

The change addresses: <finding>. Instead of <old approach>, it now <new
approach>, so <property> holds as before. Also: <smaller fixes>.

Verify: <the specific invariants: atomicity, cleanup on every path, isolation
between concurrent requests, behavior under non-default env>, and that the
test still proves something (<known weakness of the test, if any — is that
acceptable or is there a cheap way to make it stricter?>).

<mindset block>

Do NOT modify any files. Output: severity, file:line, concrete failure
scenario, minimal fix. Say explicitly if it is fine to push.
```

## Implementing follow-up issues with codex

Setup: one worktree per issue under `$S/wt/<n>`, each on its own branch from
the current PR head (fetch first and check nobody pushed since). Prompt =
`task_<n>.md` + shared `rules.md`.

`task_<n>.md`: issue number and `gh issue view`, the problem as you verified
it, the expected direction ("adapt it after reading the code"), the
non-negotiable safety requirements, which other parallel issues touch nearby
code, and exactly which tests must exist. Do not prescribe an approach the
review has not validated yet.

`rules.md` (shared):

```
## How to work
You are a senior <lang> engineer. A senior developer will review this PR line
by line. <mindset block> Match the surrounding code: comment density, naming,
error handling, logging, test style. Read neighbouring files and tests first.

## Environment (strict)
- Work only in this worktree and branch. Do not touch other branches or
  worktrees; do not modify, rebase or push <PR branch>; no force-push, no merge.
- Other agents work in parallel. Do NOT run anything that starts or resets
  the shared database or binds ports (<list the make targets>).
- Gates before committing: <project gates>. For lint: no NEW findings compared
  with the base branch. Do NOT fix pre-existing findings in files you do not
  otherwise touch.
- Every behavior change needs a test that FAILS without the fix; prove it
  with a mutation check and mention it in the PR.

## Git and PR
- Conventional commit in the repo's style (`git log --oneline -15`), body
  explains why, ends with `Closes #<issue>`.
- NO AI attribution anywhere (trailers, "Generated with", mentions of any
  assistant) in commits, code, comments, branch names or the PR.
- `git push -u origin HEAD`; `gh pr create --base <PR branch> ... --body-file
  <file outside the repo>`. Body: follow an existing PR's structure; start
  with a "Stacked on #<n>" note if stacked; list which checks were NOT run.
- Final message: PR URL and a 5-line summary.
```

The "no NEW lint findings" line exists because a "lint must be clean" rule
made three parallel sessions each refactor the same pre-existing tests in
conflicting ways.

Afterwards, per PR: `gh pr view <n> --json baseRefName,commits`,
`gh pr diff <n> --name-only` (scope creep?), grep body and commit messages for
`claude|codex|anthropic|openai|co-authored|generated with`, rerun gates on the
branch, strip out-of-scope changes, update the PR body to match, remove the
worktrees.
