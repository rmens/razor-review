---
name: razor-review
description: "Razor-sharp finish of every open PR in a GitHub repo (or only the PR whose link is given): simplify pass (reuse/simplification/efficiency/altitude), multi-agent review (bugs, silent failures, tests, comments, types), codex --yolo second opinion after each pass, mutation-proof tests, then follow-up issues and optional codex PRs for out-of-scope findings. Use when the user asks to simplify and/or review PRs thoroughly, says a senior dev is watching, or wants codex to double-check."
user-invocable: true
argument-hint: "[PR link(s) | owner/repo] [simplify|review|all] [extra focus]"
---

# Razor review

Takes PR branches from "works" to "a senior dev can read it line by line".
Two passes per PR, each followed by an independent codex check, then
follow-ups.

## Arguments and target selection

- **PR link(s)** (`https://github.com/<owner>/<repo>/pull/<n>`, or `#<n>`):
  process only those PRs.
- **Otherwise: ALL open PRs** of the repo. The repo is `owner/repo` if given,
  else the current directory's GitHub remote (`gh repo view`).
- Mode: `simplify`, `review` or `all` (default `all`: simplify first, then
  review). Anything else is extra focus, passed to every agent and to codex.

List the targets once and show them to the user before starting:

```bash
gh pr list -R <owner/repo> --state open --limit 200 \
  --json number,title,author,headRefName,baseRefName,isDraft,isCrossRepository,maintainerCanModify
```

Per PR, decide what is allowed (state it in the target list):

| PR | What the skill does |
|---|---|
| Authored by the authenticated user (`gh api user -q .login`) | full flow: apply fixes, commit, push |
| Someone else's PR, or a fork without `maintainerCanModify` | review only; findings in the report. Ask once, for all such PRs together, before pushing commits to them or posting PR comments |
| Bot dependency bump (dependabot, renovate) | review pass only (changelog, breaking changes, lockfile sanity, CI status); no simplify, no code edits |
| Draft | included, unless the user excludes drafts |

## Batch execution (more than one PR)

1. **Order**: stacked PRs follow their base. A PR whose `baseRefName` is
   another open PR's head goes after that PR. Otherwise oldest first.
2. **Isolation**: one worktree per PR in the scratchpad
   (`git worktree add $S/wt/pr-<n> origin/<headRefName>`, then
   `git switch -c`/track the head branch; for forks use `gh pr checkout <n>`
   inside the worktree). Never run two PRs in the same checkout.
3. **Concurrency**: PRs run one after another; agents within a PR fan out in
   parallel. Starting the next PR's simplify agents while the previous PR's
   codex run finishes is fine, as long as the worktrees are separate.
4. **No stalling**: for the user's own PRs, apply simplify fixes, review
   "must fix" items and clear-cut suggestions without asking per PR. Anything
   that needs a product decision is skipped and listed in the report.
5. **After a base PR changes**, dry-run merge every PR stacked on it and
   report (do not silently rebase) conflicts you caused.
6. **Progress**: one short line per PR when it starts and when it finishes
   (verdict, commits pushed, issues filed).
7. Remove the worktrees at the end; the branches are pushed.

Steps 0–3 below run per PR, inside that PR's worktree.

## The mindset (pass it verbatim to every agent and to codex)

```
1. Does this need to exist?   → no: skip it (YAGNI)
2. Already in this codebase?  → reuse it, don't rewrite
3. Stdlib does it?            → use it
4. Native platform feature?   → use it
5. Installed dependency?      → use it
6. One line?                  → one line
7. Only then: the minimum that works
The ladder runs after you understand the problem, not instead of it: read the
code the change touches and trace the real flow before picking a rung.
Lazy about the solution, never about reading.
Lazy, not negligent: trust-boundary validation, data-loss handling, security
and accessibility are never on the chopping block.
```

## Step 0 — Scope and baseline

1. `git fetch`. Diff against `origin/<baseRefName>...HEAD` (for a PR on
   main: `origin/main`), never a local branch (often stale). Include
   `git diff HEAD` if there are uncommitted changes.
2. `gh pr view <n>` for the description and linked issue, and stacked PRs
   that build on this branch (`gh pr list --base <this-branch>`).
3. Read every touched function in full plus its callers. Trace the real flow
   end to end (handler → service → helper → repository). Note which existing
   helpers sit next to the change: those are the reuse candidates.
4. Load the matching language skills (Go: `golang-how-to`, which pulls in
   code-style, error-handling, testing, security as needed).
5. Run the baseline gates once and record the result (see **Gates**). A red
   baseline is reported, not silently fixed.
6. `which codex` and check `codex exec --help` for `--yolo`, `-C`, `-o`.

## Step 1 — Simplify pass (quality, not bugs)

Launch 4 agents in ONE message (parallel), each read-only, each given the
diff command, the traced flow from step 0, the mindset, and one angle:

- **Reuse**: new code that re-implements an existing helper; name it.
- **Simplification**: redundant state, near-copies, nesting, dead code;
  give the simpler form as a snippet.
- **Efficiency**: repeated I/O, sequential independent work, hot-path or
  startup cost, closures that keep large scopes alive. Measure, don't guess.
- **Altitude**: is the fix at the root cause or a bandaid; special cases on
  shared infrastructure; what belongs in a follow-up instead.

Every agent prompt ends with: "REVIEW ONLY. Do NOT edit, stage, commit or
push anything. Report file:line, a one-line summary, the concrete cost, the
fix, and say explicitly when something is already minimal."

Then:
1. Dedup (same line or same mechanism). Findings flagged by several agents
   go first.
2. **Behavior-preservation check before keeping or rejecting anything that
   touches modes, paths, env, encodings or timing**: compare old vs new
   behavior under NON-default conditions too (e.g. `umask 077`, other
   locale/TZ, missing dirs), and test it for real (run the tool, check the
   file). "Same under defaults" is not "same". (Lesson: a hard-coded
   `Chmod(0o644)` passed every default check and was only caught later
   under `umask 077`; the fix was to let the tool create the file again.)
3. Apply the remaining fixes. Skip what changes intended behavior, reaches
   far outside the diff, or is a false positive, and note each skip with
   its reason.
4. Re-read your own final diff once (stale log messages, comments that now
   lie, unused imports).
5. Gates + mutation check (see below).
6. Codex second opinion on the applied pass (see **Codex**), with every
   applied item (A, B, C…) and every skipped item (S1, S2…) listed.
7. Apply codex's justified corrections, rerun gates and mutation check.

## Step 2 — Review pass (bugs and risk)

Launch in ONE message, all read-only with the same closing line as above:

| Agent | When | Focus to give it |
|---|---|---|
| `pr-review-toolkit:code-reviewer` | always | correctness, CLAUDE.md, who relied on old names/paths/behavior (grep docs, openapi, README, tests too), same bug class elsewhere |
| `pr-review-toolkit:silent-failure-hunter` | error paths changed | swallowed errors, cleanup on crash/kill/shutdown timeout, leftovers that now accumulate, misleading user-facing errors |
| `pr-review-toolkit:pr-test-analyzer` | tests changed | does each new test FAIL on base (prove it in a scratch worktree), redundancy, flakiness, missing critical paths |
| `pr-review-toolkit:comment-analyzer` | comments changed | every claim checked against real behavior; comments that describe the diff instead of the code |
| `pr-review-toolkit:type-design-analyzer` | types added | invariants, encapsulation |
| `pr-review-toolkit:code-simplifier` | after step 1 was skipped | same ladder, review only |

Start codex in parallel with the agents (independent opinion, saves time).

While they run, check the high-risk spots yourself (callers of changed
helpers, file modes, container user and volumes, concurrency/ordering).

Aggregate into: **Must fix before merge / Suggestions / Out of scope /
Deliberately left out / What is good**. With a single PR, ask the user before
applying unless they already said "fix everything"; in batch mode follow the
"No stalling" rule above.

After fixing: gates, mutation check, and a second codex run on the
uncommitted fixes ("Say explicitly if it is fine to push") before pushing.

## Gates

Run the project's own targets (check Makefile/CI first). For Go projects:

```bash
gofmt -l .                                   # must be empty
go vet ./... && go vet -tags integration ./...
go test -race -count=3 <touched packages>
golangci-lint run --timeout=5m
```

Compare lint with integration tags against the base branch: the gate is
"no NEW findings", not "zero findings". Never run DB-resetting targets
(`make test-integration`, `make db-reset`) without asking.

## Mutation check (every behavior fix)

Temporarily restore the old production code (keep the new tests), run the
new tests, confirm they FAIL for the right reason, restore, confirm green.
If the old code no longer compiles (unused import), fix only that. Report
which mutations were caught.

## Codex second opinion

Details and templates: [references/codex.md](references/codex.md). Short form:

1. Write the prompt to a scratchpad file. Save `git diff > before.patch`.
2. Run in the background:
   `codex exec --yolo -C <repo> -o <S>/last.md - < <S>/prompt.md > <S>/run.log 2>&1; echo "EXIT=$?" >> <S>/run.log`
3. Arm a Monitor (load it via ToolSearch) that checks every 30 s: EXIT
   marker present, process alive (`pgrep -f`), log growing. Report briefly.
4. Read `last.md`. Verify codex changed nothing:
   `git status --short` and `git diff | diff -q - before.patch`.
5. `--yolo` matters: it lets codex run real experiments (ffmpeg, umask). A
   read-only sandbox run missed the umask bug that the yolo run found.

## Guardrails (learned the hard way)

- After every agent finishes: `git status`, `git log origin/<branch>..` and
  `git log ..origin/<branch>`. A "review only" agent once edited, committed
  and pushed. Someone else once pushed to the branch mid-session. Detect
  both before building on top.
- Separate commit instead of amend on a pushed branch (no force push).
  Conventional commit messages; no AI attribution anywhere.
- After rewriting tests or helpers, dry-run merge every stacked PR
  (`git merge-tree` or a scratch worktree) and report conflicts you caused.
- Out-of-scope findings never sneak into this PR.

## Step 3 — Follow-ups

1. `gh issue list --search` for duplicates first. If an issue exists but
   describes an approach this PR just rejected, update it instead.
2. Verify each claim empirically before filing (reproduce it locally).
3. Write issues like the original issue (language, structure), labels as
   the repo uses them (e.g. `bug`/`tech-debt` + `follow-up`).
4. If the user wants codex to implement them: one worktree and branch per
   issue from the PR head, one shared rules file plus one task file per
   issue, parallel `codex exec --yolo` runs, one combined 30 s monitor. See
   the references file for the rules block.
5. Verify every codex PR before reporting: base branch, `gh pr diff
   --name-only` for out-of-scope files, attribution grep on commits and
   body, gates on the branch. Strip scope creep, then fix the PR body so it
   matches the diff.

## Final report (in the user's language)

Batch mode: start with one table, one row per PR: PR, verdict (ready /
needs a decision / blocked), commits pushed, codex verdict, issues filed.
Then, per PR that needs attention, the details below.

Per PR: verdict first, then what changed per commit, gates with results,
mutation results, codex verdict (and where codex and you disagreed), skips
with reasons, issues/PRs created or updated, and what is still open.
